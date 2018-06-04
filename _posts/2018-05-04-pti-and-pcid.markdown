---
layout: post
title:  "PCID & 与PTI的结合"
date:   2018-05-04 6:33:57
author: JiangBiao
categories: Kernel
---

#  背景

本文继续分析与PTI功能相关的PCID feature的原理、概念、实现以及与PTI的关联。前面分析的PTI实现代码，其实只分析了最基本的框架和代码，其中一些细节比如PCID的相关实现，没有深入，本文关注相关细节。

# 基本原理

## PCID概念

PCID，Process Context IDentifiers, 即针对每个进程分配专用的ID标识，用于区分TLB中不同进程对应的entry

看看Intel SDM中是如何描述的(4.10.1节)：

>Process-Context Identifiers (PCIDs)
>Process-context identifiers (PCIDs) are a facility by which a logical processor may cache information for multiple
>linear-address spaces. The processor may retain cached information when software switches to a different linear-
>address space with a different PCID

关键知识点：

1. PCID是一个12位的标识符，位于CR3寄存器的最低12位(想想CR3寄存器主要用来干嘛滴～)。

2. 通过CR4寄存器中的bit 17(PCIDE Flag)来控制是否enable PCID：

	当CR4.PCIDE=1时，使能PCID，此时CR3中的低12位用来存放实际的PCID
	当CR4.PCIDE=0时，禁用PCID，此时CR3中的低12为做保留用
	
3. PCID只能在IA-32e模式下才能使能，也就是说：抱歉，X86_32环境中(包括PAE模式)不支持，此时的PCID默认为000H。

4. 当使能PCID时，CPU创建的TLB entry和paging-structure caches entry，至于当前进程对应的PCID(也就是当前CR3寄存器中的低12位值)相关，与其他PCID无关；当禁用PCID时，CPU只会cache information for PCID 000H。

## PCID减少TLB flush的原理

简单说：在没有PCID的环境中，TLB是全局的(针对当前CPU)，即所有在当前CPU上运行的所有进程共享所有的TLB entry，那么当进程切换时，则必须flush掉所有的TLB(因为不同的进程的地址映射是不同的，独立的)；引入PCID后，即将所有的TLB entry通过PCID进行分类，不同PCID对应不同的进程，那么不同的进程就可以使用自己的独立的TLB entry，进程间互不影响和干扰，如此在进程切换时，就可以不flush TLB(当下一个进程使用PCID在TLB中还存在时，就是说刚切换出去不久后，再次切换回来)，或者只flush掉下一个进程的PCID对应的TLB entry(避免冲突)，如此即可在上下文切换时减少TLB flush操作，从而减少tlb miss，同时减少了TLB flush操作本身带来的性能开销。因此，达到提升性能的目的。

## PTI场景下PCID的作用

针对PTI的场景，PCID的作用仍是减少TLB flush操作。原理并无大的不同，只是引入PTI后，内核页表和用户页表分离，会增加TLB flush的场景和几率，此时通过PCID来控制和减少TLB flush就显得尤为重要。

## TLB invalidate操作

有很多操作可以invalidate TLB entry和paging-structure caches，包括：

1. INVLPG。该指令接线性地址作为参数，invalidate当前PCID中线性地址相应的page对应的TLB entry，也能invalidate相应page对应的global TLB entry。
2. INVPCID。针对PCID相关的invalidate操作，分为4个类型，后面单独描述。
3. mov to CR0。当将CR0.PG(是否启用分页机制)从1修改为0时，该操作invlidate掉所有的TLB entry(包括global entry)，和所有的paging-structures entry。
4. mov to CR3。熟悉的操作。分多种情况：
  - 如果PCID关闭的话(CR4.PCIDE = 0)，会flush掉(invalidate)所有的TLB
  - 如果PCID开启的话(CR4.PCIDE = 1)，当NOFLUSH bit(CR3最高位，63bit)设置时，不会进行flush操作; 当NOFLUSH bit(CR3最高位，63bit)未设置时，会flush掉(invalidate)当前PCID(CR3的低12位)对应的TLB entry。
5. mov to CR4。用的不多，具体不说了，详见SDM。
6. Task Switch。本质为mov to CR3。
7. VMX transitions. 也不说了，参见SDM 4.11.1.

另外，也是非常重要的一点：缺页异常也会自动(硬件)invalate 相关TLB和paging-structure caches。

## INVPCID

INVPCID操作可以分为4类：

1. Individual-address。单个地址，对应类型为0，即flush单个线性地址对应的TLB entry。不包括global translations。
2. Single-context。单个地址空间，对应类型为1, 即flush掉指定PCID对应的所有的TLB entry。不包括global translations。
3. All-context, including globals。所有条目，对应类型为2，即flush掉所有PCID对应的所有TLB entry，包括global translations。
4. All-context。所有条目，不包括global translations，对应类型为3。与类型2类似，只是不包括global translations。

用户可以根据需要，通过传入不同的类型，控制对TLB的flush操作。比如当修改某个用户态地址映射时，只需要使用Individual-address flush掉单个地址即可，如此可以最大程度的节省成本，提升性能。

当然，要使用这些类型，需要有CPU硬件的支持，需要CPU支持相应的特性才行，需要有如下特性：

1. pcid
2. invpcid
3. invpcid_single

通过如下命令可以确认当前CPU支持的特性：

	# cat /proc/cpuinfo
	
示例如下：

	#cat /proc/cpuinfo | grep pcid
	pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 movbe popcnt tsc_deadline_timer xsave avx f16c rdrand lahf_lm abm epb tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid xsaveopt dtherm ida arat pln pts

## Global Pages

Intel-64和IA-32 architectures允许使用global pages，通过CR4.PGE(bit 7)控制。当允许使用global page时，如果PTE中的G flag(bit 8)设置，则相应的TLB entry被视作为global(但不影响paging-structure caches)。CPU可以使用全局TLB entry，即使不关联当前的PCID。

global TLB entry在没有PCID时，进行mov to CR3操作(也就是进程切换时)，不会被flush掉。在引入PTI特性之前，所有的内核地址空间对应的映射都被设置为global，如果可以避免在进程切换时flush掉内核态地址空间对应的TLB entry，可以很大程度提升性功能。

## 缓存查找顺序

当需要进行线性地址翻译时(访存操作)，缓存查找顺序为：

	TLB->page-structure caches

# more about PCID

## PCID的八卦

PCID的概念源自于RISC架构CPU中的类似概念，比如asid之类的，龙芯(MIPS)中也有。在X86的硬件上，如果没记错的话，可能早在10年前就在硬件上引入了并支持了，但是一直没有真正使用起来(至少在Linux中)，为啥？

在Linux中，有人曾经提过支持PCID的补丁，但经过实际测试发现，支持后性能反而下降了？看似美好的概念，没有带来实际好处，所以一直没有得到利用。

后来，就是在PTI引入的同一时期，内核开放者才对PCID的使用方法进行了改良，不使用在RISC架构上用的方法，而是缩小了使用范围，局部使用，实验证明，对性能有一定的提升，所以，目前内核中才有了PCID的支持。但这里面，理论上应该还有不少可以发掘的空间，硬件特性摆在哪儿，使用方法可以有千万种，可能探索出更好(没有最好)的使用方式。有兴趣可以投入试试。

## PCID会降低性能？为啥？

再仔细想想，为啥PCID使用后反而会降低性能呢？

关键的几点：（大家理解）

1. X86中的PCID是12位，也就是总共最多只能有4096个pcid，而pcid是用来标示进程的，也就是说，每个进程都有自己对应的唯一的PCID，但是PCID总共就4096个，当进程数量超过4096个以后，该怎么办呢？只能将所有的TLB都flush掉，然后重新分配所有的PCID，这个过程是比较费劲的。对于现代CPU来说，核数越来越多，有些服务器环境的CPU核甚至超过了4096，这种情况下，PCID会处于严重的不够用状态，导致TLB被反复flush，这种情况下，性能下降尤为明显。
2. 将所有的TLB entry按PCID进行划分，假设每CPU上的TLB entry个数位4096个，如果将4096个PCID用完，那么平均每个PCID只能分到1个entry，这个显然有问题，几乎相当于没有TLB，这种情况下，性能下降是可以想象的。即使PCID不用完，那么每个PCID能用到的TLB entry数量也会减少很多，举个例子：在没有PCID之前，每个进程都是独享所有的TLB entry的，那么，显然此时TLB充足，对于当前进程来说，能最大程度的利用TLB，因此能提升性能。但当进程数量比较多，进程切换比较频繁时，由于每次切换都需要flush掉所有TLB，切换成本比较高；在引入PCID之后，每个进程享用的TLB数量变少，所以可能会出现不够用的情况，导致性能降低，但由于上下文切换时可以减少一部分TLB flush，所以切换成本会降低。因此，很显然，这是一个辩证过程，PCID没有绝对的优势和劣势，取决于具体的应用场景，以及PCID的分配和使用策励，也就是说，如果使用得当，性能会有更好的发挥，反之，则会让性能更糟糕。
3. 引入PCID后，并不意味着在进程切换时就不进行TLB flush操作了。需要理解本质，其实在引入PCID后，TLB显然还是需要flush的，只是每次flush的数量、以及flush的时机。引入PCID后，在没有其他优化机制的情况下，每次进程切换时，只flush掉当前PCID对应的TLB entry，也就是减少了每次flush的数量，如此能减少flush的成本，但同时由于可用的TLB entry变少了，也可能会增加TLB miss的概率。另一方面，即使在一些条件下可以不进行flush操作，能节省更多的成本，但是当TLB entry用完后，总是需要flush的(可能由硬件自己完成)，所以flush是不可避免的，需要把握 **度**。

## 合理使用PCID

如上节描述，PCID需要合理使用，才能发挥更佳的性能，那么如何才算合理，Linux需要面临的业务场景实在太多，要设计出满足所有的场景的 **合理设计** 其实非常困难。当前内核中针对PCID使用方式的优化，主要两个，还算是比较通用。

1. 使用少量的PCID。当前内核中使用的PCID数量为6个(当开启PTI时，为12个)，并没有完整的使用4096个，6个在当前各种应用场景中，能较好的体现PCID的优势。（但是否还能优化？大家可以尝试、思考下）
2. 记录最近使用的PCID。当前内核通过记录最近使用的PCID(其实也就是这6个使用的PCID)，来控制是否在进程切换时进行当前PCID对应的TLB flush操作。如果是最近使用过的PCID，那么就可以直接利用之前TLB中的缓存，避免flush，如此可以提升性能。

# 代码分析

## INVPCID实现

先看看前面所说的INVPCID的4中类型对应的实现：

	// 类型对应的宏
	#define INVPCID_TYPE_INDIV_ADDR		0
	#define INVPCID_TYPE_SINGLE_CTXT	1
	#define INVPCID_TYPE_ALL_INCL_GLOBAL	2
	#define INVPCID_TYPE_ALL_NON_GLOBAL	3
	// 类型0，单个地址
	/* Flush all mappings for a given pcid and addr, not including globals. */
	static inline void invpcid_flush_one(unsigned long pcid,
					     unsigned long addr)
	{
		__invpcid(pcid, addr, INVPCID_TYPE_INDIV_ADDR);
	}
	// 类型1, 单个地址空间
	/* Flush all mappings for a given PCID, not including globals. */
	static inline void invpcid_flush_single_context(unsigned long pcid)
	{
		__invpcid(pcid, 0, INVPCID_TYPE_SINGLE_CTXT);
	}
	// 类型2
	/* Flush all mappings, including globals, for all PCIDs. */
	static inline void invpcid_flush_all(void)
	{
		__invpcid(0, 0, INVPCID_TYPE_ALL_INCL_GLOBAL);
	}
	// 类型3
	/* Flush all mappings for all PCIDs except globals. */
	static inline void invpcid_flush_all_nonglobals(void)
	{
		__invpcid(0, 0, INVPCID_TYPE_ALL_NON_GLOBAL);
	}

继续看看__invpcid():

	static inline void __invpcid(unsigned long pcid, unsigned long addr,
				     unsigned long type)
	{
		struct { u64 d[2]; } desc = { { pcid, addr } };

		/*
		 * The memory clobber is because the whole point is to invalidate
		 * stale TLB entries and, especially if we're flushing global
		 * mappings, we don't want the compiler to reorder any subsequent
		 * memory accesses before the TLB flush.
		 *
		 * The hex opcode is invpcid (%ecx), %eax in 32-bit mode and
		 * invpcid (%rcx), %rax in long mode.
		 */
		asm volatile (".byte 0x66, 0x0f, 0x38, 0x82, 0x01"
			      : : "m" (desc), "a" (type), "c" (&desc) : "memory");
	}

其实是调用invpcid指令，并传入了不同的类型。

## PCID相关的初始化

相关操作在setup_pcid()中完成，其调用链为：

	start_kernel
	  setup_arch
	    init_mem_mapping
	      setup_pcid

setup_pcid():

	static void setup_pcid(void)
	{	// 只有64位才支持
		if (!IS_ENABLED(CONFIG_X86_64))
			return;
		// CPU是否有PCID特性
		if (!boot_cpu_has(X86_FEATURE_PCID))
			return;
		// 支持PGE，即Global page
		if (boot_cpu_has(X86_FEATURE_PGE)) {
			/*
			 * This can't be cr4_set_bits_and_update_boot() -- the
			 * trampoline code can't handle CR4.PCIDE and it wouldn't
			 * do any good anyway.  Despite the name,
			 * cr4_set_bits_and_update_boot() doesn't actually cause
			 * the bits in question to remain set all the way through
			 * the secondary boot asm.
			 *
			 * Instead, we brute-force it and set CR4.PCIDE manually in
			 * start_secondary().
			 */
			cr4_set_bits(X86_CR4_PCIDE);

			/*
			 * INVPCID's single-context modes (2/3) only work if we set
			 * X86_CR4_PCIDE, *and* we INVPCID support.  It's unusable
			 * on systems that have X86_CR4_PCIDE clear, or that have
			 * no INVPCID support at all.
			 */
			if (boot_cpu_has(X86_FEATURE_INVPCID))
				setup_force_cpu_cap(X86_FEATURE_INVPCID_SINGLE);
		} else {
			/*
			 * flush_tlb_all(), as currently implemented, won't work if
			 * PCID is on but PGE is not.  Since that combination
			 * doesn't exist on real hardware, there's no reason to try
			 * to fully support it, but it's polite to avoid corrupting
			 * data if we're on an improperly configured VM.
			 */
			setup_clear_cpu_cap(X86_FEATURE_PCID);
		}
	}

相关注释写的比较清楚，就不赘述了。

## 用户态/内核态切换时PCID的使用

之前关于PTI的文章中说明了PCID在PTI开启时有非常关键的左右，主要用来减少TLB flush，提升性能。其中一个关键点就是在用户态/内核态切换时，相关代码分析之前做过了，想见前面的文章：PTI(page table isolation)--性能分析

## 进程切换时PCID的使用

PCID在进程切换时，也起到了非常重要的优化作用，部分代码在前面的文章：PTI(page table isolation)--性能分析 中已经做过一些分析，本节看看相关关键代码，关键在于PCID的分配和使用：

### 相关宏

	//CR3中的低12位用来表示PCID，总共4096个
	/* There are 12 bits of space for ASIDS in CR3 */
	#define CR3_HW_ASID_BITS		12

	/*
	 * When enabled, PAGE_TABLE_ISOLATION consumes a single bit for
	 * user/kernel switches
	 */
	 //当使用PTI时，PCID空间被分为两部分：user和kernel，通过PCID中的最高位控制，所以可以理解为PTI消耗了1 bit。
	#ifdef CONFIG_PAGE_TABLE_ISOLATION
	# define PTI_CONSUMED_PCID_BITS	1
	#else
	# define PTI_CONSUMED_PCID_BITS	0
	#endif

	#define CR3_AVAIL_PCID_BITS (X86_CR3_PCID_BITS - PTI_CONSUMED_PCID_BITS)

	/*
	 * ASIDs are zero-based: 0->MAX_AVAIL_ASID are valid.  -1 below to account
	 * for them being zero-based.  Another -1 is because PCID 0 is reserved for
	 * use by non-PCID-aware users.
	 */
	 // PCID最大空间，本来是4096个，但是0是保留用来当关闭PCID时使用的，所以最多只能使用4095个。
	#define MAX_ASID_AVAILABLE ((1 << CR3_AVAIL_PCID_BITS) - 2)

	/*
	 * 6 because 6 should be plenty and struct tlb_state will fit in two cache
	 * lines.
	 */
	// 内核中实际使用了6个pcid（开启PTI时，实际为12个），是在进程切换时动态分配的。
	#define TLB_NR_DYN_ASIDS	6

相关的关键知识点：

1. PCID类似于传统的RISC架构中的ASID的概念
2. 但内核中没有使用传统ASID的实现：即每个进程都有自己的ASID，当AISD用尽时重新分配
3. 内核中只用了少量的(6/12个)per-cpu的asid，并且缓存了当前CPU上最近使用的mm，如此能尽量减少mm切换(进程/上下文切换)时的成本。
4. 内核中使用ASID，uPCID，kPCID等概念，需要进行区分：
  - ASID  - [0, TLB_NR_DYN_ASIDS-1]，即0-5, 对应于传统的asid标识。
  - kPCID - [1, TLB_NR_DYN_ASIDS]，即1-6,内核态pcid，为实际写入CR3中的id，实际为asid+1，因为pcid=0有特殊用途。
  - uPCID - [2048 + 1, 2048 + TLB_NR_DYN_ASIDS]，即2049-2054，用户态pcid，实际为kPCID+2048，其实就是将PCID中的最高位(也就是CR3中的bit 11)置为1即可。
 
 当启用PTI时，每个进程(mm)都有两个地址空间(内核态和用户态，使用独立的页表)，因此需要两个PCID与之对应，用户态和内核态都需要自己的PCID，所以，此时PCID数量需要翻倍，而uPCID和kPCID直接转换，只需要通过设置/清理相应bit即可。如下是相关接口实现：
 
	 /*
	 * Given @asid, compute kPCID
	 */
	static inline u16 kern_pcid(u16 asid)
	{
		VM_WARN_ON_ONCE(asid > MAX_ASID_AVAILABLE);

	#ifdef CONFIG_PAGE_TABLE_ISOLATION
		/*
		 * Make sure that the dynamic ASID space does not confict with the
		 * bit we are using to switch between user and kernel ASIDs.
		 */
		BUILD_BUG_ON(TLB_NR_DYN_ASIDS >= (1 << X86_CR3_PTI_PCID_USER_BIT));

		/*
		 * The ASID being passed in here should have respected the
		 * MAX_ASID_AVAILABLE and thus never have the switch bit set.
		 */
		VM_WARN_ON_ONCE(asid & (1 << X86_CR3_PTI_PCID_USER_BIT));
	#endif
		/*
		 * The dynamically-assigned ASIDs that get passed in are small
		 * (<TLB_NR_DYN_ASIDS).  They never have the high switch bit set,
		 * so do not bother to clear it.
		 *
		 * If PCID is on, ASID-aware code paths put the ASID+1 into the
		 * PCID bits.  This serves two purposes.  It prevents a nasty
		 * situation in which PCID-unaware code saves CR3, loads some other
		 * value (with PCID == 0), and then restores CR3, thus corrupting
		 * the TLB for ASID 0 if the saved ASID was nonzero.  It also means
		 * that any bugs involving loading a PCID-enabled CR3 with
		 * CR4.PCIDE off will trigger deterministically.
		 */
		return asid + 1;
	}
 
 其实就是：kPCID=asid+1
 
	 /*
	 * Given @asid, compute uPCID
	 */
	static inline u16 user_pcid(u16 asid)
	{
		u16 ret = kern_pcid(asid);
	#ifdef CONFIG_PAGE_TABLE_ISOLATION
		ret |= 1 << X86_CR3_PTI_PCID_USER_BIT;
	#endif
		return ret;
	}
其实就是，set了X86_CR3_PTI_PCID_USER_BIT
	
	#ifdef CONFIG_PAGE_TABLE_ISOLATION
	# define X86_CR3_PTI_PCID_USER_BIT	11
	#endif
	
X86_CR3_PTI_PCID_USER_BIT为CR3中的bit 11，也就是PCID的最高位(共12位)。

### 分别new PCID

相关逻辑在choose_new_asid()中，其调用链为：

	switch_mm
	  switch_mm_irqs_off
	    choose_new_asid
	    
即在进程切换时分配，看看具体代码：

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
		// 当修改内核地址的映射时，需要flush掉所有其他PCID域中对应的entry
		if (this_cpu_read(cpu_tlbstate.invalidate_other))
			clear_asid_other();
		/*
		 * 分配新的asid，目前使用6个(PTI情况下12个)pcid，便利per-cpu中的asid的
		 * 分配状态，在0-TLB_NR_DYN_ASIDS-1范围内， 找到一个还没有使用asid
		 * 如果还有没有使用的，则直接使用。
		 */
		for (asid = 0; asid < TLB_NR_DYN_ASIDS; asid++) {
			if (this_cpu_read(cpu_tlbstate.ctxs[asid].ctx_id) !=
			    next->context.ctx_id)
				continue;

			*new_asid = asid;
			// tlb_gen用来标记相应asid对应的TLB entry的年龄，以此来判断是否需要flush
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

上下文切换时，会根据pcid的分配和使用情况设置是否需要flush。如果需要flush，则会设置相应的per-cpu标记，待下次切换到用户态时进行实际的flush操作(lazy模式，之前分析过)。

可见，pcid(asid)的分配和使用完全时软件自己控制的，硬件层面只提供框架和机制，具体策略由软件控制。机制与策略分离，perfect。



