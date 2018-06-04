---
layout: post
title:  "TLB flush && vmalloc"
date:   2018-04-11 17:15:28
author: JiangBiao
categories: Kernel
---

#  背景

针对pti带来的性能影响，继续分析，本文针对TLB flush与vmalloc的关系做简要分析。

# 基本概念

有关于TLB和TLB flush的相关概念在上一篇文章中已经说明，这里就不啰嗦了。

## vmalloc

vmalloc是内核中用于分配内存的常用接口，与常规的alloc_page和get_free_page不同的是，vmalloc用于分配物理不连续的大块内存，并将其映射到连续的线性地址空间中，即返回的是连续的虚拟内存，但物理上是不连续的。通常用于需要分配较多内存，而且不要求物理上连续的场景中。在驱动、容器相关代码中用的比较多。

vmalloc使用的是内核中专用的地址空间区间，在32位x86环境中，是896M-1G，64位中范围更大。有关vmalloc的具体原理，我之前好像写过文章，后续可以针对最新的内核代码再写专门的文章说明。

## vmalloc中的TLB flush

vmalloc与TLB flush有何关系，看似内核地址空间的地址映射都是提前做好的(内核初始化时)，即相应的页表都是创建好的，为何还需要TLB flush？

其实不然，应该说内核地址空间中大部分地址是在内核初始化时映射好的，但仍有小部分地址空间是按需动态映射的，典型的如vmalloc区域，另外的如fixmap区域。

如上节所述，vmalloc是用来分配连续的虚拟内存的，是在内核中动态使用的，类似于用户态的malloc接口，其调用时分配连续的虚拟内存，但并不实际分配物理内存，物理内存的分配跟malloc一样，是通过page fault分配，并建立映射、修改页表的。相关流程大家可以看看page fault的相关代码。

那么vmalloc分配的内存对应的TLB entry应该在什么时候flush呢？

如上一篇文章所述，TLB flush实际上适合在内存释放时做。对于vmalloc来说，也是一样，实际是在vfree流程中flush的。

另一方面，为提升tlbflush效率，vfree流程中，对于tlbflush操作，采用lazy模式，即：先收集，不真正释放，当达到限制(lazy_max_pages)时，再一起释放

# 相关代码

## 主要代码流程

从vfree开始，到tlbflush执行的主要代码流程如下：

	vfree
	  __vunmap
	    remove_vm_area
	      free_unmap_vmap_area
	        free_vmap_area_noflush
	          try_purge_vmap_area_lazy();
		    __purge_vmap_area_lazy
		      flush_tlb_kernel_range
		      
## more detailed

 free_vmap_area_noflush函数：
 
	/*
	 * Free a vmap area, caller ensuring that the area has been unmapped
	 * and flush_cache_vunmap had been called for the correct range
	 * previously.
	 */
	static void free_vmap_area_noflush(struct vmap_area *va)
	{
		int nr_lazy;
		// 增加lazy计数，原子操作保持并发一致性
		nr_lazy = atomic_add_return((va->va_end - va->va_start) >> PAGE_SHIFT,
					    &vmap_lazy_nr);
		// 将当前va加入到vmap_purge_list链表中，后续会根据是否达到释放条件决定是否释放
		/* After this point, we may free va at any time */
		llist_add(&va->purge_list, &vmap_purge_list);
		// 判断是否达到释放条件，如未达到，则不释放(体现出lazy)；如达到则进行释放操作
		if (unlikely(nr_lazy > lazy_max_pages()))
			try_purge_vmap_area_lazy();
	}

try_purge_vmap_area_lazy->__purge_vmap_area_lazy函数：

	/*
	 * Purges all lazily-freed vmap areas.
	 */
	static bool __purge_vmap_area_lazy(unsigned long start, unsigned long end)
	{
		struct llist_node *valist;
		struct vmap_area *va;
		struct vmap_area *n_va;
		bool do_free = false;

		lockdep_assert_held(&vmap_purge_lock);
		// 清空vmap_purge_list链表，待下次使用
		valist = llist_del_all(&vmap_purge_list);
		// 遍历链表，获取va的区间
		llist_for_each_entry(va, valist, purge_list) {
			if (va->va_start < start)
				start = va->va_start;
			if (va->va_end > end)
				end = va->va_end;
			do_free = true;
		}

		if (!do_free)
			return false;
		// flush掉相应的tlb，针对的是va对应的地址区间。
		flush_tlb_kernel_range(start, end);

		spin_lock(&vmap_area_lock);
		llist_for_each_entry_safe(va, n_va, valist, purge_list) {
			int nr = (va->va_end - va->va_start) >> PAGE_SHIFT;
			// 释放va
			__free_vmap_area(va);
			// decrease lazy数，与increase保持一致性
			atomic_sub(nr, &vmap_lazy_nr);
			cond_resched_lock(&vmap_area_lock);
		}
		spin_unlock(&vmap_area_lock);
		return true;
	}

代码比较简单，整体思路：增加lazy计数，并将要释放的va加入专用列表，当累积到一定数量时一起释放，并flush掉相应的tlb entrys，否则不释放，先存起来，待后续一起释放。

## others

在设置page属性时，也会根据情况flush掉所有lazy模式的tlb entrys，相关代码流程如下：

	change_page_attr_set
	  change_page_attr_set_clr/__set_memory_enc_dec
	    vm_unmap_aliases
	      flush_tlb_kernel_range
	        do_kernel_range_flush
	          __flush_tlb_one_kernel

其中 vm_unmap_aliases 函数的代码注释如下，大家可以理解下：
	/**
	 * vm_unmap_aliases - unmap outstanding lazy aliases in the vmap layer
	 *
	 * The vmap/vmalloc layer lazily flushes kernel virtual mappings primarily
	 * to amortize TLB flushing overheads. What this means is that any page you
	 * have now, may, in a former life, have been mapped into kernel virtual
	 * address by the vmap layer and so there might be some CPUs with TLB entries
	 * still referencing that page (additional to the regular 1:1 kernel mapping).
	 *
	 * vm_unmap_aliases flushes all such lazy mappings. After it returns, we can
	 * be sure that none of the pages we have control over will have any aliases
	 * from the vmap layer.
	 */

另外，vmalloc区域在X64环境中的起始地址定义如下，供参考：

	#define __VMALLOC_BASE_L4	0xffffc90000000000
	#define __VMALLOC_BASE_L5 	0xffa0000000000000
