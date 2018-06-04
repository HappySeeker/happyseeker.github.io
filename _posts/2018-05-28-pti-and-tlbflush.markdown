---
layout: post
title:  "PTI & TLB flush"
date:   2018-05-28 6:30:24
author: JiangBiao
categories: Kernel
---

#  背景

本文继续分析与PTI功能相关TLB flush相关操作的基本原理、代码实现。没有完整覆盖TLB相关的所有内容，仅关注PTI相关的且个人比较疑惑的部分，当笔记。

# 基本原理

关于TLB和TLB flush的原理，前面已经写了不少了，这里就不重复了。

# 代码分析

## __flush_tlb_one_kernel

	/*
	 * flush one page in the kernel mapping
	 */
	 // flush内核地址空间内针对指定addr的TLB entry
	static inline void __flush_tlb_one_kernel(unsigned long addr)
	{
		count_vm_tlb_event(NR_TLB_LOCAL_FLUSH_ONE);

		/*
		 * If PTI is off, then __flush_tlb_one_user() is just INVLPG or its
		 * paravirt equivalent.  Even with PCID, this is sufficient: we only
		 * use PCID if we also use global PTEs for the kernel mapping, and
		 * INVLPG flushes global translations across all address spaces.
		 *
		 * If PTI is on, then the kernel is mapped with non-global PTEs, and
		 * __flush_tlb_one_user() will flush the given address for the current
		 * kernel address space and for its usermode counterpart, but it does
		 * not flush it for other address spaces.
		 */
		/* 
		  * flush user地址空间的某addr对应的TLB entry，这里的命名尽管已经经过
		  * 调整，但还是比较乱～～
		  * 当pti off时，只是简单的INVLPG
		  */
		__flush_tlb_one_user(addr);
		//当没有PTI时，就只是INVLPG
		if (!static_cpu_has(X86_FEATURE_PTI))
			return;

		/*
		 * See above.  We need to propagate the flush to all other address
		 * spaces.  In principle, we only need to propagate it to kernelmode
		 * address spaces, but the extra bookkeeping we would need is not
		 * worth it.
		 */
		/* 
		 * 由PTI时，需要将相应的改动(地址映射的改动)传播到其他的PCID中，因为不同的
		 * 进程使用不同的PCID，而内核地址空间部分的映射是所有的进程共享的，对于
		 * 所有进程都是相同的，所以，一旦有一个进程中对内核部分的地址映射发生了改变，
		 * 就需要将这种改变传播到其他所有进程的内核态的地址空间中
		 * 这是一个成本比较高的操作，虽然使用的场景比较少(比如vmalloc时)，但还是有
		 * 优化的空间。
		 */		 
		invalidate_other_asid();
	}
	
 __flush_tlb_one_kernel主要用来flush掉内核地址空间中指定线性地址对应映射的TLB entry。这里会涉及两个部分的工作：
 
 1. 调用__flush_tlb_one_user来flush掉用户地址空间的部分，其实，这里是有些矛盾的，个人觉得还是有点问题，有优化的空间。大家可以仔细理解下。
 2. flush掉其他所有asid中相应线性地址映射对应的TLB entry。这里，理论上实际只需要flush掉其他所有进程地址空间中的内核部分。回忆一下，引入PTI后，每个进程都有两个独立的地址空间：内核态和用户态，分别使用不同的页表，对应不同的PCID(asid)，而每个进程地址空间中的内核部分的内容，应该是相同的，所以，一旦内核地址映射有改变，就需要传播到其他进程中，但理论上，只需要传播到内核部分，也就是说，这里可以节省一半的asid对应的TLB flush操作。看似还是比较可观的，但实际测试发现，这里对系统的整体性能(包括系统调用)影响很小，原因是这个流程很少走到。
 
 我在内核中加了打印，在我的环境中，仅在内核启动过程中该流程有大量的调用(1k数量级)，在执行unixbench syscall测试时，一直没有走到相关流程。
 
 大家想想，什么情况下会走到这里，只有内核地址映射发生改变时，而对于内核来说，绝大部分地址映射都是线性映射，在内核初始化时就已经提前映射好，所以绝大部分地址映射是不会改变的，只有动态映射的区域(如vmalloc和fixmap)才可能会走到相应流程。补丁作者认为没有必要为了这点性能而做额外的bookkeeping操作，but who knows，也许你可以试试。
 
## __flush_tlb_one_user

	#define __flush_tlb_one_user(addr) __native_flush_tlb_one_user(addr)

__native_flush_tlb_one_user():

	/*
	 * flush one page in the user mapping
	 */
	static inline void __native_flush_tlb_one_user(unsigned long addr)
	{
		u32 loaded_mm_asid = this_cpu_read(cpu_tlbstate.loaded_mm_asid);
		/*
		  * 对于当前的PCID，直接调用invlpg指令，显然，当前的PCID是对应当前进程的
		  * 内核PCID，因为只有进入内核态(此时即会进行pcid切换，切换至kPCID)才能走
		  * 到这个流程。
		  */
		asm volatile("invlpg (%0)" ::"r" (addr) : "memory");
		// 如果没有PTI，则只是调用了invlpg
		if (!static_cpu_has(X86_FEATURE_PTI))
			return;

		/*
		 * Some platforms #GP if we call invpcid(type=1/2) before CR4.PCIDE=1.
		 * Just use invalidate_user_asid() in case we are called early.
		 */
		/*
		 * 如果CPU不支持invpcid_single，则通过invalidate_user_asid flush掉当前asid
		 * 对应的所有TLB entry，这样成本显然更高一些，但也没有办法
		 */
		if (!this_cpu_has(X86_FEATURE_INVPCID_SINGLE))
			invalidate_user_asid(loaded_mm_asid);
		else //如果支持invpcid_single，则通过类型为0的INVPCID指令完成单个add的flush
			invpcid_flush_one(user_pcid(loaded_mm_asid), addr);
	}
	
__flush_tlb_one_user用于flush用户态线性地址映射对应的TLB entry，通常在用户态地址映射发生修改时调用(比如free()，使用的场景较多)

__flush_tlb_one_kernel和__flush_tlb_one_user函数名称已经经过重命名了，原来的名称叫flush_tlb_one和flush_tlb_single，后来有内核开发者认为原命名比较绕，没有表达清楚实际用途，所以对其进行了重命名，但个人觉得新命名也不太好，反而原命名更准确一些(尽管当前内核代码中，__flush_tlb_one_kernel确实仅针对内核地址映射调用，而__flush_tlb_one_user仅针对用户地址调用)。

__flush_tlb_one_user是多个TLB flush函数的底层实现，用途非常广，需要仔细对待。

## invalidate_other_asid
 
 本质为设置invalidate_other变量：
 
	 /*
	 * Mark all other ASIDs as invalid, preserves the current.
	 */
	 // 设置invalidate_other，在choose_new_asid时，会根据该变量的设置情况，进行flush操作
	static inline void invalidate_other_asid(void)
	{
		this_cpu_write(cpu_tlbstate.invalidate_other, true);
	}
	
choose_new_asid()中相关流程：

	static void choose_new_asid(struct mm_struct *next, u64 next_tlb_gen,
				    u16 *new_asid, bool *need_flush)
	{
	...
		// 如果设置了invalidate_other，则clear掉除当前asid之外的所有asid
		if (this_cpu_read(cpu_tlbstate.invalidate_other))
			clear_asid_other();

		for (asid = 0; asid < TLB_NR_DYN_ASIDS; asid++) {
			if (this_cpu_read(cpu_tlbstate.ctxs[asid].ctx_id) !=
			    next->context.ctx_id)
				continue;

			*new_asid = asid;
			*need_flush = (this_cpu_read(cpu_tlbstate.ctxs[asid].tlb_gen) <
				       next_tlb_gen);
			return;
		}
		/* 
		 * 如果前面执行了clear操作，则这里的循环必然会执行失败，必然需要重新
		 * 分别asid，必然会导致need_flush被设置，也即导致TLB flush操作(在切换
		 * 到用户态时执行)
		 */
		/*
		 * We don't currently own an ASID slot on this CPU.
		 * Allocate a slot.
		 */
		*new_asid = this_cpu_add_return(cpu_tlbstate.next_asid, 1) - 1;
		if (*new_asid >= TLB_NR_DYN_ASIDS) {
			*new_asid = 0;
			this_cpu_write(cpu_tlbstate.next_asid, 1);
		}
		*need_flush = true;
	}

clear_asid_other():

	/*
	 * We get here when we do something requiring a TLB invalidation
	 * but could not go invalidate all of the contexts.  We do the
	 * necessary invalidation by clearing out the 'ctx_id' which
	 * forces a TLB flush when the context is loaded.
	 */
	void clear_asid_other(void)
	{
		u16 asid;

		/*
		 * This is only expected to be set if we have disabled
		 * kernel _PAGE_GLOBAL pages.
		 */
		if (!static_cpu_has(X86_FEATURE_PTI)) {
			WARN_ON_ONCE(1);
			return;
		}

		for (asid = 0; asid < TLB_NR_DYN_ASIDS; asid++) {
			/* Do not need to flush the current asid */
			if (asid == this_cpu_read(cpu_tlbstate.loaded_mm_asid))
				continue;
			/*
			 * Make sure the next time we go to switch to
			 * this asid, we do a flush:
			 */
			this_cpu_write(cpu_tlbstate.ctxs[asid].ctx_id, 0);
		}
		this_cpu_write(cpu_tlbstate.invalidate_other, false);
	}

很简单，就是把除当前asid之外的所有asid对应的ctx_id置为0。

如之前说明，这个流程尚有优化空间，大家可以思考。

## invalidate_user_asid

	/*
	 * Given an ASID, flush the corresponding user ASID.  We can delay this
	 * until the next time we switch to it.
	 *
	 * See SWITCH_TO_USER_CR3.
	 */
	static inline void invalidate_user_asid(u16 asid)
	{
		/* There is no user ASID if address space separation is off */
		if (!IS_ENABLED(CONFIG_PAGE_TABLE_ISOLATION))
			return;

		/*
		 * We only have a single ASID if PCID is off and the CR3
		 * write will have flushed it.
		 */
		if (!cpu_feature_enabled(X86_FEATURE_PCID))
			return;

		if (!static_cpu_has(X86_FEATURE_PTI))
			return;

		__set_bit(kern_pcid(asid),
			  (unsigned long *)this_cpu_ptr(&cpu_tlbstate.user_pcid_flush_mask));
	}

用户flush指定uPCID对应的所有TLB entry，本质上只是设置了user_pcid_flush_mask，在从内核态返回用户态时的汇编代码中会检查这个变量，如果设置，则会通过clear CR3中的NOFLUSH bit，然后mov to CR3操作，来实现相应PCID对应的TLB的flush操作。相关代码在另一篇文章中已经分析了，这里不赘述了。

这部分代码是否有优化空间？大家也可以想想。

