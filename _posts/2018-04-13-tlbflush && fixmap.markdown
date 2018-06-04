---
layout: post
title:  "TLB flush && fixmap"
date:   2018-04-13 17:22:51
author: JiangBiao
categories: Kernel
---

#  背景

针对pti带来的性能影响，继续分析，本文针对TLB flush与fixmap的关系做简要分析。

# 基本概念

## fixmap

fixmap区域是内核地址空间中的一段特殊区间，这段区间在编译时指定，也就是说是代码中写死的(不是动态分配的)。在内核启动后，根据需要通过set_fixmap接口修改相应线性地址映射的物理地址(也就是修改相应的页表项)，从而使相应线性地址可以动态映射到新的物理地址，类似于动态分配内存的概念，与vmalloc类似，只是vmalloc的线性地址是动态分配的，而fixmap区域是编译时指定的、静态的。

从目前的代码看，主要有两个用途：

1. 用于kprobe之类的工具，作为探测点的实现方式，通过修改在内核运行过程中动态修改内核代码段中的数据，从而实现实现跳转，或类似的内核运行流的控制。
2. 在ioremap()可用之前(内核初始化的前期阶段)，通过fixmap区域提供512个临时的boot-time mappings，在early_ioremap()中使用。
。

## fixmap 与 TLB flush

由于fixmap区域也需要动态修改地址映射，所以相应的TLB entry在地址映射修改后也需要及时刷新，所以，在修改fixmap区域的地址映射后会进行相应的TLB flush操作。

# 相关代码

## fixmap区间的定义

对于x86_32环境，fixmap区域使用0xfffff000往后的区间，这段区间也是用来做fail-safe vmalloc()的，但是相关机制能保证vmalloc区间与fixmap区间不冲突。

对于x86_64环境，fixmap区域使用VSYSCALL_ADDR+PAGE_SIZE往后的区域，关于VSYSCALL后续有空可以单独写文章说明。

相关地址定义如下：

	#ifdef CONFIG_X86_32
	/* used by vmalloc.c, vsyscall.lds.S.
	 *
	 * Leave one empty page between vmalloc'ed areas and
	 * the start of the fixmap.
	 */
	extern unsigned long __FIXADDR_TOP;
	#define FIXADDR_TOP	((unsigned long)__FIXADDR_TOP)
	#else
	#define FIXADDR_TOP	(round_up(VSYSCALL_ADDR + PAGE_SIZE, 1<<PMD_SHIFT) - \
				 PAGE_SIZE)
	#endif

x86_32:

	unsigned long __FIXADDR_TOP = 0xfffff000;
	
x86_64:

	#define VSYSCALL_ADDR (-10UL << 20)

## CGroup文件系统挂载时相关流程

CGroup文件系统挂载时，会通过fixmap来修改地址映射，实现内核代码修改，相关流程如下：

	May 10 10:20:31 localhost kernel: __native_set_fixmap+0x24/0x30
	May 10 10:20:31 localhost kernel: native_set_fixmap+0x3d/0x40
	May 10 10:20:31 localhost kernel: text_poke+0xd7/0x240
	May 10 10:20:31 localhost kernel: ? balance_dirty_pages_ratelimited+0x50/0x390
	May 10 10:20:31 localhost kernel: ? balance_dirty_pages_ratelimited+0x51/0x390
	May 10 10:20:31 localhost kernel: text_poke_bp+0x7f/0xe0
	May 10 10:20:31 localhost kernel: ? balance_dirty_pages_ratelimited+0x50/0x390
	May 10 10:20:31 localhost kernel: arch_jump_label_transform+0x97/0x110
	May 10 10:20:31 localhost kernel: __jump_label_update+0x72/0xb0
	May 10 10:20:31 localhost kernel: static_key_disable_cpuslocked+0x51/0x80
	May 10 10:20:31 localhost kernel: static_key_disable+0x16/0x20
	May 10 10:20:31 localhost kernel: rebind_subsystems+0x2e0/0x3c0
	May 10 10:20:31 localhost kernel: cgroup_setup_root+0x157/0x310
	May 10 10:20:31 localhost kernel: cgroup1_mount+0x28f/0x4b0
	May 10 10:20:31 localhost kernel: cgroup_mount+0xa9/0x3b0
	May 10 10:20:31 localhost kernel: mount_fs+0x3a/0x160
	May 10 10:20:31 localhost kernel: vfs_kern_mount+0x62/0x120
	May 10 10:20:31 localhost kernel: do_mount+0x1ee/0xc50
	May 10 10:20:31 localhost kernel: ? _cond_resched+0x15/0x30
	May 10 10:20:31 localhost kernel: ksys_mount+0x7e/0xd0
	May 10 10:20:31 localhost kernel: __x64_sys_mount+0x21/0x30
	May 10 10:20:31 localhost kernel: do_syscall_64+0x5b/0x180
	May 10 10:20:31 localhost kernel: entry_SYSCALL_64_after_hwframe+0x44/0xa9
	May 10 10:20:31 localhost kernel: RIP: 0033:0x7f408ca48aca

## text_poke && tlbflush流程

上述流程中主要通过text_poke来修改运行中的内核代码段中的代码，从而实现内核代码的动态替换，可以用来实现多种功能，kprobe之类的工具即基于此实现。这里主要看看text_poke主要流程以及与tlbflush相关的主要流程。

	/**
	 * text_poke - Update instructions on a live kernel
	 * @addr: address to modify
	 * @opcode: source of the copy
	 * @len: length to copy
	 *
	 * Only atomic text poke/set should be allowed when not doing early patching.
	 * It means the size must be writable atomically and the address must be aligned
	 * in a way that permits an atomic write. It also makes sure we fit on a single
	 * page.
	 *
	 * Note: Must be called under text_mutex.
	 */
	void *text_poke(void *addr, const void *opcode, size_t len)
	{
		unsigned long flags;
		char *vaddr;
		struct page *pages[2];
		int i;
		/*
		 * 判断当前的地址是否属于core kernel代码段，如果不是，则说明该地址
		 * 是vmalloc区域的，通过vmalloc相关接口进行转换
		 */
		if (!core_kernel_text((unsigned long)addr)) {
			pages[0] = vmalloc_to_page(addr);
			pages[1] = vmalloc_to_page(addr + PAGE_SIZE);
		} else { // 如果是，这直接使用virt_to_page将地址转换为page
			pages[0] = virt_to_page(addr);
			WARN_ON(!PageReserved(pages[0]));
			pages[1] = virt_to_page(addr + PAGE_SIZE);
		}
		BUG_ON(!pages[0]);
		local_irq_save(flags);
		/*
		 * 通过set_fixmap动态修改fixmap区域中相应地址(通过索引来指定，
		 * FIX_TEXT_POKE0为其中一个索引)映射的物理地址，这里传入的物理地址为
		 * 通过入参addr传入的虚拟地址转换后的物理地址
		 */
		set_fixmap(FIX_TEXT_POKE0, page_to_phys(pages[0]));
		if (pages[1])
			set_fixmap(FIX_TEXT_POKE1, page_to_phys(pages[1]));
		// 将FIX_TEXT_POKE0索引转换为对应的虚拟地址
		vaddr = (char *)fix_to_virt(FIX_TEXT_POKE0);
		/*
		  * 将opcode(源)中的代码拷贝到FIX_TEXT_POKE0对应虚拟地址中，由于前面
		  * 已经通过set_fixmap修改了该虚拟地址对应的物理地址，所以，如此即实现了
		  * 将opcode中的代码拷贝到addr(入参)对应内存中，实现了代码的动态修改
		  */
		memcpy(&vaddr[(unsigned long)addr & ~PAGE_MASK], opcode, len);
		// 代码已经拷贝完成，需要恢复FIX_TEXT_POKE0对应的地址映射
		clear_fixmap(FIX_TEXT_POKE0);
		if (pages[1])
			clear_fixmap(FIX_TEXT_POKE1);
		/*
		 * 由于地址映射发生过改变，而且已经使用(memcpy使用，此后TLB中会有相应
		 * 地址映射的entry)，在clear掉相应的fixmap后，需要flush掉相应的tlb entry。
		 */
		local_flush_tlb();
		sync_core();
		/* Could also do a CLFLUSH here to speed up CPU recovery; but
		   that causes hangs on some VIA CPUs. */
		for (i = 0; i < len; i++)
			BUG_ON(((char *)addr)[i] != ((char *)opcode)[i]);
		local_irq_restore(flags);
		return addr;
	}
