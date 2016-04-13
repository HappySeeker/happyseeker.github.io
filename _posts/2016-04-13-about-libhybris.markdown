---
layout: post
title:  "闲聊Libhybris"
date:   2016-04-13 6:45:12
author: HappySeeker
categories: Android
---

# 闲话

最近开始考虑Android和Linux桌面兼容的问题，目的是丰富Linux当前最大的短板：应用生态圈。了解到Libhybris相关的内容，在此闲聊，权当总结和记录

# 什么是Libhybris？

Libhybris是一套适配库，用于解决GNU lib和Android lib之间的兼容性问题，目标是让标准的Linux中的应用程序能够调用Android lib。

这里说的GNC lib即：Glibc(GNC C库)和基于Glibc的一些基础库。Android lib即：Android中的C库(Bionic C库)以及基于Android C库的一些基础库。

# 为什么需要Libhybris？

需要从Linux和Android的工具链的差异说起。通常来说，工具链包括：编译器、汇编器、链接器和与之匹配各种库(比如C库)，Linux通常都使用GNU工具链，而Android则使用Google自己的工具链，其编译器和汇编器都是基于GCC的，问题不大，主要差别还在链接器和C库。

Android中C库是google自己开发的，取名为Bionic C，在构建Android时，需要先构建出Bionic C库，然后在基于Bionic C进行Android的构建。Bionic C与Glibc不完全兼容。

除此之外，Android中使用链接器/system/bin/linker，跟Linux中使用的LD也不相同，因此在加载动态库时使用的机制和策略也有差别。

由于Bionic C与Glibc不完全兼容，加上链接器的不同，导致在Android上编译的动态库，无法直接在标准Linux环境中使用，当需要在标准Linux环境中使用Android的动态库时，就需要Libhybris了。典型的应用场景如：

- Android环境中一些硬件相关的驱动库，通常由硬件厂商提供，而部分厂商的驱动是闭源的，仅提供二进制的动态库，想要厂商针对Linux环境重新编译提供一套动态库时不现实的，此时如果需要这些闭源驱动库，就需要Libhybris了，这应该也是Libhybris设计的初衷。
- 当需要在Linux环境中复用Android的基础设施时，比如，想要在Linux中使用Android的图形加速框架，此时需要用到Android中现有的动态库，也需要使用Libhybris。


# Bionic C的特点

Bionic C相比于Glibc有如下特点：

1. 采用BSD License，而不是Glibc的GPL License;
2. 大小只有大约200k，比Glibc差不多小一半，且比Glibc更快;
3. 实现了一个更小、更快的pthread库;
4. 提供了一些Android所需要的重要函数，如”getprop”，“LOGI”等;
5. 不完全支持POSIX标准，比如C++ exceptions，wide chars等;
6. 不提供libthread_db和libm的实现。

# Libhybris基本原理

如之前所说，Libhybris是一套中间层适配库，其基本原理为：封装所有Android lib的接口，并在Libhybris内部使用类似于linker(Android中的链接器)机制动态加载Android lib和重定向相应的接口。

其中有几个关键点：

1. Libhybris自身是用GNU的工具链编译出来的，所以，其可以在Linux环境中被LD链接器动态加载。
2. Libhybris中的所有符号(函数接口)本质上都是封装的，其真实的实现是在Android lib或GNU C中。
3. Libhybris本质上就是一个伪动态库。
4. Libhybris其实就是做了linker的工作。

# Libhybris工作流

应用程序使用Libhybris的工作流主要如下:

1、应用程序启动
2、如果是动态链接，且应用程序中调用了Android lib的接口，则使用LD加载Libhybris库。
3、调用Libhybris库中的接口
4、使用linker技术，调用真正的Android lib接口

关于Libhybris的详细的工作原理，这里就不多说了，感兴趣可以看看代码

# Question？

反过来，如果需要在Android系统中调用标准Linux库接口，该怎么办呢？大家可以思考下~