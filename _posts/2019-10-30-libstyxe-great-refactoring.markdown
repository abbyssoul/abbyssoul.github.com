---
layout: post
title:  "libstyxe: Great Refactoring"
author: abbyssoul
date:   2019-10-30 12:00:00 +1000
categories: engineering
tags: [c++, styxe, engineering, 9p]
---

Recently I completed a redesign of `libstyxe`. As it is often the case - it took longer then I hoped. The new design is better suited for extensibility. The downside is that I had to break existing interfaces. So in this post, I would like to explain why I decided to do it and what goals I had in mind.

## What is libstyxe?
[libstyxe][libstyxe-git] is a parser library for [9p2000][9p-protocol]. [9p2000](http://9p.cat-v.org/) is a network file system protocol that is a foundation of [Plan9 OS](https://9p.io/plan9/). It is famous for taking the concept of "everything is a file" to the new level.
As I am working on a distributed system as a hobby - I thought it would be an excellent match to 9p as a protocol. That meant that I had implemented one in C++. `libstyxe` is the result of this work.

I wanted the library to follow the same principles I use for [libsolce]({% post_url 2019-10-07-libsolace-philosophy %}): no uncontrolled memory allocations and not thread creation.
This requirement somewhat limits the scope of what the library provides. Notably, the library features no IO/networking support. Also, message parsing should only provide views into the message buffer rather than copy data. Likely in practice, however, it is never an issue with 9p.

My first implementation had some experimental design choices. For example, I opted for builder/writer hybrid of message builder interface.
`MessageBuilder` as the name suggests is a class that a library consumer would use to create a 9p message. In my implementation, however, no message object was returned by a builder. Instead, message data written into a data stream passed to the constructor of the `MessageBuilder`.

To clarify the usage, I later renamed this class to `MessageWriter`. It is also advantageous to keep `RequestWriter` and `ResponseWriter` as separate classes to prevent unexpected usage. We don't want servers - that only ever write `Response-type` messages to send us a request message accidentally.
For example, to write `TRead` a message into a byte stream one would call:

```c++
 RequestWriter writer{byteStream};
 writer.read(fid, offset, count);
```
Notice how the type of the message is defined by a class used. `RequestWriter` in the example produces a `request`. Call to  `RequestWriter::read` writes a _read-request_ message without creating an intermediate object. In that sense, it is a mapping of input arguments into a byte stream with some extra information about the message type.

The server for example, could reply with data message:
```c++
 ResponseWriter writer{outputByteStream};
 writer.read(data);
```

This design worked for a while.

## Why change anything?
9p is not a single a protocol. Several 'flavours' has been created over time. Notably:
- [9p2000.u](https://ericvh.github.io/9p-rfc/rfc9p2000.u.html) - is Unix adoption.
- [9p2000.L](https://github.com/chaos/diod/blob/master/protocol.md) - is extension over 9p2000.u
- [9p2000.e](https://github.com/cloudozer/ling/blob/master/doc/9p2000e.md) - is Erlang on Xen extensions that adds short read and short write messages. This minimized network round-trip.

My initial plan was to implement the smallest subset - that is `9p2000`. But after some time using a protocol on my server - I realized that some things could be streamlined to minimize network traffic. Also, if a server implements Unix-style user ids - 9p2000.u is a better fit.

That is why I realized that I might need to support other versions. Implementation, however, proved challenging with the existing design.

## Required changes.
In order to support a new protocol version, a few changes required. At very least I'd need to add new methods to the `ResponseWriter` and `RequestWriter` to support new messages.
If that would have been the only way new versions differ - that would have been a minor issue to solve.
It turns out that some extension also changes existing messages. Thus 9p2000.u `Stats` struct has extra fields.

Then there is also a problem of version negotiation. 9p's first message in the session establishment sequence is a client proposition of protocol version and message size.
A server reply with its own preferred version. Thus if parser supports multiple versions `['9p2000', '9p2000.e', '9p2000.l']` - selecting one would change the parser.

In OOP terms it means that parser is `polymorphic` and there is a factory that creates a parser version given a string:
```
auto parser = createParser(versionStr);
```

Given library constraints - I would need to solve it without allocating dynamic memory. Thus no inheritance.

### Polymorphism without inheritance
Without going full ["Inheritance Is The Base Class of Evil"](https://sean-parent.stlab.cc/papers-and-presentations/#inheritance-is-the-base-class-of-evil) here
we may notice that parser is a mapping of message type-number and a byte stream into a message object:
```
parse(protocolVersion, MessageType, byteStream) -> Result<Message, Error>
```
I used result here to indicate that message type may be invalid within a given protocol version. Or data may be invalid. See my [post about errors]({% post_url 2019-10-01-thinking-about-error %}) in the code.

Given that `MessageType` is byte we can not have more them _256_ mappings. So we can use a table of function pointers: `parse[messageType] -> *parserFunction(byteSteam)` and to select
the table for each version. In other words, we have invented a virtual function table.

The new version of `Parser` takes request parser table and response parser table. The extra benefit of using parser table this way is that all entries for 'unsupported' message types - point to
the same error producing function. So we have our 'unsupported message type' case covered.



### Modularity

It would be nice also to have some modularity. I want to keep the code for different version separately. This way it should be possible for library users to chose protocols
they actually want to support.

Modularity should also help with future extensibility.

Thus, first of all, I had to redesign `ResponseWriter`/`RequestWriter` interface to be extendible without inheritance.
The way to do it? Good all standalone `operator<<(ResponseWriter& , ...)`. What should be passed as arguments? Protocol message we would like serialized.
This, unfortunately, means I do need to have an object to represent each message. As it turned out, I already had it - this is the result of message parsing.



## Incidental changes
After separating `MessageType` enums into independent modules, it turned out that `MessageHeader` struct can not include enum field for type. Instead, it should be a simple byte.
It is easy to understand - the same byte value may mean different messages in different versions. Not the best practice but nothing preventing it.
The consequence of this change - is printing message name now depends on the negotiated protocol version.

Thus simple `operator<<(std::ostream& out, MessageHeader)` is no longer an option as it does not accept parser version to be used.
A bit more verbose solution is used instead:
```c++
Parser::messageName(Solace::byte messageNumber) -> StringView
```
This member function of `Parser` class returns string representation of the message name, if it is a valid `messageNumber` value for a selected protocol version.

## How a new interface compares to the old version

And so putting it all together - a new interface to write `TRead` message into a byte stream:

```c++

 RequestWriter writer{byteStream};

 writer << Request::Read{fid, offset, count};
```

...and a server side response:
```c++
 ResponseWriter writer{outputByteStream};

 writer << Response::Read{data};
```

This design means that we do have to create temporary objects only to shuttle arguments into a call to `stream::write()`. Lucky for us - all messages only view and don't care about any data that requires allocations.



# Conclusion
In software engineering, We often discuss over-engineered solutions and how is it a problem. It is essential to focus on a problem at hand.
Focusing resources on solving theoretical problems that may never happen - is a lousy resource management strategy. At the same time,
it is crucial not to designs yourself in a corner such that when requirements change - your code can evolve. Walking this path - is what the art of software engineering is about.
`libstyxe` design goal was to keep things simple. And now the time has come for `libstyxe` to support a new set of extensions. That required some rework and review of initial design choices interfaces.

I kind of like this new design. Keen to see how far it can take me.
Let me know what you think about this approach to design of a message parser.


[9p-impl]: http://9p.cat-v.org/implementations
[9p-protocol]: https://en.wikipedia.org/wiki/9P_(protocol)
[plan9-wiki]: https://en.wikipedia.org/wiki/Plan_9_from_Bell_Labs
[libstyxe-git]: https://github.com/abbyssoul/libstyxe
