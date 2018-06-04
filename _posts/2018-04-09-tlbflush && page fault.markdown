---
layout: post
title:  "TLB flush && page fault"
date:   2018-04-09 9:10:36
author: JiangBiao
categories: Kernel
---

#  背景

最近在移植 Meltdown&&Spectre 漏洞补丁时，遇到了各种各样的问题，其中性能问题显得尤为突出，无论是pti，还是IBRS/IBPB，都会带来较大的性能损耗。IBRS带来的性能损耗有后续的retpoline的解决方案，能基本解决，而IBPB自身对性能影响较小，基本可以忽略。但pti带来的性能影响却无法忽略，影响比预期大很多，最近断断续续抽空做了些测试和分析，引入的性能损耗和优化思路主要跟如下几个方面相关：

1. TLB
2. PCID
3. CR3 Write

关于pti引入的性能损耗，后续再抽空单独写文章来做针对性分析。这里先针对其中涉及的局部技术细节做相关分析，可能会有一系列的短文，合在一起应该就能拼出比较完整的视图。本文主要关注TLB flush和page fault的关系以及相关流程。

# 基本概念
## TLB

TLB的字面意思就不说了，不容易理解。关于TLB的几个基本点，理解了就对了：

1. 本质上是一种缓存，是硬件上的物理单元。不同CPU上数量(大小)不同，典型的如1024
2. 用于缓存 **线性地址-->物理地址的映射关系**。我们知道，在CPU保护模式下，CPU访问内存使用的是线性地址(虚拟地址)，**页表**的用途就是将线性地址翻译成物理地址，那么TLB跟页表是啥关系？可以将TLB理解为页表的缓存。因为页表是存放在内存(主存)中的，而且页表的访问非常频繁(只要访问内存都需要)，访问页表的成本还是比较高的，所以，在硬件上加上一个缓存(就是TLB)，当需要进行地址映射时，先访问TLB，TLB miss的情况下再访问页表，由于TLB访问的速度要比内存访问快很多，如此就能极大的提升地址映射性能。跟CPU Cache的思想差不多。

## TLB Flush

如前面所说，TLB的数量是有限的，比如1024个，也就是最多只能缓存1024个(以page为单位)地址映射，那么当TLB用完了时该怎么办呢？就需要TLB flush了。

另一方面，当进程切换时，需要进行页表切换(通过CR3寄存器)，页表切换后，原有的TLB中缓存的映射关系该怎么办呢？当然需要flush掉，否则，就会出大问题了，因为原有的TLB缓存的映射关系是上一个进程地址空间中的，跟下一个进程很可能完全不同(或者根本就没有建立映射)

上面即是TLB flush的两种典型场景，当然还有其他一些场景，比如释放内存的时候。本文关注TLB flush和page fault之间的关系。

# TLB flush && page fault

TLB flush与page fault有关系么？事情看起来是这样的：

前面说了，TLB用来缓存地址映射，而Page fault主要就是用来分配内存并建立这种映射关系的，理论上这种映射关系建立以后，肯定会有一个时机点，将这种映射关系缓存到TLB中。而如果TLB中已有这样的映射关系，那是不是应该先flush掉呢？看起来好像如此。

## page fault相关代码

下面来实际看看page fault中与TLB flush相关的代码和流程。

进入page fault的主要流程(4.17内核)：

	do_page_fault
	  __do_page_fault
	    handle_mm_fault
	      __handle_mm_fault
		handle_pte_fault

主要的内存分配以及建立映射的过程都在handle_pte_fault中完成。仔细看了一圈handle_pte_fault的代码，与TLB相关的代码仅有如下一段

	handle_pte_fault：
	{
		...	
		/*
		 * This is needed only for protection faults but the arch code
		 * is not yet telling us if this is a protection fault or not.
		 * This still avoids useless tlb flushes for .text page faults
		 * with threads.
		 */
		if (vmf->flags & FAULT_FLAG_WRITE)
			flush_tlb_fix_spurious_fault(vmf->vma, vmf->address);
		...
	}

看看flush_tlb_fix_spurious_fault的定义：

	#ifndef flush_tlb_fix_spurious_fault
	#define flush_tlb_fix_spurious_fault(vma, address) flush_tlb_page(vma, address)
	#endif
	
其实就是：flush_tlb_page，从名字上看就很好理解：flush指定page对应的TLB entry。在看看具体实现：

	static inline void flush_tlb_page(struct vm_area_struct *vma, unsigned long a)
	{
		// flush指定mm(对应进程)的指定range(线性地址)的tlb entrys
		flush_tlb_mm_range(vma->vm_mm, a, a + PAGE_SIZE, VM_NONE);
	}

	void flush_tlb_mm_range(struct mm_struct *mm, unsigned long start,
					unsigned long end, unsigned long vmflag)
	{
		int cpu;

		struct flush_tlb_info info __aligned(SMP_CACHE_BYTES) = {
			.mm = mm,
		};

		cpu = get_cpu();

		/* This is also a barrier that synchronizes with switch_mm(). */
		info.new_tlb_gen = inc_mm_tlb_gen(mm);

		/* Should we flush just the requested range? */
		if ((end != TLB_FLUSH_ALL) &&
		    !(vmflag & VM_HUGETLB) &&
		    ((end - start) >> PAGE_SHIFT) <= tlb_single_page_flush_ceiling) {
			info.start = start;
			info.end = end;
		} else {
			info.start = 0UL;
			info.end = TLB_FLUSH_ALL;
		}
		// 如果mm对应当前CPU上运行的进程，则直接flush掉本CPU上的对应的TLB
		if (mm == this_cpu_read(cpu_tlbstate.loaded_mm)) {
			VM_WARN_ON(irqs_disabled());
			local_irq_disable();
			flush_tlb_func_local(&info, TLB_LOCAL_MM_SHOOTDOWN);
			local_irq_enable();
		}
		// 如果在其他CPU上，则通过IPI向其他CPU发送指令，让其flush掉相应的TLB
		if (cpumask_any_but(mm_cpumask(mm), cpu) < nr_cpu_ids)
			flush_tlb_others(mm_cpumask(mm), &info);

		put_cpu();
	}

后面的代码就不发散了，关于TLB Flush的具体代码和原理，有时间再单独写文章说明。

整体来看，这个流程都是针对**protection faults**的，**protection faults**主要就是用来通过设置相应的page为保护属性(比如只读)，然后通过捕获对相关page的write操作(由于权限不够，触发page fault)来进行相关操作。这是一种特殊的page fault用途。

相应的内核调用流程示例如下：

	[  411.950508]  native_flush_tlb_one_user+0x2a/0xb0
	[  411.950991]  ? get_page_from_freelist+0xb96/0x12a0
	[  411.951496]  flush_tlb_func_common.isra.12+0x144/0x230
	[  411.952031]  flush_tlb_mm_range+0x103/0x120
	[  411.952474]  ? page_add_file_rmap+0x13/0x210
	[  411.952921]  ? alloc_set_pte+0x105/0x4b0
	[  411.953337]  change_protection_range+0x8f9/0x9b0
	[  411.953821]  mprotect_fixup+0xfb/0x270
	[  411.954220]  do_mprotect_pkey+0x20e/0x360
	[  411.954641]  __x64_sys_mprotect+0x1b/0x20
	[  411.955062]  do_syscall_64+0x5b/0x180

而其他流程中(包括最主要的do_anonymous_page留存)确实都没有TLB相关操作。为啥？

再理解一下：page fault分配物理内存并建立映射后，相应的TLB entry是谁写入的？是CPU硬件。
当页表中建立了映射后，CPU访问相应的线性地址时，会通过MMU(硬件)自动从页表中读取相应的物理地址，而此时CPU硬件会自动将相应的映射关系写入TLB中(至少对于X86来说是这样)。

那如何能保证写入TLB中时，TLB中没有现有的相关的映射呢？比如新的映射是：

	0xffff1000-->0x00001000

而TLB中可能已经存在这样的映射关系：

	0xffff1000-->0x00002000

那么，此时会出现冲突，如果两个entry同时存在的话，CPU应该以哪个为准呢，显然需要有一定的机制来避免这种冲突。

其实，X86 CPU硬件自身解决了这种冲突，不需要软件参与，参见SDM中的相关说明：

>page faults invalidate entries in the TLBs and paging-structure caches. In particular, a page-fault exception resulting from an attempt to use a linear address will invalidate any TLB entries that are for a page number corresponding to that linear address and that are associated with the current PCID. It also invalidates all entries in the paging-structure caches that would be used for that linear address and that are associated with the current PCID. These invalidations ensure that the page-fault exception will not recur (if the faulting instruction is re-executed) if it would not be caused by the contents of the paging structures in memory (and if, therefore, it resulted from cached entries that were not invalidated after the paging structures were modified in memory).

另一方面，其实只需要在**解除映射(也就是内存释放)**时flush掉相应的TLB entry，也可以避免这种冲突了(内存释放后，TLB中相应的entry也就没有存在的意义了)。

所以，在内存管理过程中，从软件的角度看，TLB flush操作主要发生在内存释放过错中，而不是分配过程中。

## 内存释放中的TLB flush

内存释放过程中的TLB flush相关代码主要流程如下：

	[  412.244354]  native_flush_tlb_one_user+0x2a/0xb0
	[  412.244357]  flush_tlb_func_common.isra.12+0x144/0x230
	[  412.247377]  flush_tlb_mm_range+0x103/0x120
	[  412.247815]  tlb_flush_mmu_tlbonly+0x87/0xe0
	[  412.248264]  arch_tlb_finish_mmu+0x3a/0x70
	[  412.248692]  tlb_finish_mmu+0x1f/0x30
	[  412.249077]  unmap_region+0xd9/0x120
	[  412.249458]  do_munmap+0x213/0x390
	[  412.249817]  vm_munmap+0x64/0xa0
	[  412.250162]  __x64_sys_munmap+0x22/0x30
	[  412.250565]  do_syscall_64+0x5b/0x180
	
对应的用户态接口是啥以及具体内核代码流程，留作大家分析。欢迎探讨～