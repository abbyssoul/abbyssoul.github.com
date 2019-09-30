---
layout: post
title:  "libstyxe - a 9P2000 parser"
date:   2019-09-27 12:00:00 +1000
categories: [C++, styxe, 9p, plan9]
---

I my spare time I am working on a pet distributed system of mine. The kind where multiple processes all running on different machines (raspberry PIs, Jeton Nanos and x86 server) - all do its own thing and come together to advance a common goal. This kind. So while working on this project I realised that
most important part is for my processes to communicate. That is to exchange data. It would be very good if I could re-use something proven.
At this point I realised that I have heard about one interesting approach to this problem. There is this operating system - [Plan 9][plan9-wiki] - where everything is a file (for real)! That means all HW devices as well as remote resources are files. Do you need more compute power? No problmes! Just mount extra CPUs from a remote machine and run your task.
It does not sound like a radical idea this days - but it sure was an interesting take on communicating processes and system design back in the day.
Most interesting are the results.

So with that in mind I decided to implement similar approach to IPC. And to do that I needed 9P2000 protocol parser and message builder.

# Reasons to exist
I need an implementation of 9P2000 message parser and building in C++. Despite 9P protocol being relatively simple and well documented - there are no
too many implementation in C++. The best resource I got is [this list][9p-impl]. So I decided to implement my own :)

## What is 9P2000 and why should I care?
[9P2000][9p-protocol] is a network protocol. More specify it is a remote filesystem protocol initially designed for by Plan9 operating system to build truly networking OS.
When we talk about networking filesystem one may think of NFS. Indeed 9p protocol is similar in concept but has a number of design differences.

## What is Plan 9?
"Plan 9 from Bell Labs is a distributed operating system, originating in the Computing Sciences Research Center (CSRC) at Bell Labs in the mid-1980s, and building on UNIX concepts first developed there in the late 1960s. The final official release was in early 2015." [from Wikipedia][plan9-wiki]
What is interesting about it is the fact that it is minimal in terms of design. 9P filesystem protocol is a foundation
that allows Plan9 to be simple and yet powerful distributed system. For example it is possible to configure your system such that
 on a local machine you will only have a thin client starting tasks and the computation will be done by a remote plan9 running machine.
 Storage would be provided / mounted form yet another remote instance. It is pretty flexible in that sense.

Since the system I am working on is meant to be distributed I could clearly see some similarities. So it made sense to try to apply already existing concepts.

# Design and implementation
When designing my own parser I wanted a few things:
 - No threading. I don't want libraries spawning threads without my knowledge. My applications make the best use of resources
 and for that I need to control resource usage of my libraries. Luckily for me - parsing of messages does not normally requires extra threads.
 - No memory allocations. That it pretty much more of the above. My App controls resource. If a parser needs memory - app allocates it and hand over to the library.
 This one proved a bit tricky but nothing crazy. If a user has a message buffer parsing can be done with 0 allocations.
 - No exceptions. This is more of a design experiment - not a political stand on exceptions. Can I design a library that does not use exceptions.
    And yet robust in presence of exceptions. I decided to use `Solace::Result<>` to communicate error. This communicates that most errors are
    recoverable. Indeed in case of message parse / writer where everything depends on the user input - it is reasonable to expect that this input can be invalid.
    That is - it is not an exception. Thus it must be communicated back the caller. 
 - Composition over inheritance. This library turned out to be perfect design exercise to implement something without using inheritance.

# Conclusion
In order to make my distributed system I chose to implement an existing, well documented distributed filesystem protocol 9P.
The result is an opensouce library [libstyxe][libstyxe-git] that you can use in your C++ project to read and write 9P messages.
Add networking and you can communicate with other machines speaking 9p.


[9p-impl]: http://9p.cat-v.org/implementations
[9p-protocol]: https://en.wikipedia.org/wiki/9P_(protocol)
[plan9-wiki]: https://en.wikipedia.org/wiki/Plan_9_from_Bell_Labs
[libstyxe-git]: https://github.com/abbyssoul/libstyxe
