---
layout: post
title:  "libsolace: library philosophy"
date:   2019-10-07 12:00:00 +1000
author: abbyssoul
categories: engineering
tags: [solace, c++, engineering, reliability, philosophy]
---

It is no secret that software is all around as this days. It runs traffic lights, washing machines, cars, businesses etc. Every type you plan a trip - your booking is processed online by a data centre. The plain you travel on is control by the software. As is the airport you travel to and from. And not even touching on medical equipment and production plants.
That is why I believe we all can benefit from the software being well built. No unexpected crashes. I touched on the topic error handling before in my [previous post]({% post_url 2019-10-01-thinking-about-error %}).

For software to be useful, it needs to be reliable. Not only must it perform its function 24x7 but be resilient when unexpected things happen.
To build such good software engineers must to follow good engineering practices. And in time we can notice that we keep repeating what worked for us in the past. This is how one forms a library.

[libsolace][libsolace-git] is my attempt to capture some of the primitives that I found useful in the past.


## Philosophy and promises

I wanted to collect primitives that enable me to build different applications. Such collection is known as a _library_:
> a collection of types, functions, classes, etc. implementing a set of facilities (abstractions) meant to be potentially used as part of more that one program.
>
> -[Cpp Code guidelines glossary](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#glossary)

Since we are foremost interested in reliability we should first identify some principles that a library must follow to be a foundation of reliable software.
One such principle - is the property of architecture where using it incorrectly is difficult, better yet impossible.

### P10
A pretty good starting point is the P10:  [NASA's Rules for Developing Safety Critical Code](http://spinroot.com/gerard/pdf/P10.pdf).
A major take away for me was to write maintainable code in terms of size and readability and have explicit control over memory management. This, however, poses some challenges in the C++ world. Most notable dynamic collections and string has a tendency to allocate memory in the most unexpected moment. Note that it, not allocation that is the problem here - unless it happens on a critical path. But rather the time when it happens.

For example, it is easy to pre-allocate memory for a vector and ensure no resize penalties. Unfortunately, it is also easy to omit a call to `vector.reserve()` with no noticeable consequences.
What if it was impossible to create a vector without pre-allocation? Of course, we can use a custom `create` function to get an instance of a vector:
```c++
template<typename T>
auto makeVector(size_t initialSize) {
    std::vector<T> result;
    result.reserve(initialSize);
    return result;
}
```

Notice there is no memory copy penalty when using C++ 17 due to [copy elision](https://en.cppreference.com/w/cpp/language/copy_elision).

Vectors are not the only STL components that allocate memory behind your back. The whole concept of PMR and [memory_resource](https://en.cppreference.com/w/cpp/memory/memory_resource) introduced in C++17 is there to confirm it.

When `libsolace` were conceived there [polymorphic memory resources](https://isocpp.org/blog/2018/10/pmr-polymorphic-memory-resources) were not yet proposed. In that sense `libsolace` attempts to solve a similar problem of giving a developer more control over memory allocation.


### Developer in control
Memory management is not the only aspect of software engineering that an engineer must keep in mind when designing reliable software.
In general, an application is required to manage various resources: CPU allocation, memory, disk space, networking, etc. That may be surprising at first: all of these are taken care of by an operating system, no? On the one hand, that is true - a  primary goal of an operating system is to manage resources of the system. Well technically, the goal is to provide access to existing and limited resources collaboratively. So that apps won't step on each other. There is still a problem of a finite amount of CPUs and Memory and storage and network bandwidth. The operating system gives out a slice of these resources to an application. If a computation requires more memory then available on the system - OS will try its best to accommodate a request. But it can not download more RAM for your app. Nor can it spawn a new CPU if there are no left or app takes too long to compute.
If you are familiar with web development - the solution there is to design horizontally 'scalable' applications. Question of how to actually do that makes for a good job interview question.
When a single instance of scalable web-app struggles to handle incoming requires - _an operator_ creates another instance to run on a different machine. A web app can also be designed to handle a single function of the system, and the system is _composed_ of several such web-apps working together. This approach is called micro-service architecture - has a subtle and essential property in terms of system resilience. In a well-designed system, failure of a single micro-service - does not result in failure of the system as a whole. Also - when the load on the system increases only the services responsible for an impacted function should be scaled - not the whole system.
Example: if your web app users engage in a heated discussion and the load on the comments micro-service increases - only this service should be scaled up. That is more instances of the `comments` micro-service are to should be brought online — no need to scale purchasing services.

#### No implicit threading
Given that we want to have a resilient system - we need tools that can provide us with reasonable control of system resources. If your app is CPU intensive - you don't want a third-party library, the app is built with, to create new system threads without your permission to run some task. Interestingly enough - this reminds me of GC pause in garbage-collecting languages: GC runs in a separate thread can interrupt the application to collect garbage. Although tolerable in some cases - it is not precisely a desires behaviour for time-critical apps.

#### No implicit networking
Another aspect of resource control - third party library should not run network queries unless it is the primary goal of the library. Performance-wise - there is the issue that networking calls can be blocking and non-blocking. And I have heard enough was stories about people chasing blocking calls in a library used by an async application. So networking model is an essential choice of the application design, and I wouldn't want a non-networking library interfering with this chose. I'd rather have composable library that works with wide variety of IO models. But there is even more severe aspect of network calls from third-party libraries - security. I don't want to go too deep into details but suffice to mentioned that there were cases in the past when a library would send data over the network.

#### No implicit memory allocation
Finally, some applications are concerned about when and how memory is allocated. Thus a composable library must provide such applications with means to manage memory. It can be in a simple form - each call that needs to allocate memory - must take a memory allocator as an argument.  This is the way I decided to do it in libsolace.
There might be other ways. I am curious to hear how else this problem can be approached.


### Composition over inheritance
When designing a library it is difficult to predict all possible ways a library user will use it. There are two ways to go about it that spring to my mind: be _prescriptive_ or be _flexible_.
A _prescriptive_ library - or frameworks, or how it is called these days 'an _opinionated_ library' - dictates the way it expected to be used. It usually defines a hierarchy of classes and order they should be wired together. In other words - it has a tight coupling between components.
The upside of this approach - is that it is usually easy to use it given that the library-provided _solution_ matches your _problem_ and your approach in general. One of the libraries I created - [libtribe][libtribe-git] - is an example of such approach - it provides a model and a set of actions to be performed on this model. This is known as [Action-Model-View](https://sinusoid.es/talks/immer-cppcon17/#/34) approach. If this is the way you model your domain - [libtribe][libtribe-git] - matches your application nicely. And if not - hey - this is a good opportunity to learn more about this approach :P I might write a blog post about it :) (Yes - this is an example of opinionated approach).

The downside of _prescriptive_ libraries - you can't just take components you found useful and reuse them. The library might be not designed to take parts out of it. For example, you in is not expected that someone would take only message parser out of [libtribe][libtribe-git] and not the whole model.

It is exactly the opposite situation with _flexible_ libraries. It may offer an assorted collection of components but no instruction on how to put them together to solve your particular problem.
All standard library of programming languages fall into this category. [libSolace][libsolace-git] also follows this approach. A language support library is not designed to solve any domain problem - but rather give implementation level component.

In some sense, this debate _opinionated_ vs _flexible_ library reminds me of [Composition vs Inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance).
libSolace does not solve any particular problem but gives you tools that you can use to build a solution. For that to happen - these tools or components need to be composable. That is they need to work together.
One example of composition of components in [libSolace][libsolace-git] is design of `ByteReaded` and `ByteWriter`.
These classes model a simple stateful data stream. That is a fancy way to say a byte array with an integer denoting current position to read from / write to. `ByteReaded` and `ByteWriter`
operate over a memory buffer. A simples use case is to have a static C-style array you would like to write data to:

```c++

MyData my_data = ...;
byte data[256];
ByteWriter writer{wrapMemory(data)};
writer << my_data.x << my_data.y << message;

```

This snippet demonstrates how ByteWriter can be used to serialize data into a user-provided byte array.
But the libsolace also provides memory management facilities. So a user can allocate a _memory resource_ dynamically and use it to write data to:

```c++

MyData my_data = ...;
MemoryResource memory = memoryManage.allocate(321).unwrap();
ByteWriter writer{memory};
writer << my_data.x << my_data.y << message;

```

In this example ByteWriter is using dynamically allocated memory buffer as a write target. It is also possible to move memory resource into ByteWriter for it to take ownership of the resource
```c++
ByteWriter writer{memoryManage.allocate(321).unwrap()};
writer << my_data.x << my_data.y << message;
```

In this case - the memory will be de-allocated when the `writer` goes out of scope.


These examples illustrate an important point - a library user is not required to use a memory manager or memory resources to use `ByteReaded` and `ByteWriter`.
It may be a good idea in some cases but not in others. For example if you need to interop with STL/ legacy code:
```c++
std::vector<uint8_t> buf = ...;
ByteReader reader{wrapMemory(buf.data(), buf.size())};
```

This way you are free to pick and chose components that are required for your solution.


## Conclusion
This was just a brief overview of the philosophy and motivation for the `libsolace` library.
Software is everywhere around us and we rely on it more and more each day. Thus the software must be reliable and resilient
I plan to describe how this philosophy is has resulted in
In the meantime, I encourage you to check its Github [repository][libsolace-git].

Please let me know what library component you found useful and which one's purpose not clear.
Also, please [raise issues][libsolace-issues] if you run into troubles.


[libsolace-git]: https://github.com/abbyssoul/libsolace
[libsolace-issues]: https://github.com/abbyssoul/libsolace/issues/new
[libtribe-git]: https://github.com/abbyssoul/libtribe
