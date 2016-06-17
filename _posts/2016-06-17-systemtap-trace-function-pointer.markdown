---
layout: post
title:  "SystemTap使用技巧-跟踪函数指针"
date:   2016-06-17 17:52:12
author: HappySeeker
categories: Kernel
---

# 背景

内核中(应该说是C代码中)经常会使用函数指针，原因很简单，依赖倒置原则，利用函数指针可以实现在运行时动态调整依赖，解除各模块直接的耦合，增加代码(框架)的扩展性，这里不关注~

在阅读代码是，遇到函数指针，通常会比较难准确找到其具体对应的函数，通常需要在代码中去搜索相应的关键字，比较麻烦，而且不容易找到。

在内核中问题尤其明显，内核中为实现层间的隔离，使用了大量的函数指针，函数指针由底层实现，不同底层会有不同的实现，比如一个显卡驱动实现的函数指针，在内核中有大量的显卡驱动，每个显卡驱动都有自己的实现，光阅读代码，很难找到自己的环境中具体使用的是那个函数实现，除非你非常了解硬件的相关信息。

示例如mmap流程中的如下代码：

	1288                 get_file(file);
	1289                 error = file->f_op->mmap(file, vma);

如果要找mmap对应的具体实现，就比较麻烦。

这种问题，如果在用户态，通常用gdb单步跟踪一下，就能知道函数指针的具体实现了。但是，对于内核，就不太容易了，毕竟内核不能直接用gdb调试。

还好的是，我们有SystemTap，使用如下介绍的方法可以跟踪具体的函数指针。


# 内核中跟踪函数指针

道理很简单，找到相应的probe点，然后通过SystemTap提供的symname接口解析函数指针对应的符号。
示例如下：

	probe kernel.statement("mmap_region@mm/mmap.c:1288")
	{
	        if (pid() == target()) {
	                printf("%s(%d) the mmap func point to:%s  \n",execname(),pid(),symname($file->f_op->mmap))
	        }
	}

结果如下：

	-bash-4.1# stap -c /home/jb/a.out trace-pointer.stp 
	a.out(17038) the mmap func point to:ext4_file_mmap 
	a.out(17038) the mmap func point to:ext4_file_mmap  
	a.out(17038) the mmap func point to:fb_mmap  
	a.out(17038) the mmap func point to:fb_mmap  
