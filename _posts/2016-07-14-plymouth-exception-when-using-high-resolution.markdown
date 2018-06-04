---
layout: post
title:  "高分辨率显示器时开机动画无法显示问题分析"
date:   2016-07-14 06:50:05
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

所以，可以解释为什么23寸(分辨率为1920\*1080)显示器时，开机动画显示异常；而19寸显示器(分辨率为1440\*900)时可以显示了。

# 尝试去掉__vmalloc中的`_PAGE_CACHE_WC`标记

尝试去掉cdv(Atom处理器上的集成显卡)驱动中，__vmalloc中的`_PAGE_CACHE_WC`标记后，重新编译内核，测试结果**无效**，仍然报memtype冲突。

问题另有原因？代码地方没有找对？(后续的事实证明，路还长着呢~，远未到终点)

无论问题在哪，memtype冲突都是目前最直接的现象，需要首先定位。

这个测试说明，这里的memtype不是在__vmalloc中设置的，另有别处。重新查找代码发现在psbfb驱动初始化ttm的时候，设置了相关的ttm属性，相关代码如下(`./imgv/psb_buffer.c:48`)：

	 40 /*
	 41  * MSVDX/TOPAZ GPU virtual space looks like this
	 42  * (We currently use only one MMU context).
	 43  * PSB_MEM_MMU_START: from 0x00000000~0xe000000, for generic buffers
	 44  * TTM_PL_CI: from 0xe0000000+half GTT space, for camear/video buffer sharing
	 45  * TTM_PL_RAR: from TTM_PL_CI+CI size, for RAR/video buffer sharing
	 46  * TTM_PL_TT: from TTM_PL_RAR+RAR size, for buffers need to mapping into GTT
	 47  */
	 48 static int psb_init_mem_type(struct ttm_bo_device *bdev, uint32_t type,
	 49                              struct ttm_mem_type_manager *man)
	 50 {
	 51
	 52         struct drm_psb_private *dev_priv =
	 53             container_of(bdev, struct drm_psb_private, bdev);
	 54         struct psb_gtt *pg = dev_priv->pg;
	 55
	 56         switch (type) {
	 57         case TTM_PL_SYSTEM:
	 58                 man->flags = TTM_MEMTYPE_FLAG_MAPPABLE;
	 59                 man->available_caching = TTM_PL_FLAG_CACHED |
	 60                     TTM_PL_FLAG_UNCACHED | TTM_PL_FLAG_WC;
	 61                 man->default_caching = TTM_PL_FLAG_CACHED;
	 62                 break;
	 63         case DRM_PSB_MEM_MMU:
	 64                 man->func = &ttm_bo_manager_func;
	 65                 man->flags = TTM_MEMTYPE_FLAG_MAPPABLE |
	 66                     TTM_MEMTYPE_FLAG_CMA;
	 67                 man->gpu_offset = PSB_MEM_MMU_START;
	 68                 man->available_caching = TTM_PL_FLAG_CACHED |
	 69                     TTM_PL_FLAG_UNCACHED | TTM_PL_FLAG_WC;
	 70                 man->default_caching = TTM_PL_FLAG_WC;
	 71                 break;
	 72         case TTM_PL_CI:
	 73                 man->func = &ttm_bo_manager_func;
	 74                 man->flags = TTM_MEMTYPE_FLAG_MAPPABLE |
	 75                         TTM_MEMTYPE_FLAG_FIXED;
	 76                 man->gpu_offset = pg->mmu_gatt_start + (pg->ci_start);
	 77                 man->available_caching = TTM_PL_FLAG_UNCACHED;
	 78                 man->default_caching = TTM_PL_FLAG_UNCACHED;
	 79                 break;
	 80         case TTM_PL_RAR:        /* Unmappable RAR memory */
	 81                 man->func = &ttm_bo_manager_func;
	 82                 man->flags = TTM_MEMTYPE_FLAG_MAPPABLE |
	 83                         TTM_MEMTYPE_FLAG_FIXED;
	 84                 man->available_caching = TTM_PL_FLAG_UNCACHED;
	 85                 man->default_caching = TTM_PL_FLAG_UNCACHED;
	 86                 man->gpu_offset = pg->mmu_gatt_start + (pg->rar_start);
	 87                 break;
	 88         case TTM_PL_TT: /* Mappable GATT memory */
	 89                 man->func = &ttm_bo_manager_func;
	 90 #ifdef PSB_WORKING_HOST_MMU_ACCESS
	 91                 man->flags = TTM_MEMTYPE_FLAG_MAPPABLE;
	 92 #else
	 93                 man->flags = TTM_MEMTYPE_FLAG_MAPPABLE |
	 94                     TTM_MEMTYPE_FLAG_CMA;
	 95 #endif
	 96                 man->available_caching = TTM_PL_FLAG_CACHED |
	 97                     TTM_PL_FLAG_UNCACHED | TTM_PL_FLAG_WC;
	 98                 man->default_caching = TTM_PL_FLAG_WC;
	 99                 man->gpu_offset = pg->mmu_gatt_start + (pg->rar_start + dev_priv->rar_region_size);
	...
	}
进入该函数的调用链为：

	psb_driver_load
	  psb_do_init
	    ttm_bo_init_mm
		  init_mem_type
		    psb_init_mem_type

这段初始化代码中，有好几个地方设置了`TTM_PL_FLAG_WC`标记，分析相关的初始化代码，发现默认使用的是`DRM_PSB_MEM_MMU`类型，将该类型中的`default_caching`修改为`TTM_PL_FLAG_UNCACHED`，重新验证，结果还是失败。

进一步将这里的所有的WC标记都修改为`TTM_PL_FLAG_UNCACHED`，测试结果还是失败。

再进一步将cdv驱动中所有涉及`TTM_PL_FLAG_WC`标记地方都修改了，测试结果仍然失败，还是冲突。这下有点迷茫了，难道之前的想法都错了？

# 从现场重新分析

只能从现场重新分析了，从报“memtype冲突”的代码中，逐级向上分析，发现除了上述的标记外，还可以通过`ioremap_wc`类似的接口来设置WC标记。重新分析代码，发现cdv驱动中，stolen内存(显存)正是通过`ioremap_wc`接口分配的，代码如下：

	  73 int psb_gtt_init(struct psb_gtt *pg, int resume)
	  74 {
	  ...
	 171         pg->vram_addr = ioremap_wc(pg->stolen_base, stolen_size);
	 172         if (!pg->vram_addr) {
	 173                 DRM_ERROR("Failure to map stolen base.\n");
	 174                 ret = -ENOMEM;
	 175                 goto out_err;
	 176         }
	  ...
	      }

这里的`vram_addr`即为显存(stolen内存)在CPU虚拟地址空间中映射的虚拟地址，其中的`stolen_base`和`stolen_size`都是从显卡的PCI配置空间中读取的。

进入该函数的调用流程为

	psb_driver_load
	  psb_gtt_init
	    ioremap_wc // 这里也要分配内存(stolen) ---》screen_base 内核态？
	      reserve_memtype

这段代码表明，stolen内存的属性中设置了WC标记，看似可疑，但是根据之前的分析，当使用高分辨率显示器时，由于需要Framebuffer大于stolen内存的大小，所以就没有使用stolen内存么？那stolen内存设置为WC属性跟这个问题有什么关系？

看似没有关系，但那只是因为没有理解透彻，需要搞清楚mmap的过程中到底有没有使用到stolen内存，需要继续深入挖掘。

# 再次分析fb_mmap流程

问题是在对fb设备的mmap操作中报出来的，重新分析相关代码，关键代码如下：

	static int
	fb_mmap(struct file *file, struct vm_area_struct * vma)
	{
	...
		//start为mmap映射的物理内存地址的起始值，len为长度。
		start = info->fix.smem_start;
		len = info->fix.smem_len;
	...
		fb_pgprotect(file, vma, start);
		//将start和len传入，用于分配物理内存
		return vm_iomap_memory(vma, start, len);
	}

这里需要仔细理解一下mmap的作用，mmap是用来将“设备/文件”的内容映射到虚拟地址空间中，映射后，可以直接读写得到的虚拟地址，即可实现直接读写“设备/文件”的内容的目的。这里fb驱动中，设备就对应于Framebuffer(就是一段内存，用于直接显示于屏幕上)，也就是说，得到mmap映射后的虚拟地址后，直接向这个地址范围copy开机动画的数据，就可以直接显示开机动画了。

那这个数据最终拷贝到哪里去了呢？mmap得到的是虚拟地址，会映射到相应的物理地址上，这个就是mmap本质的作用，所以，这段代码中的start和len其实就标识了最终数据的目的地。

看看这段代码，发现，这里要分配物理内存的start和len都是从`info->fix`读取出来的，`info->fix`是具体驱动相关的，应该是在驱动初始化时赋值的。

搜搜代码发现在cdv驱动的初始化代码中，的确对其进行了初始化：

	./psb_fb.c:384:	info->fix.smem_start = dev->mode_config.fb_base;

继续看代码，这里的fb_base其实就是stolen_base，也就是stolen内存。如此看来fb_mmap仍然映射的是stolen内存，所以如果stolen内存属性为WC，就可能冲突了。

将前面的ioremap_wc修改为ioremap_nocache后，没有再报冲突问题，但是图形界面就无法启动了，问题更严重了，这个问题后面继续分析。这里需要先解释另一个问题：为什么低分辨率时，没有报冲突呢，也是使用stolen内存吧？

是的，确实如此，这里让人很困惑，经过打点分析，确认是这个代码流程中，有些奇怪的条件组合，当分辨率低时，刚好就跳过了类型检查，所以没有报冲突，但高分辨率时会报，具体打点过程也比较曲折，这里不详述了。后面着重分析“图形界面就无法启动”的问题。

# 问题变严重了

在消除了memtype 冲突的问题后，本期待问题就这样解决了，但是路还长着呢~

现在的现象的黑屏了，图形界面无法显示。涉及到更底层、更本质的问题了。再仔细梳理下上面的分析过程，发现有个大问题，在`psbfb_create`过程中，分明判断了大于`stolen_size`的情况，并使用__vmalloc为Framebuffer在物理内存中分配了新的内存，为什么`fb_mmap`中，还要继续使用stolen内存呢？这里有明显的冲突。

的确，这里确实有问题，在驱动初始化时，没有使用stolen内存作为Framebuffer，此时mmap中继续想stolen内存中拷贝数据，这里显然有问题，可能导致未知现象。那fb驱动中为何还要使用stolen内存呢？这个问题又复杂了~

简单说，是因为这个内核版本中的驱动太老，没有实现相应的功能，那到底需要实现什么样的功能呢？

需要：显卡驱动实现的针对fb设备(如/dev/fb1)的专门的mmap接口。而不只是通用的`fb_mmap`接口。见如下通用的fb驱动实现的`fb_mmap()`实现代码：

	static int
	fb_mmap(struct file *file, struct vm_area_struct * vma)
	{
		struct fb_info *info = file_fb_info(file);
		struct fb_ops *fb;
		unsigned long mmio_pgoff;
		unsigned long start;
		u32 len;

		if (!info)
			return -ENODEV;
		fb = info->fbops;
		if (!fb)
			return -ENODEV;
		mutex_lock(&info->mm_lock);
		// 如果显卡驱动中实现了具体的fb_mmap接口，就调用该接口，此时就不会继续执行后面的代码了，也就不会使用stolen内存了。
		if (fb->fb_mmap) {
			int res;
			res = fb->fb_mmap(info, vma);
			mutex_unlock(&info->mm_lock);
			return res;
		}
	...
	}

如果显卡驱动不实现`fb_mmap`接口，那么就只能使用fb驱动实现的通用接口了，那就会使用stolen内存。那显卡驱动应该怎么实现`fb_mmap`接口呢，如下是新版本的gma500驱动中的实现方式，供参考：

	static int psbfb_mmap(struct fb_info *info, struct vm_area_struct *vma)
	{
		struct psb_fbdev *fbdev = info->par;
		struct psb_framebuffer *psbfb = &fbdev->pfb;

		if (vma->vm_pgoff != 0)
			return -EINVAL;
		if (vma->vm_pgoff > (~0UL >> PAGE_SHIFT))
			return -EINVAL;

		if (!psbfb->addr_space)
			psbfb->addr_space = vma->vm_file->f_mapping;
		/*
		 * If this is a GEM object then info->screen_base is the virtual
		 * kernel remapping of the object. FIXME: Review if this is
		 * suitable for our mmap work
		 */
		/*设置vm_ops，其中包括page fault的处理接口，也就是说这里的vma对应的物理内存通过page fault来分配，而不直接分配*/
		vma->vm_ops = &psbfb_vm_ops;
		vma->vm_private_data = (void *)psbfb;
		vma->vm_flags |= VM_IO | VM_MIXEDMAP | VM_DONTEXPAND | VM_DONTDUMP;
		return 0;
	}

这里可以看出，这种实现方式下，vma对应的物理内存是通过page fault来分配的，而fb驱动实现的`fb_mmap()`接口中是直接分配物理内存的(stolen内存)，两者实现方式有明显的差别。

问题：通过缺页分配的物理内存如何能与之前在`MRSTLFBAllocBuffer`中通过`vmalloc`的方式分配的物理内存对应起来呢，这个就需要在page fault的处理函数中做具体的处理了，这里面的机制比较复杂，涉及GEM/TTM相关的内存管理方式，这里就不详述了，在另外的文章中做单独的讲解。感兴趣的同学可以自己去看看psbfb_vm_fault(gma500驱动的vm_ops中的page fault处理函数)。

总的来说，当前版本内核(3.0)中自带的cdv驱动中完全没有实现这套机制，而高版本内核中cdv驱动被完全合并到gma500驱动中处理，而且相关的驱动都移动到了drm目录(原来在staging目录中)，架构上也有比较大的变化，反向从高版本内核移植相关功能的工作量巨大。简单尝试了将gma500驱动移植到低版本中，编译问题一大堆，暂时没有时间去做深入移植工作了。

# 为什么不使用drm驱动
理论上，plymouth显示时，应该使用drm驱动才对，不应该使用直接操作Framebuffer的方式。为什么这里为使用fb设备？

原因是：问题环境中的plymouth版本比较低，没有针对cdv显卡(Intel Atom集成显卡cedarview)实现专门的drm驱动接口，看看plymouth这个版本中renderers目录中的内容：

	[root@955 jiangbiao]#ll plymouth-0.8.3/src/plugins/renderers/drm/
	total 152
	-rw-r--r-- 1 4153 4153  1470 Jan 15  2010 Makefile.am
	-rw-r--r-- 1 4153 4153 25184 May  7  2010 Makefile.in
	-rw-r--r-- 1 root 4153 34548 Jun  7 14:54 plugin.c
	-rw-r--r-- 1 4153 4153 32451 Apr 14  2010 plugin.c.handle-cloned-outputs
	-rw-r--r-- 1 4153 4153  2511 Sep 29  2009 ply-renderer-driver.h
	-rw-r--r-- 1 4153 4153 10042 Apr  1  2010 ply-renderer-i915-driver.c
	-rw-r--r-- 1 4153 4153  1184 Sep 29  2009 ply-renderer-i915-driver.h
	-rw-r--r-- 1 4153 4153  9177 Apr  1  2010 ply-renderer-nouveau-driver.c
	-rw-r--r-- 1 4153 4153  1199 Sep 29  2009 ply-renderer-nouveau-driver.h
	-rw-r--r-- 1 4153 4153  9874 Apr  1  2010 ply-renderer-radeon-driver.c
	-rw-r--r-- 1 4153 4153  1194 Sep 29  2009 ply-renderer-radeon-driver.h

可以看出，这个版本的plymouth中只针对intel i915、NVIDIA、和AMD Radeon显卡做了实现，具体实现的内容，主要是利用drm库接口分配Framebuffer，然后设置Framebuffer。以Radeon显卡为例，其主要实现接口有:

	static uint32_t
	create_buffer (ply_renderer_driver_t *driver,
	               unsigned long          width,
	               unsigned long          height,
	               unsigned long         *row_stride)
	{
	  struct radeon_bo *buffer_object;
	  ply_renderer_buffer_t *buffer;
	  uint32_t buffer_id;

	  *row_stride = ply_round_to_multiple (width * 4, 256);
	  /*调用显卡驱动bo创建接口，分配内存，创建buffer，实际以bo的形式，这里指定的GTT，说明是在内存(非显存)上分配*/
	  buffer_object = radeon_bo_open (driver->manager, 0,
	                                  height * *row_stride,
	                                  0, RADEON_GEM_DOMAIN_GTT, 0);

	  if (buffer_object == NULL)
	    {
	      ply_trace ("Could not allocate GEM object for frame buffer: %m");
	      return 0;
	    }
	  /*调用drm驱动接口，将刚分配的buffer设置为Framebuffer*/
	  if (drmModeAddFB (driver->device_fd, width, height,
	                    24, 32, *row_stride, buffer_object->handle,
	                    &buffer_id) != 0)
	    {
	      ply_trace ("Could not set up GEM object as frame buffer: %m");
	      radeon_bo_unref (buffer_object);
	      return 0;
	    }

	  buffer = ply_renderer_buffer_new (driver,
	                                    buffer_object, buffer_id,
	                                    width, height, *row_stride);
	  buffer->added_fb = true;
	  ply_hashtable_insert (driver->buffers,
	                        (void *) (uintptr_t) buffer_id,
	                        buffer);

	  return buffer_id;
	}


这里可以看出没有针对cdv显卡的实现。所以在这个环境中，plymouth无法使用drm方式显示。

# 升级plymouth

市面上这么多的显卡，难道plymouth需要针对每种显卡都实现相应的create_buffer接口么？

当然这不太现实，新版本plymouth中的解决方案是，实现了一个generic的接口，在如下文件中

	src/plugins/renderers/drm/ply-renderer-generic-driver.c

实现的create_buffer接口如下：

	static uint32_t
	create_buffer (ply_renderer_driver_t *driver,
	               unsigned long          width,
	               unsigned long          height,
	               unsigned long         *row_stride)
	{
	  ply_renderer_buffer_t *buffer;
	  //分配Framebuffer，调用驱动的CREATE_DUMB接口
	  buffer = ply_renderer_buffer_new (driver, width, height);

	  if (buffer == NULL)
	    {
	      ply_trace ("Could not allocate GEM object for frame buffer: %m");
	      return 0;
	    }
	  // 将新分配的Framebuffer设置为显卡的Framebuffer
	  if (drmModeAddFB (driver->device_fd, width, height,
	                    24, 32, buffer->row_stride, buffer->handle,
	                    &buffer->id) != 0)
	    {
	      ply_trace ("Could not set up GEM object as frame buffer: %m");
	      ply_renderer_buffer_free (driver, buffer);
	      return 0;
	    }
	...
	}

关键调用了`ply_renderer_buffer_new`，主要实现过程如下：

	static ply_renderer_buffer_t *
	ply_renderer_buffer_new (ply_renderer_driver_t *driver,
	                         uint32_t               width,
	                         uint32_t               height)
	{
	  ply_renderer_buffer_t *buffer;
	...
	  //通过ioctl的DRM_IOCTL_MODE_CREATE_DUMB命令分配Framebuffer。
	  if (drmIoctl (driver->device_fd,
	                DRM_IOCTL_MODE_CREATE_DUMB,
	                &create_dumb_buffer_request) < 0)
	    {
	      free (buffer);
	      ply_trace ("Could not allocate GEM object for frame buffer: %m");
	      return NULL;
	    }
	...
	}

这两段代码说明：generic接口依赖于底层驱动(内核中特定显卡的drm驱动)的CREATE_DUMB接口来为Framebuffer分配内存。而前面Radeon的接口实现直接使用的是`radeon_bo_open`接口。

将问题环境中的plymouth升级至新版本后(环境中没有使用systemd，升级过程还是比较麻烦，需要修改initramfs中的相关内容，具体不详述了)，发现还是没有正常加载drm.so，继续分析看看原因。

继续看看内核中的CREATE_DUMB接口实现：

	DRM_IOCTL_DEF(DRM_IOCTL_MODE_CREATE_DUMB, drm_mode_create_dumb_ioctl, DRM_CONTROL_ALLOW|DRM_UNLOCKED),

通用接口为`drm_mode_create_dumb_ioctl`，该接口会调用底层显卡驱动实现的接口：

	dev->driver->dumb_create(file_priv, dev, args);

看看对应内核版本(3.0)中的cdv驱动，其中并没有实现dumb_create接口：

	1812 static struct drm_driver driver = {
	1813         .driver_features = DRIVER_HAVE_IRQ | DRIVER_IRQ_SHARED | \
	1814                            DRIVER_IRQ_VBL | DRIVER_MODESET,
	1815         .load = psb_driver_load,
	1816         .unload = psb_driver_unload,
	1817
	1818         .ioctls = psb_ioctls,
	1819         .num_ioctls = DRM_ARRAY_SIZE(psb_ioctls),
	1820         .device_is_agp = psb_driver_device_is_agp,
	1821         .irq_preinstall = psb_irq_preinstall,
	1822         .irq_postinstall = psb_irq_postinstall,
	1823         .irq_uninstall = psb_irq_uninstall,
	1824         .irq_handler = psb_irq_handler,
	1825         .enable_vblank = psb_enable_vblank,
	1826         .disable_vblank = psb_disable_vblank,
	1827         .get_vblank_counter = psb_get_vblank_counter,
	1828         .firstopen = NULL,
	1829         .lastclose = psb_lastclose,
	1830         .open = psb_driver_open,
	1831         .postclose = PVRSRVDrmPostClose,
	1832         .suspend = PVRSRVDriverSuspend,
	1833         .resume = PVRSRVDriverResume,
	1834         .preclose = psb_driver_preclose,
	1835         .fops = {
	1836                  .owner = THIS_MODULE,
	1837                  .open = psb_open,
	1838                  .release = psb_release,
	...

而新版本内核(4.1)中的gma500驱动(新版本内核中，cdv驱动被整合到gma500驱动中)实现了相应接口

	static struct drm_driver driver = {
		...
		.dumb_create = psb_gem_dumb_create,
		.dumb_map_offset = psb_gem_dumb_map_gtt,
		.dumb_destroy = drm_gem_dumb_destroy,
		.ioctls = psb_ioctls,
		.fops = &psb_gem_fops,
		...
	};

从新版本的实现接口`psb_gem_dumb_create`看，这里的内存分配是通过gem实现的，内容相对比较复杂，不继续啰嗦了。有兴趣的话，可以再理解下`dumb_create`和类似于`radeon_bo_open`接口之间的区别，可以直接看看相关代码，这里也不啰嗦了。

从上面的分析可以得出的答案：因为内核中的cdv显卡驱动中没有实现`dumb_create`接口，所以升级plymouth后，还是无法使用drm，所以根源还在内核中的cdv驱动太老。移植到新版本后，理论上应该也能解决问题，但问题仍然是移植的工作量较大，投入产出比不高。

# 结论

结论是：内核中的显卡驱动没有实现mmap和create_dump等相关机制。

可能的解决方法：需要做如下两方面的工作：

1. 反向移植高版本的显卡驱动(或者自己在老版本中实现)，但工作量和难度都比很大。

2. 升级plymouth至高版本。这里需要注意问题环境中没有使用systemd机制，需要手工修改initramfs中的相关内容。
