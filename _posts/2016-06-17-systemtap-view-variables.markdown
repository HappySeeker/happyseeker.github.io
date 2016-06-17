---
layout: post
title:  "SystemTap使用技巧-查看变量"
date:   2016-06-17 06:02:01
author: HappySeeker
categories: Kernel
---

# 背景

最近分析内核驱动故障时，重新又开始用SystemTap了，这个用起来最方便。

对我个人来说，SystemTap中最常用的就是*查看变量(包括函数入参、局部变量和全局变量)*操作了，查看变量看似很简单，但其中也有不少坑。

# 基本用法

SystemTap脚本中，查看变量的方法很简单，就直接$变量名就可以了，比如

	#!/usr/bin/stap
	probe kernel.function("do_mmap_pgoff")
	       printf("%s(%d) %s (%s)\n",execname(),pid(),$pgoff,$len)
	}

其中的$pgoff和$len就是函数的入参。很简单。

# 结构体成员查看

方法也比较简单，直接**结构体变量名->成员名**就可以了，但需要注意的是，这里的结构体变量无论是指针还是非指针，都只能用->, 不能用**.**。 如：

	#!/usr/bin/stap
	probe kernel.statement("fb_mmap@drivers/video/fbmem.c:1357")
	{
			printf("%s(%d) %x\n",execname(),pid(),$info->node)
	}

# void指针指向的结构体成员

场景如：

	struct fb_info {
	...
		void *par;
	...
	}

实际中，par指向psb_fbdev对象

	struct psb_fbdev {
		struct drm_fb_helper psb_fb_helper;
		struct psb_framebuffer pfb;
	};
	

此时，如果想要通过par指针来访问psb_fbdev对象的成员(如pfb)，这个就比较麻烦了，因为这里的par是个void类型的指针，SystemTap无法知道其具体指向的是什么对象，也就没有办法直接访问其成员了。

这种情况，在内核中还是比较常见的，这个是一种典型的解耦的方法，用得比较多。

此时，如果需要达到上述目的，还需要进行类型转换，即使用@cast接口，示例如下：

	#!/usr/bin/stap
	probe kernel.statement("fb_mmap@drivers/video/fbmem.c:1357")
	{
 		p = @cast($info->par, "struct psb_fbdev")->pfb
		printf("%s(%d) %d\n",execname(),pid(),p)
	}

更复杂一点的情况，pfb中如果还有void指针，如果要要继续访问其成员，则需要继续进行类型转换，如：

	#!/usr/bin/stap
	probe kernel.statement("fb_mmap@drivers/video/fbmem.c:1357")
	{
 		p = @cast(@cast($info->par, "struct psb_fbdev")->psb_fb_helper->dev->dev_private, "struct drm_psb_private")->pg->stolen_size
		printf("%s(%d) %d\n",execname(),pid(),p)
	}
