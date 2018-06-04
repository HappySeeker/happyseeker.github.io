---
layout: post
title:  "PTI(page table isolation)--性能分析"
date:   2018-04-25 6:44:35
author: JiangBiao
categories: Kernel
---

#  背景

本文分析PTI带来的性能损耗，从原理以及实际测试数据角度进行相关分析、记录。

# PTI性能损耗原理

如之前的文章(基本原理及性能开销)说明，PTI会带来额外的性能损耗，性能损耗来自多方面，包括：

1. 在用户态和内核态间切换时(系统调用、中断、异常)，CR3操作(主要是mov to CR3)带来的性能开销。mov to CR3带来的性能开销以100 cycles位单位计。
2. 系统调用entry时，必须时用trampoline方式，这种方式依赖于更小的资源集合(比non-PTI的传统方式)，所以只需要映射更少的内核数据到用户页表中。但是缺点是每次entry时，需要同时切换堆栈，带来额外的性能开销。
3. 全局页disabled。在引入pti之前，所有的内核空间的page都设置位Global page，这样能让不同的进程共享内核地址空间对应的所有TLB entries，能在进程切换时减少TLB flush数量。全局页disabled后，必然会增加TLB miss，带来额外的性能开销。官方文档中说该开销很小，不应该超过1%，个人并不认同，通过实际测试看，实际性能开销应该比这个大不少。
4. TLB flush。当用户态/内核态切换导致页表切换时(通过mov to CR3)，会触发TLB flush，当没有PCID时，会Flush掉所有的TLB，这个操作带来的性能开销巨大，且不说TLB flush操作本身的性能开销，在所有的TLB entry都失效后，TLB misses带来的开销也是不容忽视的。
5. PCID(Process Context IDentifiers, CPU特性，较新CPU基本都有)可以用来在页表切换时减少(或避免)TLB flush操作(通过设置CR3中的NO_FLUSH位，63位)。但是，当支持PCID时，上下文切换时就必须同时flush掉用户和内核页表对应的TLB entry，也就是比没有PCID支持的情况，在上下文切换时多了一些TLB flush操作。user PCID TLB flush操作会被延迟到exit to用户态时执行，如此能减小开销。
6. 在进程创建时(fork)，需要同时额外创建并初始化用户态页表，需要在fork()中处理。
7. 在fork()时，每当通过set_pdg()来映射用户地址空间时，还需要同时更新用户空间的PGD，这样才能保证内核页表和用户页表总是映射相同的用户态地址空间。
8. INVPCID指令可以用来flush指定PCID对应的TLB entry，但是一些硬件支持PCID，但不支持INVPCID，这种情况下，只能flush当前PCID对的entry，如果需要flush一个内核地址(比如vfree)，则必须flush掉所有的PCID，而且必须在进行上下文切换时，通过mov to CR3的方式flush，如此带来的性能损耗也需要关注。

# 实测数据

针对PTI带来的性能损耗，做了专项测试，分别针对物理机(Host)、虚拟机(KVM Guest)环境下，系统的总体性能和系统调用性能做了对比测试。

测试环境：

Architecture: x86_64
CPU op-mode(s): 32-bit, 64-bit
Byte Order: Little Endian
CPU(s): 32
On-line CPU(s) list: 0-31
Thread(s) per core: 2
Core(s) per socket: 8
Socket(s): 2
NUMA node(s): 2
Vendor ID: GenuineIntel
CPU family: 6
Model: 63
Model name: Intel(R) Xeon(R) CPU E5-2640 v3 @ 2.60GHz
Stepping: 2
CPU MHz: 1373.632
CPU max MHz: 3400.0000
CPU min MHz: 1200.0000
BogoMIPS: 5199.93
Virtualization: VT-x
L1d cache: 32K
L1i cache: 32K
L2 cache: 256K
L3 cache: 20480K
NUMA node0 CPU(s): 0-7,16-23
NUMA node1 CPU(s): 8-15,24-31
Flags: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 ds_cpl vmx smx est tm2 ssse3 fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm epb invpcid_single spec_ctrl ibpb_support tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid cqm xsaveopt cqm_llc cqm_occup_llc dtherm arat pln pts
mem： 48G

测试工具：

unixbench-5.1.2

内核版本:

4.17.0-rc1

## 总体性能

使用unixbench默认参数，测试物理机(Host)上系统整体性能，测试执行命令为：

	# ./Run

测试项包括：

	Dhrystone 2 using register variables       
	Double-Precision Whetstone                   
	Execl Throughput                               
	File Copy 1024 bufsize 2000 maxblocks       
	File Copy 256 bufsize 500 maxblocks         
	File Copy 4096 bufsize 8000 maxblocks      
	Pipe Throughput                              
	Pipe-based Context Switching               
	Process Creation                             
	Shell Scripts (1 concurrent)                   
	Shell Scripts (8 concurrent)                 
	System Call Overhead      

running 1 parallel copy of tests，在打开PTI(未设置nopti内核参数)时，性能数据为：

	1150.4

在关闭PTI(设置nopti参数)时，性能数据为：
	1399.8

打开PTI时，性能损耗：
	- 17.8%

总体看，性能损耗与redhat以及其他厂商数据基本一致，但还是偏高，对于很多业务来说，可能比较难接受。

## 系统调用性能

根据PTI的原理，其主要影响内核态与用户态的切换性能，而系统调用是最典型的场景，也比较容易测试，所以这里主要关注PTI对系统调用带来的性能影响，从更细节、更深的层次观察性能损耗。

测试环境与之前的一致，只是测试方法有所不同，仅测试系统调用、单线程场景下的性能，测试命令如下：

	# ./Run syscall -c 1

参数说明：

	syscall： 表示仅测试系统调用
	-c 1 ： 表示仅运行1个copy

然后，针对Host和Guest做分别测试，需要同时确认对虚拟机和物理机的性能影响差异。

### Host系统调用性能

在Host上进行系统调用性能测试，结果为：

	开启pti：374
	关闭pti：689
	性能损耗：45.7%

可见，PTI对系统调用的影响非常大。

### Guest系统调用性能

#### Guest without PCID

在Kvm Guest(4cpu, 4G mem)上进行系统调用性能测试，结果为：

	开启pti：200
	关闭pti：679
	性能损耗：68%

性能损失一大半，这个数据可能超出了你的想象，但的确如此。如此低的性能主要原因是，Guest中CPU没有PCID feature，说明在开启PTI的情况下，如果没有PCID，性能惨不忍睹。

#### Guest with PCID

为Guest中CPU配置PCID相关feature，主要包括：

	pcid
	invpcid
	invpcid_single

配置方法有两种：

1. 直接在xml中添加相关feature，示例如：

  <cpu mode='custom' match='exact'>
    <model fallback='allow'>IvyBridge</model>
    <feature name='pcid'/>
    <feature name='invpcid'/>
  </cpu>

2. 直接将Host的feature透传给Guest(通常用这种方式即可)，示例如：

	  <cpu mode='host-model'>
	  </cpu>

测试结果：

	开启pti：354
	关闭pti：679
	性能损耗：47.8%

可见，PCID能明显提升PTI的性能，原理见上一节说明，后面相信分析相关代码。另一方面，可以看出，Guest中的性能与Host性能相差不大，PTI带来的性能影响也基本一致。

# 性能分析

这节结合PTI相关的代码来分析性能损耗的相关原理。

## CR3 manipulation

实际为mov to CR3操作，该操作应该是PTI引入的性能损耗的最主要来源，从原理上，在引入PTI之前，由于进程的用户态和内核态使用同一个页表，所以在用户态/内核态切换时，不需要切换页表，也就不需要mov to CR3操作，所以就没有相应的性能损耗；而引入PTI后，由于内核和用户态使用独立页表，所以在切换时，必须切换页表，通常也就需要通过mov to CR3操作来切换页表(是否还有其他方法？大家可以想想～)，所以带来相应损耗。

看看相关代码，主要在arch/x86/entry/calling.h文件中，其中定义了用户态与内核态切换时需要调用的一些基本接口（宏），相关代码在之前的文章中已经分析过了，这里只关注相关的关键部分：

	// 从用户态切换到内核态时需要调用的接口
	.macro SWITCH_TO_KERNEL_CR3 scratch_reg:req
		ALTERNATIVE "jmp .Lend_\@", "", X86_FEATURE_PTI
		mov	%cr3, \scratch_reg
		ADJUST_KERNEL_CR3 \scratch_reg
		// mov to CR3操作
		mov	\scratch_reg, %cr3
	.Lend_\@:
	.endm
	...
	// 从内核态切换到用户态时需要调用的接口
	.macro SWITCH_TO_USER_CR3_NOSTACK scratch_reg:req scratch_reg2:req
		ALTERNATIVE "jmp .Lend_\@", "", X86_FEATURE_PTI
		mov	%cr3, \scratch_reg

		ALTERNATIVE "jmp .Lwrcr3_\@", "", X86_FEATURE_PCID
	...
	.Lwrcr3_\@:
		/* Flip the PGD to the user version */
		orq     $(PTI_USER_PGTABLE_MASK), \scratch_reg
		// mov to CR3操作
		mov	\scratch_reg, %cr3
	.Lend_\@:
	.endm

可见，在切换时都进行了mov to CR3操作。

mov to CR3对性能到底有多大影响，内核文档中说：order of hundred cycles，不太容易直接测量，我们可以试试：

修改上述的SWITCH_TO_KERNEL_CR3宏，在末尾加入重复的mov to CR3动作，修改后的代码为：

	.macro SWITCH_TO_KERNEL_CR3 scratch_reg:req
		ALTERNATIVE "jmp .Lend_\@", "", X86_FEATURE_PTI
		mov	%cr3, \scratch_reg
		ADJUST_KERNEL_CR3 \scratch_reg
		mov	\scratch_reg, %cr3
		// 添加的重复的mov to CR3操作
		mov	\scratch_reg, %cr3
	.Lend_\@:
	.endm

使用同样的测试方法，物理机测试系统调用性能：

	结果为：311
	相比修改之前(374)的性能损耗为：-17%

如果加入两句重复的mov to CR3动作：

   	.macro SWITCH_TO_KERNEL_CR3 scratch_reg:req
		ALTERNATIVE "jmp .Lend_\@", "", X86_FEATURE_PTI
		mov	%cr3, \scratch_reg
		ADJUST_KERNEL_CR3 \scratch_reg
		mov	\scratch_reg, %cr3
		// 添加的重复的mov to CR3操作(2次)
		mov	\scratch_reg, %cr3
		mov	\scratch_reg, %cr3
	.Lend_\@:
	.endm

物理机测试系统调用性能：

	结果为：270
	相比修改添加一次mov to cr3操作(311)的性能损耗为：-13%

可见，每次mov to CR3带来的性能损耗都在10%以上，那么，再想想性能损耗从何而来，几种可能：

1. mov to CR3指令自身的执行时间
2. TLB flush
3. CPU Cache flush

对于可能2(TLB flush), 再看看ADJUST_KERNEL_CR3代码：

	.macro ADJUST_KERNEL_CR3 reg:req
		// 当CPU支持PCID时，set noflush，即不进行TLB flush操作
		ALTERNATIVE "", "SET_NOFLUSH_BIT \reg", X86_FEATURE_PCID
		/* Clear PCID and "PAGE_TABLE_ISOLATION bit", point CR3 at kernel pagetables: */
		andq    $(~PTI_USER_PGTABLE_AND_PCID_MASK), \reg
	.endm

可见，在CPU支持PCID时(这里的测试环境中是支持的)，是不进行TLB flush操作的(后面深入分析)，所以这里可以排除可能性2带来的性能影响。

对于可能性3(CPU Cache flush)，由于在执行新增的mov to cr3操作之前，刚进行了一次相同的操作，页就是说，Cache刚刚已经被刷过一遍了，再次flush应该影响不大，另外，添加第2次mov to cr3操作同样带来超过了10%，可以看出，Cache的影响应该也比较有限。

所以，说明mov to cr3指令本身带来的性能影响是性能损耗的主要原因，那mov to cr3到底需要多少cycles呢？

再看看Guest中syscall的unixbench测试结果：

	System Call Overhead                         528438.3 lps   (10.0 s, 7 samples)

	System Benchmarks Partial Index              BASELINE       RESULT    INDEX
	System Call Overhead                          15000.0     528438.3    352.3
		                                                           ========
	System Benchmarks Index Score (Partial Only)                          352.3

528438.3 lps表示，每秒完成50万+次系统调用，说明每次系统调用耗费约2us(2000ns)的时间。

粗略估算，每次mov to CR3操作带来约15%的性能损耗，那么mov to CR3操作耗费的时间约为：

	2000×15% = 300ns

我的测试环境的主频 2.60GH， 大家可以算算mov to CR3操作需要多少cycles。

## PCID
### PCID性能提升原理

上一节的性能测试数据可以看出，PCID能显著提升PTI内核的性能，这里来看看相关基本原理和代码。

PCID，Process Context IDentifiers, 即针对每个进程分配专用的ID标识，用于区分TLB中不同进程对应的entry，简单说：在没有PCID的环境中，TLB是全局的(针对当前CPU)，即所有在当前CPU上运行的所有进程共享所有的TLB entry，那么当进程切换时，则必须flush掉所有的TLB(因为不同的进程的地址映射是不同的，独立的)；引入PCID后，即将所有的TLB entry通过PCID进行分类，不同PCID对应不同的进程，那么不同的进程就可以使用自己的独立的TLB entry，进程间互不影响和干扰，如此在进程切换时，就可以不flush TLB(当下一个进程使用PCID在TLB中还存在时，就是说刚切换出去不久后，再次切换回来)，或者只flush掉下一个进程的PCID对应的TLB entry(避免冲突)，如此即可在上下文切换时减少TLB flush操作，从而减少tlb miss，同时减少了TLB flush操作本身带来的性能开销。因此，达到提升性能的目的。关于PCID的相关原理和代码，后续可以单独写文章说明。

针对PTI的场景，PCID的作用仍是减少TLB flush操作，如何体现？

### 用户态->内核态

看看相关代码：

	// 当从用户态切换到内核态时，需要调用该接口
	.macro ADJUST_KERNEL_CR3 reg:req
		// 当CPU支持PCID时，set noflush，即不进行TLB flush操作
		ALTERNATIVE "", "SET_NOFLUSH_BIT \reg", X86_FEATURE_PCID
		/* Clear PCID and "PAGE_TABLE_ISOLATION bit", point CR3 at kernel pagetables: */
		andq    $(~PTI_USER_PGTABLE_AND_PCID_MASK), \reg
	.endm

还是刚刚分析过的那段，当从用户态切换到内核态时，如果有X86_FEATURE_PCID，则会调用SET_NOFLUSH_BIT：

	// 设置NO_FLUSH bit，设置后，在mov to CR3时，将不会进行TLB flush操作
	.macro SET_NOFLUSH_BIT	reg:req
		bts	$X86_CR3_PCID_NOFLUSH_BIT, \reg
	.endm

通过设置NO_FLUSH bit，达到不flush的目的，

	#define X86_CR3_PCID_NOFLUSH_BIT 63 /* Preserve old PCID */
	#define X86_CR3_PCID_NOFLUSH    _BITULL(X86_CR3_PCID_NOFLUSH_BIT)

NO_FLUSH bit是CR3寄存器中的最后一位，第63位。

可见，从用户态到内核态切换时，是不进行TLB Flush操作的，这点与引入PTI之前一致，所以，在有PCID的情况下，PTI带来的从用户态到内核态切换的开销主要来自与mov to CR3操作。

### 内核态->用户态

看看相关代码：

	/*
	 * per cpu变量TLB_STATE_user_pcid_flush_mask用于记录PCID对应的TLB entry
	 * 是否需要在返回用户态时flush，这是一种lazy设计，用于延迟flush操作，将flush操作
	 * 延迟到返回用户态时执行，能提升效率。
	 */
	#define THIS_CPU_user_pcid_flush_mask   \
		PER_CPU_VAR(cpu_tlbstate) + TLB_STATE_user_pcid_flush_mask
	// 从内核态返回用户态时调用
	.macro SWITCH_TO_USER_CR3_NOSTACK scratch_reg:req scratch_reg2:req
		// nopti场景下，什么也不做，不切换页表
		ALTERNATIVE "jmp .Lend_\@", "", X86_FEATURE_PTI
		mov	%cr3, \scratch_reg
		// 没有PCID的场景下，直接进行mov to cr3操作
		ALTERNATIVE "jmp .Lwrcr3_\@", "", X86_FEATURE_PCID

		/*
		 * Test if the ASID needs a flush.
		 */
		/*
		 * 检查是否需要在返回用户态时执行TLB flush操作，通过判断变量
		 * THIS_CPU_user_pcid_flush_mask是否设置(在上下文切换时根据情况设置)，
		 * 然后通过NO_FULSH bit来实现。
		 */
		movq	\scratch_reg, \scratch_reg2
		andq	$(0x7FF), \scratch_reg		/* mask ASID */
		// 判断是否设置了相关变量
		bt	\scratch_reg, THIS_CPU_user_pcid_flush_mask
		jnc	.Lnoflush_\@

		/* Flush needed, clear the bit */
		// 需要flush，清空变量，然后跳过noflush相关操作
		btr	\scratch_reg, THIS_CPU_user_pcid_flush_mask
		movq	\scratch_reg2, \scratch_reg
		jmp	.Lwrcr3_pcid_\@

		//需要flush，设置NOFLUSH_BIT
	.Lnoflush_\@:
		movq	\scratch_reg2, \scratch_reg
		SET_NOFLUSH_BIT \scratch_reg
		/*
		 * 将pcid flip成用户态的版本，用户态PCID和内核态PCID相差1位，由CR3中的第12位
		 * 控制(bit 11)，由于CR3中的低12位表示PCID，那就意味着，user PCID对应于2048-4095；
		 * 而kernel PCID对应于1-2047(0由特殊含义)。用户和内核使用不同的PCID，那就意味者
		 * 内核和用户态位于不同的TLB Context，这对于抵御Meltdown攻击也很关键。
		 */
	.Lwrcr3_pcid_\@:
		/* Flip the ASID to the user version */
		orq	$(PTI_USER_PCID_MASK), \scratch_reg

	.Lwrcr3_\@:
		// mov to CR3操作
		/* Flip the PGD to the user version */
		orq     $(PTI_USER_PGTABLE_MASK), \scratch_reg
		mov	\scratch_reg, %cr3
	.Lend_\@:
	.endm

整体看，通过THIS_CPU_user_pcid_flush_mask per cpu变量来控制是否进行TLB flush操作，再看看该变量的控制逻辑，主要在invalidate_user_asid函数中：

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
		// 设置cpu_tlbstate.user_pcid_flush_mask，对应于THIS_CPU_user_pcid_flush_mask per cpu
		__set_bit(kern_pcid(asid),
			  (unsigned long *)this_cpu_ptr(&cpu_tlbstate.user_pcid_flush_mask));
	}

很简单，不解释了。相关的调用流程主要在上下文切换过程中

	switch_mm
	  switch_mm_irqs_off
	    load_new_mm_cr3
	      invalidate_user_asid

即，在上下文切换时，会根据条件设置THIS_CPU_user_pcid_flush_mask per cpu。什么条件？再看看相关逻辑：

load_new_mm_cr3：

	static void load_new_mm_cr3(pgd_t *pgdir, u16 new_asid, bool need_flush)
	{
		unsigned long new_mm_cr3;
		// 根据need_flush变量判断
		if (need_flush) {
			invalidate_user_asid(new_asid);
			// 需要flush，则构造新的cr3,其中不设置NOFLUSH bit
			new_mm_cr3 = build_cr3(pgdir, new_asid);
		} else {
			// 需要flush，则构造新的cr3,其中设置NOFLUSH bit
			new_mm_cr3 = build_cr3_noflush(pgdir, new_asid);
		}

		/*
		 * Caution: many callers of this function expect
		 * that load_cr3() is serializing and orders TLB
		 * fills with respect to the mm_cpumask writes.
		 */
		 /*
		  * mov to cr3,这里为什么需要？不是在切换回用户态时再进行的么?
		  * 上下文切换时本来就需要，在未引入PTI之前，这里就是主要的切换点
		  * 这里主要是切换不同进程之间的内核态的页表。
		  * 思考一下：在引入PTI之后，这里的mov to cr3操作是否可以去掉？想个办法，肯定能提升性能
		  */
		write_cr3(new_mm_cr3);
	}

在看看need_flush的设置逻辑：

	void switch_mm_irqs_off(struct mm_struct *prev, struct mm_struct *next,
				struct task_struct *tsk)
	{
	...
		choose_new_asid(next, next_tlb_gen, &new_asid, &need_flush);

			if (need_flush) {
				this_cpu_write(cpu_tlbstate.ctxs[new_asid].ctx_id, next->context.ctx_id);
				this_cpu_write(cpu_tlbstate.ctxs[new_asid].tlb_gen, next_tlb_gen);
				load_new_mm_cr3(next->pgd, new_asid, true);
	...
	}

	static void choose_new_asid(struct mm_struct *next, u64 next_tlb_gen,
				    u16 *new_asid, bool *need_flush)
	{
		u16 asid;
		// 没有PCID，总是需要need_flush
		if (!static_cpu_has(X86_FEATURE_PCID)) {
			*new_asid = 0;
			*need_flush = true;
			return;
		}
		// 当修改内核地址的映射时，需要flush掉所有其他PCID域中对应的entry，相关逻辑后续说明
		if (this_cpu_read(cpu_tlbstate.invalidate_other))
			clear_asid_other();
		// 分配新的asid，目前使用6个(PTI情况下12个)pcid，后续文章说明
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
		 * We don't currently own an ASID slot on this CPU.
		 * Allocate a slot.
		 */
		/*
		 * 当PCID用完时，从头分配，并设置need_flush，因为pcid需要重复利用，那么之前的pcid
		 * 对应的TLB entry都应该flush掉
		 */
		*new_asid = this_cpu_add_return(cpu_tlbstate.next_asid, 1) - 1;
		if (*new_asid >= TLB_NR_DYN_ASIDS) {
			*new_asid = 0;
			this_cpu_write(cpu_tlbstate.next_asid, 1);
		}
		*need_flush = true;
	}

可见，上下文切换时，会根据pcid的分配和使用情况设置是否需要flush。详细情况大家可以仔细理解下代码。

总体看，从内核态返回用户态时，有可能会执行TLB flush操作，这取决于上下文切换时的条件判断，根据当前逻辑，理论上当CPU上只有系统调用时操作而且只有执行系统调用的单个进程运行时(实际上不可能哈)，从内核态返回用户态也是不需要flush TLB的，这与引入PTI之前一致。但实际上，上下文切换不仅仅发生在系统调用过程中，也发生在中断和异常返回过程中，所以，中断和异常，以及多任务运行，都可能影响系统调用的性能，因为都把tlbflush操作延迟到了返回用户态时执行。所以，这部分的带来的性能影响应该比从用户态切换到内核态时更大。

## trampline

在引入PTI之后，系统调用entry时，必须使用trampoline方式，这种方式依赖于更小的资源集合(比non-PTI的传统方式)，所以只需要映射更少的内核数据到用户页表中。但是缺点是每次entry时，需要同时切换堆栈，带来额外的性能开销。

相关代码如下：

	void syscall_init(void)
	{
		extern char _entry_trampoline[];
		extern char entry_SYSCALL_64_trampoline[];

		int cpu = smp_processor_id();
		unsigned long SYSCALL64_entry_trampoline =
			(unsigned long)get_cpu_entry_area(cpu)->entry_trampoline +
			(entry_SYSCALL_64_trampoline - _entry_trampoline);

		wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
		/*启用PTI的情况下，系统调用entry使用trampline方式，否则，使用常规方式*/
		if (static_cpu_has(X86_FEATURE_PTI))
			wrmsrl(MSR_LSTAR, SYSCALL64_entry_trampoline);
		else
			wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
		...
	}

修改这段代码，使其总是使用trmpline方式：

	void syscall_init(void)
	{
		extern char _entry_trampoline[];
		extern char entry_SYSCALL_64_trampoline[];

		int cpu = smp_processor_id();
		unsigned long SYSCALL64_entry_trampoline =
			(unsigned long)get_cpu_entry_area(cpu)->entry_trampoline +
			(entry_SYSCALL_64_trampoline - _entry_trampoline);

		wrmsr(MSR_STAR, 0, (__USER32_CS << 16) | __KERNEL_CS);
		/*总是使用trampline方式*/
		wrmsrl(MSR_LSTAR, SYSCALL64_entry_trampoline);
		...
	}

然后测试，关闭PTI情况下的syscall性能，

	结果为：651
	相比修改代码前(走常规的syscall entry流程)性能(680)损耗：-4%

## 其他

关于前面提到的其他因素，这里就不一一分析了，理论上应不是主要因素，或者场景比较局限，留作大家想象和研究。
