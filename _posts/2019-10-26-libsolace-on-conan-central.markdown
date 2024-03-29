---
layout: post
title:  "libsolace is now on Conan-central"
date:   2019-10-26 13:00:00 +1000
author: abbyssoul
categories: [announcement]
tags: [c++, solace, conan]
---

![Good news, everyone!](http://www.quickmeme.com/img/be/beeea97c622c5614a80371eb0aece7082a3ff1b50421cf5acae3125c65ba5d8f.jpg)

[Libsolace][libsolace-git] is now available on [Conan central][libsolace-conan-latest].


> Conan is an open-source, decentralized and multi-platform
package manager.

If you are using [Conan](https://conan.io/) for your project and would like to use `libsolace` it is as easy as adding these lines to your project `conanfile.txt`:
```
[requires]
libsolace/0.3.6
```

Just make sure to use [latest version][libsolace-conan-latest].

Let me know if you find it useful or you are using a different package manager for your code.

[libsolace-conan-latest]: https://conan.io/center/libsolace
[libsolace-git]: https://github.com/abbyssoul/libsolace
[conan-center]: https://conan.io/center/
