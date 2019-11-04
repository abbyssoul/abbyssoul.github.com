---
layout: post
title:  "New version of libstyxe supports 9p2000.u and 9p2000.L"
author: abbyssoul
date:   2019-11-05 09:00:00 +1000
categories: engineering
tags: [c++, styxe, engineering, 9p]
---

After a week of intensive work I am happy to share a new version of [libstyxe][libstyxe-git] that now supports
extended version of [9p][9p-protocol] protocol: `9p2000.u` and `9p2000.L`.


Original 9p protocol designed for [Plan9][plan9-wiki] to be minimalistic.
>  9P is a distributed resource sharing protocol developed as part of the Plan 9 research operating system at AT&T Bell Laboratories (now a part of Lucent Technologies) by the Computer Science Research Center. It can be used to distributed file systems, devices, and application services. It was designed as an interface to both local and remote resources, making the transition from local to cluster to grid resources transparent.

> 9P2000.u is a set of extensions to the 9P protocol to better support UNIX environments connecting to Plan 9 file servers and UNIX environments connecting to other UNIX environments. The extensions include support for symbolic links, additional modes, and special files (such as pipes and devices). Also included are hints to better support mapping of numeric user-ids, group-ids, and error codes.

(From [9p2000.u RFC](https://ericvh.github.io/9p-rfc/rfc9p2000.u.html) )


According to [9p overview](https://www.slideshare.net/ericvh/9p-overview) - 9p2000.u did not provide a direct mapping to Linux operations and thus [9p2000.L](https://github.com/chaos/diod/blob/master/protocol.md) has been designed.


This extensions to the protocol took quite different design approach. `9p2000.u` adds no new messages but extend existing ones adding new fields.
`9p2000.L` on the other hand - add the whole bunch of new messages that better match Linux FS operations at the price of doubling number of messages a server need to handle.  


Please let me know if you found these extensions useful what is your experience with 9p versions.


[9p-impl]: http://9p.cat-v.org/implementations
[9p-protocol]: https://en.wikipedia.org/wiki/9P_(protocol)
[plan9-wiki]: https://en.wikipedia.org/wiki/Plan_9_from_Bell_Labs
[libstyxe-git]: https://github.com/abbyssoul/libstyxe
