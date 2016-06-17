---
layout: post
title:  "SystemTap使用技巧-找代码"
date:   2016-06-17 17:52:13
author: HappySeeker
categories: Kernel
---

# 背景

使用SystemTap打点时，首先要确认探测点，但有时我们可能只知道探测点的函数名称，不知道其具体的实现位置，典型的就是系统调用，内核中实现系统调用的方式有点特别，比如`sys_open()`，在内核中找不到`sys_open()`的函数的实现。

另一方面，由于代码编译时做的优化，一些代码行并不能直接做为探测点，此时需要有方法来获取可用的探测点。


# 查找函数位置

使用stap -l 'kernel.function("<函数名>")'可用查看函数定义的位置，示例如：

	-bash-4.1# stap -l 'kernel.function("sys_open")'
	kernel.function("sys_open@fs/open.c:1016")

实际`sys_open()`定义如下：

	1016 SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, int, mode)
	1017 {
	1018         long ret;
	1019 
	1020         if (force_o_largefile())
	1021                 flags |= O_LARGEFILE;
	1022 
	1023         ret = do_sys_open(AT_FDCWD, filename, flags, mode);
	1024         /* avoid REGPARM breakage on x86: */
	1025         asmlinkage_protect(3, ret, filename, flags, mode);
	1026         return ret;
	1027 }

# 查找探测点

使用stap -L 可以查看具体可用的探测点，已经每个探测点对应的可以探测的变量信息。示例如：

	-bash-4.1# stap -L 'kernel.function("sys_mmap*")'
	kernel.function("sys_mmap_pgoff@mm/mmap.c:1090") $addr:long unsigned int $len:long unsigned int $prot:long unsigned int $flags:long unsigned int $fd:long unsigned int $pgoff:long unsigned int

示例2：

	-bash-4.1# stap -L 'kernel.statement("sys_open@fs/open.c:*")'
	kernel.statement("sys_open@fs/open.c:1017") $filename:char const* $flags:int $mode:int $ret:long int
	kernel.statement("sys_open@fs/open.c:1023") $filename:char const* $flags:int $mode:int $ret:long int
	kernel.statement("sys_open@fs/open.c:1025") $filename:char const* $flags:int $mode:int $ret:long int
	kernel.statement("sys_open@fs/open.c:1027") $filename:char const* $flags:int $mode:int $ret:long int

