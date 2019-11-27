---
layout: post
title:  "Virtual filesystem: fun and profit"
date:   2019-11-25 12:00:00 +1000
categories: engineering
tags: [c++, engineering, vfs, kasofs]
---

When we use computers these days, we take for granted the fact that our data - documents, pictures, musicWhen we use computers these days, we take for granted the fact that our data - documents, pictures, music, etc - persisted. We are pretty happy to find a document we were working on if we can recall what we named it. So we are very familiar with the idea of a filesystem. A system that allows us to store and locate named files.
As often the case - things get a bit more interesting in the realm of distributed systems. Fundamentally we still need a way to find stored data.
It's just that we don't always care where is it physically located. What is important is the path to a file.
A bit more general - we need an address of a resource we would like to access.
In this post, I'd like to talk a bit about the concept that enables access to resources by address in a general sense. Such a concept is called virtual filesystem and
I have been working on my own library for it.


# What is an FS?
A filesystem was initially designed as a means to store computation state. Over time actual storage medium changed, and the need to store and read data remained. Application developers wanted to focus on useful computation that app performed and not on supporting various hardware that application may end running on. And so the system that the application interacted with to store data was abstracted. This abstract concept of storage became known as the filesystem.
Thus in a classic sense, a filesystem is an API provided by an operating system that allows users of a computer system to store data and later access it. This process must work with storage medium details varying from configuration to configuration. Which leads to the realisation that access to the data is more important than the fact of its storage.
If we drop the notion of persistent storage - we allow for data to come from anywhere. It can be computed on request. For example, reading a value from sensors, data from a remote server or kernel itself. For computer - everything is a data.

_Filesystem is just a way to access named data_. So _virtual filesystem_ is a recognition of the fact that is it is not as important where data physically stored. But it is essential how to access it.

Filesystem as a concept is a mapping of a hierarchical name to an object with some expected operations. Namely read and write.
Here I mean mapping in the mathematical sense. There exists a function that accepts `path` and produces, or maps it to an object.
In pseudo code it will be something like:

```c++
Filesystem fs;
auto maybeFile = fs.fileByName(path);
```

There are two critical things to note here:
There is no guarantee that the mapping `path` to `file` exists for any path. Or simply, not every path results in a file.  
Path is hierarchical. Two distinct path object can refer to the same file object.
The reason why this is important is that it might be tempting to compare a filesystem to key/value storage. Or assume that in OOP such system can be replaced by a simple `map`/`dictionary`/`hashmap`/etc.

Real filesystems provide more operations than just mapping. It also allows the listing of items in directories or walks up and down the hierarchy.

### What is a File?
For this post, a `file` is a nameable _object_ that has _write_ and/or _read_ operations defined on it.
Note in this definition a file is just something one can write into or read from or both. Also, we can find it later by name. Or the hierarchy of names also known as _path_.

### What good is VFS?
So filesystem as a concept is a mapping of a hierarchical name into an object with some expected operations. What can we do with it?
One example of virtual filesystem a reader may be familiar with is a [procfs][procfs]. The idea proposed for Unix that processes running by OS can be represented as files.
Not stored files, but OS can give us this information on request. In Linux, this is implemented as `/proc`. Every time a user reads from this directory, Linux kernel handles the request and reply with information about currently running processes.
Another familiar example of virtual filesystem is [devfs][devfs]. This system represents all devices attached to a machine as files. On Linux, this traditionally mounted under `/dev`. This files can be read and written.
There is a file for a camera attached to the machine. One for each hard drive. Files for USB ports. Not only real physical devices are represented by this FS but some purely synthetic ones. `/dev/null` - a file that always read `0`. You can also write any data you want to discard into that file too. Need some random data? Just read from `/dev/random`!

There is, however, an example of an operating system that took the concept of the virtual filesystem to the whole new level. [Plan9][plan9-wiki] is a system that represents most of its subsystems as filesystems.
A mouse is just as `/dev/mouse`, read it and you know mouse position. Need to open a new TCP network connection - write to `/net/ctl` connection details, and it will create a new file `/net/tcp/<number>`
for your connected socket. Windowing system - is VFS too, naturally, with a file for each opened window.
This approach allowed to create applications that interacted naturally with the system and each-other. No separate IPC required. If one application wants to interact with another - it will open relevant files provided by app-server.
This idea is really not that dissimilar to the notion of microservices we hear so much about these days.

## FS is more than just files
Whenever three is an interaction between processes, some form of access control is necessary. Even in a basic form of persistent storage, a filesystem is concerned with the management of permission a user might have for a particular file object.
And so is when designing an FS one need to consider the representation of user objects and permission scheme. Unix traditionally offered a simple solution in the form of numeric pair of user-id and group-id. And each file having exactly one owner identified by that user-id. This own controls permissions for the group and the _rest_ of the users. Other FS implementations have more detailed permission schemas such as [access control lists][acl-wiki] (ACL). In this approach, each file has an associated list of users who are allowed to do what with the file.

# Why do I need my own library for vfs?
As mentioned in my previous posts, I am currently interested in building my own grid computing solution. This means that I will have a bunch of processes running on a number of distributed machines.
These processes need to interact to achieve a common goal. With means, I need an IPC model. And I chose 9p file server as my model as it has been proven by [Inferno][inferno-wiki] and [Plan9][plan9-wiki] OS.

I already have my custom implementation of [9p protocol parser][libstyxe-git] and have a couple of servers using it. One thing I have to repeat with every implementation of a new 9p server is a mapping of hierarchical names to some virtual objects.  That is to say that my servers don't directly map to a real FS but represent some ephemeral object. I will publish a post with examples later. For now, I have been busy extracting common code required to write a new 9p server which I intend to publish soon.

# Conclusion
In this post, we talked about what a filesystem actually is. Turnes out if you take out the storage of data out of the FS concept you get mapping of hierarchical names to IO objects. This object can represent on-demand computations or physical devices as well as plain old data.
This is a very powerful concept successfully used by some operating systems. It is also a good IPC model for a grid computing system. The kind of system I am building. Which means I need my own implementation for VFS that I can use to write services. I am currently working on such implementation and hope to share more details soon.

What do you think about VFS as an IPC model? What else can be represented as a namable hierarchy?


[devfs]: https://en.wikipedia.org/wiki/Device_file#DEVFS
[procfs]: https://en.wikipedia.org/wiki/Procfs
[plan9-wiki]: https://en.wikipedia.org/wiki/Plan_9_from_Bell_Labs
[inferno-wiki]: https://en.wikipedia.org/wiki/Inferno_(operating_system)
[libstyxe-git]: https://github.com/abbyssoul/libstyxe
