---
layout: post
title:  "Meltdown && Spectre漏洞补丁-启动失败"
date:   2018-03-022 06:45:51
author: JiangBiao
categories: Kernel
---

#  背景

在移植上游的meltdown/spectre漏洞补丁后，发现部分硬件环境中，会有启动失败的问题，现象为：内核启动挂住，没有任何异常打印，在结上串口后，依然没有任何可用信息。
另一方面，当在启动参数中加入nopti或者nopcid后，内核能正常启动，说明问题在与PCID或者PTI相关的功能中。

# 原因

原因：漏合了如下补丁：

a035795499ca1c2bd1928808d1a156eda1420383
commit a035795499ca1c2bd1928808d1a156eda1420383
Author: Thomas Gleixner <tglx@linutronix.de>
Date:   Mon Dec 4 15:07:30 2017 +0100

    x86/paravirt: Dont patch flush_tlb_single
    
    native_flush_tlb_single() will be changed with the upcoming
    PAGE_TABLE_ISOLATION feature. This requires to have more code in
    there than INVLPG.
    
    Remove the paravirt patching for it
    
    diff --git a/arch/x86/kernel/paravirt_patch_64.c b/arch/x86/kernel/paravirt_patch_64.c
index ac0be82..9edadab 100644
--- a/arch/x86/kernel/paravirt_patch_64.c
+++ b/arch/x86/kernel/paravirt_patch_64.c
@@ -10,7 +10,6 @@ DEF_NATIVE(pv_irq_ops, save_fl, "pushfq; popq %rax");
 DEF_NATIVE(pv_mmu_ops, read_cr2, "movq %cr2, %rax");
 DEF_NATIVE(pv_mmu_ops, read_cr3, "movq %cr3, %rax");
 DEF_NATIVE(pv_mmu_ops, write_cr3, "movq %rdi, %cr3");
-DEF_NATIVE(pv_mmu_ops, flush_tlb_single, "invlpg (%rdi)");
 DEF_NATIVE(pv_cpu_ops, wbinvd, "wbinvd");
 
 DEF_NATIVE(pv_cpu_ops, usergs_sysret64, "swapgs; sysretq");
@@ -60,7 +59,6 @@ unsigned native_patch(u8 type, u16 clobbers, void *ibuf,
                PATCH_SITE(pv_mmu_ops, read_cr2);
                PATCH_SITE(pv_mmu_ops, read_cr3);
                PATCH_SITE(pv_mmu_ops, write_cr3);
-               PATCH_SITE(pv_mmu_ops, flush_tlb_single);
                PATCH_SITE(pv_cpu_ops, wbinvd);
 #if defined(CONFIG_PARAVIRT_SPINLOCKS)

## 相关原理

上面的补丁看似 **半虚拟化** 的专用补丁，而我们现有的环境几乎 **半虚拟化** 几乎已经绝迹了，所以当时合补丁时，面对这个补丁，并没有多想，直接忽略了。

看看相关的基本原理，这个补丁的作用就是取消对flush_tlb_single打半虚拟化补丁，为什么要取消呢？

因为在引入PTI之后， native_flush_tlb_single() 函数就不再再简单的INVLPG指令的调用了，而是加入了新的代码合逻辑，看看相关代码(在最新内核中对应的函数为__flush_tlb_one_kernel)：

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
		  * flush user地址空间的某addr对应的TLB entry
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
		invalidate_other_asid();
	}

	/*
	 * flush one page in the user mapping
	 */
	static inline void __native_flush_tlb_one_user(unsigned long addr)
	{
		u32 loaded_mm_asid = this_cpu_read(cpu_tlbstate.loaded_mm_asid);
		/*
		  * 对于当前的PCID，直接调用invlpg指令
		  */
		asm volatile("invlpg (%0)" ::"r" (addr) : "memory");
		// 如果没有PTI，则只是调用了invlpg
		if (!static_cpu_has(X86_FEATURE_PTI))
			return;

		/*
		 * Some platforms #GP if we call invpcid(type=1/2) before CR4.PCIDE=1.
		 * Just use invalidate_user_asid() in case we are called early.
		 */
		if (!this_cpu_has(X86_FEATURE_INVPCID_SINGLE))
			invalidate_user_asid(loaded_mm_asid);
		else 
			invpcid_flush_one(user_pcid(loaded_mm_asid), addr);
	}

再引入PTI之前， native_flush_tlb_single() 函数就是简单的INVLPG指令调用。

由于有内核配置项：

	CONFIG_PARAVIRT=y
	# CONFIG_PARAVIRT_DEBUG is not set
	CONFIG_PARAVIRT_SPINLOCKS=y

所以，在半虚拟化相关初始化流程中：

	start_kernel()
	  check_bugs
	    alternative_instructions
	      apply_paravirt
	        pv_init_ops.patch
	          native_patch

对tlb_flush_single打了补丁：

	unsigned native_patch(u8 type, u16 clobbers, void *ibuf,
		              unsigned long addr, unsigned len)
	{
		const unsigned char *start, *end;
		unsigned ret;

	#define PATCH_SITE(ops, x)                                      \
		        case PARAVIRT_PATCH(ops.x):                     \
		                start = start_##ops##_##x;              \
		                end = end_##ops##_##x;                  \
		                goto patch_site
		switch(type) {
		        PATCH_SITE(pv_irq_ops, restore_fl);
		        PATCH_SITE(pv_irq_ops, save_fl);
		        PATCH_SITE(pv_irq_ops, irq_enable);
		        PATCH_SITE(pv_irq_ops, irq_disable);
		        PATCH_SITE(pv_cpu_ops, iret);
		        PATCH_SITE(pv_cpu_ops, irq_enable_sysexit);
		        PATCH_SITE(pv_cpu_ops, usergs_sysret32);
		        PATCH_SITE(pv_cpu_ops, usergs_sysret64);
		        PATCH_SITE(pv_cpu_ops, swapgs);
		        PATCH_SITE(pv_mmu_ops, read_cr2);
		        PATCH_SITE(pv_mmu_ops, read_cr3);
		        PATCH_SITE(pv_mmu_ops, write_cr3);
		        PATCH_SITE(pv_cpu_ops, clts);
		        // 补丁位置
		        case PARAVIRT_PATCH(pv_mmu_ops.flush_tlb_single):
		               if (!boot_cpu_has(X86_FEATURE_PCID)) {
		                       start = start_pv_mmu_ops_flush_tlb_single;
		                       end   = end_pv_mmu_ops_flush_tlb_single;
		                       goto patch_site;
		               } else {
		                       goto patch_default;
		               }

		        //PATCH_SITE(pv_mmu_ops, flush_tlb_single);
		        PATCH_SITE(pv_cpu_ops, wbinvd);

		patch_site:
		        ret = paravirt_patch_insns(ibuf, len, start, end);
		        break;

	patch_default:
		default:
		        ret = paravirt_patch_default(type, clobbers, ibuf, addr, len);
		        break;

而引入PTI后，之前的patch不再适用，再按这种方式打补丁必然会导致问题，所以应该要取消。

这个补丁看似半虚拟化专用，但其实不然，打上补丁后，影响是全局的，本质上是替换了原有的flush_tlb_single操作的执行流程，而flush_tlb_single操作是内核中非常关键的、频繁的、底层的基础操作，影响并不局限于半虚拟化场景。
