---
layout: post
title:  "高分辨率显示器时开机动画无法显示问题分析"
date:   2016-06-18 05:50:05
author: HappySeeker
categories: Graphic
---

# 问题现象

最近在环境中，使用23寸显示器时，发现系统的开机动画无法显示，图形环境正常，接19寸显示器时正常。

在内核启动参数中加上调试信息开关：plymouth:debug，查看plymouth的日志，发现日志中有如下报错：

	[ply-boot-splash.c]                          ply_boot_splash_show:showing splash screen
	
	[./plugin.c]                            show_splash_screen:loading lock image
	[./plugin.c]                            show_splash_screen:loading box image
	[./plugin.c]                            show_splash_screen:loading corner image
	[./plugin.c]                            show_splash_screen:loading header image
	[./plugin.c]                                     view_load:loading entry
	[./plugin.c]                                     view_load:loading animation
	[./plugin.c]                                     view_load:loading progress animation
	[./plugin.c]                                     view_load:loading throbber
	[./plugin.c]                            show_splash_screen:starting boot animations
	[./plugin.c]                                 map_to_device:could not map fb device: Invalid argument
	[./plugin.c]                                 map_to_device:could not map fb device: Invalid argument
	[./plugin.c]                                 map_to_device:could not map fb device: Invalid argument
	[./plugin.c]                                 map_to_device:could not map fb device: Invalid argument
	[./plugin.c]                                 map_to_device:could not map fb device: Invalid argument

# 分析过程
## 日志分析

关键报错信息：

	[./plugin.c]                                 map_to_device:could not map fb device: Invalid argument

分析具体的代码，确认是在`map_to_device`中，对/dev/fb0设备调用mmap接口出错，具体的错误为： `Invalid argument`，代码调用如下：

##　Invalid argument的含义

先看看mmap接口的定义：

 	void *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset);


从mmap的man手册可以看到`Invalid argument`解释如下：

	EINVAL We don't like addr, length, or offset (e.g., they are too large, or not aligned on a page boundary).

意思是，当传入的虚拟地址(addr)、长度(length)或者文件偏移(offset)不对时，可能返回这个错误，比如：
- length太长，超出了文件的范围
- 虚拟地址或文件偏移没有以page对齐

再看看，plymouth中传入的参数(通过gdb调试获得)：
- addr为NULL，表示希望内核自己分配
- length为1920*1080*4
- offset为0

显然，addr和offset都不会有问题，length是否过大，取决于/dev/fb0设备初始化时，分配的framebuffer的大小，理论上这个大小应该是根据当前的分辨率来分配的，也就是说应该跟这里是一致的，推测上应该不存在问题，但还需要分析相关代码证实。

## 内核mmap相关流程分析

从用户态的mmap接口如何进入内核的过程，在另一篇文章中介绍，这里不说明，对于fb设备来说，mmap流程最终会进入到fb_mmap()函数(3.0版本内核中，实现于`drivers/video/fbmem.c`文件中)，具体代码实现为：

	1348 static int
	1349 fb_mmap(struct file *file, struct vm_area_struct * vma)
	1350 {
	1351         struct fb_info *info = file_fb_info(file);
	1352         struct fb_ops *fb;
	1353         unsigned long mmio_pgoff;
	1354         unsigned long start;
	1355         u32 len;
	1356 
	1357         if (!info)
	1358                 return -ENODEV;
	1359         fb = info->fbops;
	1360         if (!fb)
	1361                 return -ENODEV;
	1362         mutex_lock(&info->mm_lock);
	1363         if (fb->fb_mmap) {
	1364                 int res;
	1360         if (!fb)
	1361                 return -ENODEV;
	1362         mutex_lock(&info->mm_lock);
	1363         if (fb->fb_mmap) {
	1364                 int res;
	1365                 res = fb->fb_mmap(info, vma);
	1366                 mutex_unlock(&info->mm_lock);
	1367                 return res;
	1368         }
	1369 
	1370         /*
	1371          * Ugh. This can be either the frame buffer mapping, or
	1372          * if pgoff points past it, the mmio mapping.
	1373          */
	1374         start = info->fix.smem_start;
	1375         len = info->fix.smem_len;
	1376         mmio_pgoff = PAGE_ALIGN((start & ~PAGE_MASK) + len) >> PAGE_SHIFT;
	1377         if (vma->vm_pgoff >= mmio_pgoff) {
	1378                 vma->vm_pgoff -= mmio_pgoff;
	1379                 start = info->fix.mmio_start;
	1380                 len = info->fix.mmio_len;
	1381         }
	1382         mutex_unlock(&info->mm_lock);
	1383 
	1384         vma->vm_page_prot = vm_get_page_prot(vma->vm_flags);
	1385         fb_pgprotect(file, vma, start);
	1386 
	1387         return vm_iomap_memory(vma, start, len);
	1388 }

从这个接口看，这个函数中没有返回EINVAL错误的地方，进入`vm_iomap_memory`函数看，有几处，但从代码逻辑上看，无法确认是哪里返回的，再深入看，其子函数中，还有很多的返回EINVAL错误的地方，无法确认，很难向下分析了。

## 提升内核日志打印级别

没有进一步信息可以分析了，考虑提升内核的日志打印级别，看看是否能有很多的信息。

在内核启动参数中加入loglevel=7，重新系统，发现dmesg中多了如下错误信息：

	[    5.359982] plymouthd:129 conflicting memory types 7f800000-7ffe9000 write-combining<->uncached-minus
	[    5.359993] reserve_memtype failed 0x7f800000-0x7ffe9000, track uncached-minus, req uncached-minus
	[    5.380528] plymouthd:129 conflicting memory types 7f800000-7ffe9000 write-combining<->uncached-minus
	[    5.380540] reserve_memtype failed 0x7f800000-0x7ffe9000, track uncached-minus, req uncached-minus
	[    5.400345] plymouthd:129 conflicting memory types 7f800000-7ffe9000 write-combining<->uncached-minus
	[    5.400353] reserve_memtype failed 0x7f800000-0x7ffe9000, track uncached-minus, req 

在内核中，搜索相关打印，并分析相关内核流程，可以确认是如下流程报的错误：

	fb_mmap
	  vm_iomap_memory
		io_remap_pfn_range
		  remap_pfn_range
			track_pfn_remap
			  reserve_pfn_range
				reserve_memtype
				  rbt_memtype_check_insert
					memtype_rb_check_conflict

`memtype_rb_check_conflict`报错流程代码如下：

	static int memtype_rb_check_conflict(struct rb_root *root,
					u64 start, u64 end,
					enum page_cache_mode reqtype,
					enum page_cache_mode *newtype)
	{
	...
	failure:
		printk(KERN_INFO "%s:%d conflicting memory types "
			"%Lx-%Lx %s<->%s\n", current->comm, current->pid, start,
			end, cattr_name(found_type), cattr_name(match->type));
		return -EBUSY;
	}

分析代码，涉及东西较多，其原理简单说就是：内核中对于设备(外设，如显卡)占用的资源(如IO内存)，都以resource方式维护，对于显卡占用的内存(空间)来说，包含了内存属性(memtype)，如WC(Write Combined)、UC(Uncached)等，每段内存资源都有其对应的memtype，这些memtype被组织成了一个红黑树，在为这些空间分配物理内存时，需要检查相应的memtype是否跟用户指定的type一致，如果存在冲突，就报错。

比如，这个问题中，

`write-combining`类型和`uncached-minus`类型就是明显冲突的，所以报错。这里不解释这些类型的具体含义，有兴趣可以去查一下X86 PAT相关资料。也就是说，这些类型是X86架构独有的。

## 尝试关闭PAT

既然memtype会有冲突，最直接的解决办法就是关闭PAT相关功能，内核提供了相应的参数：nopat，加入内核启动参数了，系统能启动，报错果然没有了，但是有更大的问题，图形界面根本起不来。

看来图形显示相关流程还是对PAT有一定的依赖，不能直接关闭。这里也不继续深究和发散了，接下来还是需要回到原来的问题上来。

## memtype为何冲突？

这可能是本故障的核心问题。要解答就需要深入分析相关代码流程了，首先看看plymouth在调用mmap时要求的memtype是什么，从报错打印看，应该是`uncached-minus`类型。

显然mmap接口中并没有相关参数，需要分析内核相关驱动中是否有相关设置，分析代码，发现在如下流程中确实有设置`_PAGE_CACHE_MODE_UC_MINUS`属性：

	static inline void fb_pgprotect(struct file *file, struct vm_area_struct *vma,
					unsigned long off)
	{
		unsigned long prot;
	
		prot = pgprot_val(vma->vm_page_prot) & ~_PAGE_CACHE_MASK;
		if (boot_cpu_data.x86 > 3)
			pgprot_val(vma->vm_page_prot) =
				prot | cachemode2protval(_PAGE_CACHE_MODE_UC_MINUS);
	}

调用流程为:

	fb_mmap
	  fb_pgprotect

接下来，就要分析fb设备对应内存为何为`write-combining`类型了。

## fb设备内存为何为WC

这个问题就比较麻烦了，需要分析fb设备驱动的实现，因为这个内存是在fb设备初始化时分配的，不同的环境使用的fb驱动不一样(我的环境中用的psbfb驱动，环境中有两种不同实现，版本也比较老)，相关流程和原理比较复杂，这里不啰嗦了，直接揭示原因了。

psbfb驱动中，分配fb设备内存(就是Framebuffer)的流程为：

	fb_probe
	  drm_fb_helper_single_fb_probe
		psbfb_create
		  MRSTLFBAllocBuffer

关键代码如下：

`psbfb_create`函数中，关键代码：

	303 static int psbfb_create(struct psb_fbdev * fbdev, struct drm_fb_helper_surface_size * sizes)
	304 {
	...
	338         size = mode_cmd.pitch * mode_cmd.height;
	339         aligned_size = ALIGN(size, PAGE_SIZE);
	...
	343         if (aligned_size > pg->stolen_size) {
	344                 /* 
	345                  * allocate new buffer if the request size is larger than
	346                  * the stolen memory size
	347                  */
	348                 ret = MRSTLFBAllocBuffer(dev, aligned_size, &buffer);
	349                 if (ret) {
	350                         ret = -ENOMEM;
	351                         goto out_err0;
	352                 }
	353                 dev_priv->fb_reloc = buffer;
	354         }
	...
	}

大概意思是：当需要分配的内存大小>stolen_size的时候，就调用MRSTLFBAllocBuffer分配内存，而MRSTLFBAllocBuffer实现的关键代码如下：
	
	1568 int MRSTLFBAllocBuffer(struct drm_device *dev, IMG_UINT32 ui32Size, MRSTLFB_BUFFER **ppBuffer)
	...
	1576         pvBuf = __vmalloc( ui32Size, GFP_KERNEL | __GFP_HIGHMEM | __GFP_ZERO, __pgprot((pgprot_val(PAGE_KERNEL ) & ~_PAGE_CACHE_MASK) | _PAGE_CACHE_WC) );
	...
	}

注意最后设置的_PAGE_CACHE_WC，就对应于`write-combining`类型。就能解释冲突了。

## stolen内存

但什么情况下会走到这个流程来呢？关键就在于如下判断：

	343         if (aligned_size > pg->stolen_size) {

这里的aligned_size就是请求分配的内存大小再进行page对齐后的大小，其实就是分辨率*4，然后进行page对齐。

那stolen_size又是什么呢？

这个问题又复杂了，需要深入分析psbfb驱动的源代码。这里也不罗嗦了。简单解释为：

在Intel的这个系列的显卡中，有个stolen内存的概念，stolen内存实际上就是在**显存**上预留的一段内存，用作Framebuffer，这段显存是硬件上预先设置好的(在显卡的PCI配置空间中设置，可能是硬件自身设置好的，也可能是BIOS设置的)，大小是固定的。

当显示器分辨率较低时，需要申请较小的Framebuffer，小于stolen_size时，则直接使用**显存**上的stolen内存，这样可以不占用物理内存，而且效率更高。
当显示器分辨率较高时(比如这个问题环境中的1920*1080)，需要申请较大的Framebuffer，超过stolen_size时，stolen内存不足，就不能使用stolen内存了，因为Framebuffer是不支持非连续内存的(类似于分散聚集)，所以只能在内存(GTT)上分配Framebuffer，而此时分配时设置了WC属性，所以冲突。

那这个环境中，stolen内存到底有多大呢，从代码看，这个大小是从pci配置空间中读出来的，也就是硬件固化的，光看代码肯定是不知道其具体大小的，怎么办呢，要么在内核中打点，要么。。。

当然，用SystemTap也许更方便，个人习惯用这个。

这个变量查看比较复杂，SystemTap脚本也不好写，具体打印的脚本代码如下，供大家参考：

	#!/usr/bin/stap

	probe begin {
	        printf("mmap system call stap begin!\n")
	}
	
	probe kernel.statement("fb_mmap@drivers/video/fbmem.c:1357")
	{
	        if (pid() == target()) {
	        p = @cast(@cast($info->par, "struct psb_fbdev")->psb_fb_helper->dev->dev_private, "struct drm_psb_private")->pg->stolen_size
	        printf("%s(%d) stolen_size=%d \n",execname(),pid(),stolen_size)
	        }
	}
	
	probe end {
	        printf("mmap system call stap end!\n")
	}

结果如下：

	-bash-4.1# stap -c /home/jb/a.out mmap.stp 
	mmap system call stap begin!
	a.out(17337) stolen_size=8122368 

可见，大小为8122368，当显示器分辨率为1920*1080时，其需要的fb大小为：

	1920*1080*4=8294400 > 8122368

所以，可以介绍为什么23寸(分辨率为1920\*1080)显示器时，开机动画显示异常；而19寸显示器(分辨率为1440\*900)时可以显示了。

# 结论和解决

结论是：显卡驱动代码问题。

解决方法：我的环境中这个显卡驱动代码也老了，新版本内核中相应代码架构变化很大，无法直接移植。直接的解决方法为：修改psbfb显卡驱动代码，在内存(GTT)中分配Framebuffer时去掉WC标记即可。

问题本质很简单，但分析过程很曲折。

# 遗留问题

理论上，plymouth显示时，应该使用drm驱动才对，不应该使用直接操作Framebuffer的方式。为什么这里为使用fb设备。

原因有很多，之前也分析过类似的问题，现实是很多情况下drm不可用，比如Xorg占用了drm设备时。这种情况下，直接操作Framebuffer的方式作为back手段，使用是合理的，所以只要解决fb设备的问题即可。drm驱动没有使用的原因，这里不再深入研究了。
