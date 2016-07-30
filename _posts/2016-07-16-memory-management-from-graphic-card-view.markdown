---
layout: post
title:  "显卡内存管理机制及驱动实现(Intel gma500为例)"
date:   2016-07-22 06:15:24
author: HappySeeker
categories: Kernel
---

# 背景

在Intel Atom芯片的环境中分析故障时，顺便看了下该环境的显卡驱动中的内存管理相关机制，显卡为Atom CPU的集成显卡，低端产品，gma500显卡。本文主要关注该显卡驱动中的内存管理相关的实现原理，其他显卡原理与之类似，有一定的参考意义。

# CPU的内存管理

这个是个基础的话题，看似神秘，其实简单。简单描述如下：

CPU硬件中有个内存管理单元(MMU)，内存管理依赖于**页表**，页表是一个逻辑概念，并不存在这样一个硬件单元，其本质就是一段物理内存(内核中分配的，地址不定)，其中记录了**虚拟地址**与**物理地址**的映射关系，将这段内存的起始地址**告诉**(其实就是将其写入指定的寄存器)CPU(MMU)，然后CPU就可以利用这个页表来进行地址映射了。

页表中的**虚拟地址**与**物理地址**的映射关系是动态创建的(除了内核部分)，触发机制就是缺页异常(page fault)，也就是说物理内存是动态的通过缺页异常分配的，当用户使用malloc或mmap之类的接口分配内存时，实际得到的虚拟地址(对用户来说，只能看见虚拟地址)，实际的物理内存，是在实际访问该内存(写)时，硬件触发缺页异常，在缺页异常中分配的。

基本原理就介绍这些，关于Linux的内存管理机制足够写一本书，这里就不罗嗦了，资料也很多，以后有时间可以专门写一系列的文章来说明。

# GPU的内存管理

GPU也需要管理内存？当然。而且更复杂一些。

通常来说，对于GPU，其需要管理两类内存：主存(即CPU使用的物理内存)和显存(GPU自带的内存)。

GPU还需要管理主存？

是的，其实这个说法不太准确，应该说：GPU需要使用主存(也称GTT内存，从GPU的角度)，需要访问主存。实际的内存和显存的管理工作其实是内核做的，内核中有专门管理主存和显存的模块，通常为TTM或GEM(Intel)，内容太多，不详细说了。

GPU管理内存的方式本质上跟CPU相似，它也有自己的MMU，也有相应的页表，这些跟CPU都是独立的。也使用页表进行地址映射，页表也使用多级方式。

不同的是，GPU需要能访问主存，也就是说GPU的页表需要能映射主存中的物理地址，基本原理其实也跟CPU处理IO内存的方式相似，也是通过一定的映射(GPU有个GART表)将主存中的物理内存和GPU自己的显存映射到统一的物理地址空间(从GPU的角度。这个地址空间对CPU是不可见的)，也就是说，GPU看到的也就是自己的一个物理地址空间，GPU使用这些地址时，并不关心这些地址对应的是主存，还是显存。具体的映射工作是GPU硬件完成的，细节我们也不用关心，这里不描述。

有兴趣的可以自己搜一些资料看看。

# 如何使用内存？

前面说了基本的内存管理原理，这节从使用者的角度来继续说明。

大家平常都怎么使用内存？

内存使用有多种方式，通常有：

1. malloc分配内存，这个应该是最常见的了，可能一些coder只用过这种方式~。如前面所说，其本质为分配了一段虚拟地址范围(虚拟内存)，实际访问时(比如memset)通过缺页异常分配实际的物理内存。得到虚拟内存地址后，程序就可以通过这个地址来访问内存了，这个地址，在C语言中就对应指针。
2. mmap匿名分配内存。当mmap参数的fd为空时，称为匿名映射，此时的本质操作跟malloc一样，就是分配虚拟内存。事实上，malloc实现中，也使用了mmap来分配内存，细节这里不关注了。
3. mmap映射文件。当mmap中的fd参数不为空时，时间是将fd对应的设备空间映射到虚拟地址空间，这样，用户可以通过访问这段地址空间，而直接访问设备。当文件设备对应的是内存时(如共享内存、tmpfs)，用户访问的实际为内存。

# 显卡如何使用内存？

回到显卡的场景，我们这里主要讨论显卡使用主存的场景，显卡为何需要使用主存呢？

通常情况下，显卡要实现图像显示，至少需要一段内存，对应于屏幕上显示的内容，这段内存就是我们平常熟知的Framebuffer，Framebuffer可以是主存或显存上分配的任意一段内存，而且还不要求物理上连续。有些显卡自带的显存很小，甚至没有，此时必须要使用主存。

不连续都可以？为什么？

是的，如之前所说，显卡(GPU)有自己的MMU和页表，会将不连续的物理内存映射到连续的虚拟地址空间中去。所以，Framebuffer对应的内存不需要物理上连续。

Framebuffer在主存或显存上都可以？为什么？

是的，如之前所说，显卡(GPU)可以范围主存中的内存，只需要将其映射到GPU的地址空间中即可，显卡可以访问的这部分主存，也叫GTT内存，具体的映射是显卡硬件完成的，我们不关心。

简单说，如果要使用主存中的内存做Framebuffer的话，只需要在主存中分配一段内存，然后**告诉**显卡就可以了，这个“告诉”动作，本质上就是写一些指定的寄存器，对驱动的coder来说，就是驱动提供的一些接口，通常是与硬件紧密相关的。

显卡使用内存，有两种的典型场景。
1. 预先分配。即预先为Framebuffer分配好内存，然后直接使用。
2. 延迟分配。不预先分配物理内存，仅在map时分配虚拟内存，并设置page fault处理接口。当实际使用这段内存时，触发page fault，在page fault处理接口分配内存，并设置为Framebuffer。

预先分配的使用流程，简单描述如下：

1. 分配内存。使用drm提供的接口(通过ioctl方式调入内存，底层由具体的内核中的显卡驱动实现)分配内存，接口参数中提供用于选择内存分配类型的标志，用于选择分配**显存**或者**GTT内存**(也就是说选择权在用户手中)。内存分配通常依赖于内核中的TTM模块(或GEM)。
2. 设置Framebuffer。设置分配后的内存(通常以bo方式)为Framebuffer(drm也提供了相应的接口，直接调用即可，底层也由显卡驱动实现)。
3. map Framebuffer。步骤1中得到的内存是物理内存，还不能直接使用(使用虚拟内存的OS不能直接使用物理地址)，需要将其映射到虚拟地址空间后(将物理地址转换为虚拟地址/线性地址)才能使用，此步骤即完成此操作。本质为分配相应的虚拟地址，并修改相应页表，完成映射。
4. copy数据。将需要显示的数据拷贝到步骤上map后的虚拟地址中，该虚拟地址最终对应的就是Framebuffer，GPU会自动定期将Framebuffer中的内容显示到屏幕上。

延迟分配的使用流程，简单描述如下：
1. 分配gtt相关管理结构(gtt_range)。gtt结构用于管理物理内存，此时并未实际分配物理内存。
2. map内存。通常是对drm设备(/dev/dri/cardx)进行mmap操作，分配虚拟地址空间，此时也不分配物理内存，仅分配虚拟内存(vma)，并设置相应的vm_ops操作，其中包括了关键的page fault处理接口。
3. 拷贝数据，触发page fault。用户使用map后得到的虚拟地址，向其中拷贝数据，此时，由于此需要地址还没有对应的物理内存，所以，硬件会自动触发page fault，然后进入之前设置的page fault处理接口，该接口由显卡驱动实现，在其中实现物理内存的分配操作，并最终将其设置为Framebuffer。

其实过程都挺简单的，但是出于性能等因素的考虑，实际的操作可能要复杂很多，比如涉及3D加速之类的内容，这里就不详述了，后面有时间再写3D加速相关的内容。就本文而言，理解到这就可以了。

# 显卡驱动中的内存管理

前面说了基本的内存管理机制和使用方式，这节从驱动的角度，看看这些机制具体是如何实现的。

这里以Intel Atom集成显卡gma500为例说明。由于这个显卡的显存很小，只有8M左右，所以，很多情况下，显卡需要使用主存。本文关注的也是**GPU使用主存**的相关原理，对于使用显存的情况(相对简单)，不做相关分析。

老版本的gma500显卡驱动中，对于Framebuffer小于8M的情况(分辨率低于1920*1080)，直接在显存中分配内存并设置为Framebuffer。对于大于8M的情况，则使用vmalloc接口在主存中分配内存。其流程是典型的**预先分配**的场景。具体代码本文不做描述了，有兴趣的同学可以自己看看代码。

新版本的gma500显卡驱动中，对此进行改进，使用了**延迟分配**的方式，在后续章节中对相应代码进行分析。

##　分配Gtt Range

gtt_range可以理解为一段io mem，是显卡的pci配置空间中配置的一段IO mem资源，专门用来管理gtt内存的。

分配Gtt range的流程在显卡初始化的过程中，代码流程如下：

	psbfb_create
	  psbfb_alloc
	    psb_gtt_alloc_range

	/*
	 *	GTT resource allocator - allocate and manage GTT address space
	 */
	
	/**
	 *	psb_gtt_alloc_range	-	allocate GTT address space
	 *	@dev: Our DRM device
	 *	@len: length (bytes) of address space required
	 *	@name: resource name
	 *	@backed: resource should be backed by stolen pages
	 *
	 *	Ask the kernel core to find us a suitable range of addresses
	 *	to use for a GTT mapping.
	 *
	 *	Returns a gtt_range structure describing the object, or NULL on
	 *	error. On successful return the resource is both allocated and marked
	 *	as in use.
	 */
	struct gtt_range *psb_gtt_alloc_range(struct drm_device *dev, int len,
					      const char *name, int backed, u32 align)
	{
		struct drm_psb_private *dev_priv = dev->dev_private;
		struct gtt_range *gt;
		struct resource *r = dev_priv->gtt_mem;
		int ret;
		unsigned long start, end;
	
		if (backed) {
			/* The start of the GTT is the stolen pages */
			start = r->start;
			end = r->start + dev_priv->gtt.stolen_size - 1;
		} else {
			/* The rest we will use for GEM backed objects */
			start = r->start + dev_priv->gtt.stolen_size;
			end = r->end;
		}
	
		gt = kzalloc(sizeof(struct gtt_range), GFP_KERNEL);
		if (gt == NULL)
			return NULL;
		gt->resource.name = name;
		gt->stolen = backed;
		gt->in_gart = backed;
		gt->roll = 0;
		/* Ensure this is set for non GEM objects */
		gt->gem.dev = dev;
		ret = allocate_resource(dev_priv->gtt_mem, &gt->resource,
					len, start, end, align, NULL, NULL);
		if (ret == 0) {
			gt->offset = gt->resource.start - r->start;
			return gt;
		}
		kfree(gt);
		return NULL;
	}

从代码看，关键就是调用了`allocate_resource`分配了一段空闲io mem资源，这些资源本质上是BIOS中协调和预留的。

## mmap实现

用户要访问为显卡分配的内存，必须将其映射到CPU的虚拟地址空间中。通常，都是通过mmap方式。

这种情况下，显卡驱动就需要实现dri设备(/dev/dri/cardx)的mmap接口，看看dri设备的mmap接口是如何实现的：

static const struct file_operations psb_gem_fops = {
	.owner = THIS_MODULE,
	.open = drm_open,
	.release = drm_release,
	.unlocked_ioctl = psb_unlocked_ioctl,
	.mmap = drm_gem_mmap,
	.poll = drm_poll,
	.read = drm_read,
};

可以看出，其mmap接口对应为`drm_gem_mmap`，继续看看该接口流程：

	drm_gem_mmap
	  drm_gem_mmap_obj

	int drm_gem_mmap_obj(struct drm_gem_object *obj, unsigned long obj_size,
			     struct vm_area_struct *vma)
	{
		struct drm_device *dev = obj->dev;
	
		lockdep_assert_held(&dev->struct_mutex);
	
		/* Check for valid size. */
		if (obj_size < vma->vm_end - vma->vm_start)
			return -EINVAL;
	
		if (!dev->driver->gem_vm_ops)
			return -EINVAL;
	
		vma->vm_flags |= VM_IO | VM_PFNMAP | VM_DONTEXPAND | VM_DONTDUMP;
		//设置vma_ops，其中包括了page fault的处理接口
		vma->vm_ops = dev->driver->gem_vm_ops;
		vma->vm_private_data = obj;
		vma->vm_page_prot = pgprot_writecombine(vm_get_page_prot(vma->vm_flags));
	
		/* Take a ref for this mapping of the object, so that the fault
		 * handler can dereference the mmap offset's pointer to the object.
		 * This reference is cleaned up by the corresponding vm_close
		 * (which should happen whether the vma was created by this call, or
		 * by a vm_open due to mremap or partial unmap or whatever).
		 */
		drm_gem_object_reference(obj);
	
		drm_vm_open_locked(dev, vma);
		return 0;

可以看出，这个流程中设置了vma_ops，其中包括了page fault的处理接口。

接下来的流程就跟硬件紧密相关了，gma500显卡驱动的实现如下：

	static struct drm_driver driver = {
		.driver_features = DRIVER_HAVE_IRQ | DRIVER_IRQ_SHARED | \
				   DRIVER_MODESET | DRIVER_GEM,
		.load = psb_driver_load,
		.unload = psb_driver_unload,
		.lastclose = psb_driver_lastclose,
		.preclose = psb_driver_preclose,
		.set_busid = drm_pci_set_busid,
	
		.num_ioctls = ARRAY_SIZE(psb_ioctls),
		.device_is_agp = psb_driver_device_is_agp,
		.irq_preinstall = psb_irq_preinstall,
		.irq_postinstall = psb_irq_postinstall,
		.irq_uninstall = psb_irq_uninstall,
		.irq_handler = psb_irq_handler,
		.enable_vblank = psb_enable_vblank,
		.disable_vblank = psb_disable_vblank,
		.get_vblank_counter = psb_get_vblank_counter,
	
		.gem_free_object = psb_gem_free_object,
		.gem_vm_ops = &psb_gem_vm_ops,

`gem_vm_ops`设置为了`psb_gem_vm_ops`，看看对应的实现：

	static const struct vm_operations_struct psb_gem_vm_ops = {
		.fault = psb_gem_fault,
		.open = drm_gem_vm_open,
		.close = drm_gem_vm_close,
	};

可以看到，page fault对应的处理接口为：`psb_gem_fault`。

## page fault处理接口实现

继续看看`psb_gem_fault`的具体实现。
	
	int psb_gem_fault(struct vm_area_struct *vma, struct vm_fault *vmf)
	{
		struct drm_gem_object *obj;
		struct gtt_range *r;
		int ret;
		unsigned long pfn;
		pgoff_t page_offset;
		struct drm_device *dev;
		struct drm_psb_private *dev_priv;
		//获取drm_gem_object对象
		obj = vma->vm_private_data;	/* GEM object */
		dev = obj->dev;
		dev_priv = dev->dev_private;
		//获取gtt_range对象
		r = container_of(obj, struct gtt_range, gem);	/* Get the gtt range */
	
		/* Make sure we don't parallel update on a fault, nor move or remove
		   something from beneath our feet */
		mutex_lock(&dev->struct_mutex);
	
		/* For now the mmap pins the object and it stays pinned. As things
		   stand that will do us no harm */
		if (r->mmapping == 0) {
			//为gtt_range中对应的page分配物理内存。
			ret = psb_gtt_pin(r);
			if (ret < 0) {
				dev_err(dev->dev, "gma500: pin failed: %d\n", ret);
				goto fail;
			}
			r->mmapping = 1;
		}
	
		/* Page relative to the VMA start - we must calculate this ourselves
		   because vmf->pgoff is the fake GEM offset */
		page_offset = ((unsigned long) vmf->virtual_address - vma->vm_start)
					>> PAGE_SHIFT;
	
		/* CPU view of the page, don't go via the GART for CPU writes */
		// 当使用显存时，pfn直接对应stolen_base中相应的page，这些page是从显存中映射过来的，通常由BIOS完成。
		if (r->stolen)
			pfn = (dev_priv->stolen_base + r->offset) >> PAGE_SHIFT;
		else
		// 当使用GTT内存时，从gtt_range->pages中取相应page的物理地址，这些page是在psb_gtt_pin()中分配的。
			pfn = page_to_pfn(r->pages[page_offset]);
		// 将取得的物理地址对应pfn，写入vma中，同时会修改相应页表，完成物理地址到虚拟地址的映射。
		ret = vm_insert_pfn(vma, (unsigned long)vmf->virtual_address, pfn);
	
	fail:
		mutex_unlock(&dev->struct_mutex);
		switch (ret) {
		case 0:
		case -ERESTARTSYS:
		case -EINTR:
			return VM_FAULT_NOPAGE;
		case -ENOMEM:
			return VM_FAULT_OOM;
		default:
			return VM_FAULT_SIGBUS;
		}
	}

相关流程见注释，其中最关键的是`psb_gtt_pin()`，其中完成内存分配，实现如下：
	
	/**
	 *	psb_gtt_pin		-	pin pages into the GTT
	 *	@gt: range to pin
	 *
	 *	Pin a set of pages into the GTT. The pins are refcounted so that
	 *	multiple pins need multiple unpins to undo.
	 *
	 *	Non GEM backed objects treat this as a no-op as they are always GTT
	 *	backed objects.
	 */
	int psb_gtt_pin(struct gtt_range *gt)
	{
		int ret = 0;
		struct drm_device *dev = gt->gem.dev;
		struct drm_psb_private *dev_priv = dev->dev_private;
		u32 gpu_base = dev_priv->gtt.gatt_start;
	
		mutex_lock(&dev_priv->gtt_mutex);
		// Stolen==0表示使用gtt内存，而不使用显存
		if (gt->in_gart == 0 && gt->stolen == 0) {
			// 分配物理内存，分配后的page放在了gtt_range->pages中
			ret = psb_gtt_attach_pages(gt);
			if (ret < 0)
				goto out;
			ret = psb_gtt_insert(dev, gt, 0);
			if (ret < 0) {
				psb_gtt_detach_pages(gt);
				goto out;
			}
			/* 从GPU的角度，将分配的物理内存对应的物理地址写入GPU的MMU中，
			  * 即修改GPU的页表，将主存中的page(可以不连续)映射到GPU的虚拟
			  * 地址空间中。这样GPU就可以访问这些内存了，作为Framebuffer使用。
			  */
			psb_mmu_insert_pages(psb_mmu_get_default_pd(dev_priv->mmu),
					     gt->pages, (gpu_base + gt->offset),
					     gt->npage, 0, 0, PSB_MMU_CACHED_MEMORY);
		}
		gt->in_gart++;
	out:
		mutex_unlock(&dev_priv->gtt_mutex);
		return ret;
	}

这里面有两个关键动作：

1. `psb_gtt_attach_pages`完成内存分配。
2. `psb_mmu_insert_pages`完成GPU中的地址映射，使分配的内存对GPU可见。

内存分配的后续流程为：

	psb_gtt_attach_pages
	  drm_gem_get_pages
		shmem_read_mapping_page
		  shmem_getpage_gfp
			shmem_alloc_page  
			  alloc_page_vma
				alloc_page
			shmem_add_to_page_cache ///由page cache中进行管理	

至此，完成gma500显卡驱动中dri设备的mmap的实现分析，其中包括了显卡驱动中内存管理相关的关键操作，理解后，相信理解其他显卡驱动应该不会有大问题。



