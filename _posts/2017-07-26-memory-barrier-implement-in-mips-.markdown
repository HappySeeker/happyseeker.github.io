---
layout: post
title:  "内核中Mips架构内存屏障的实现"
date:   2017-07-06 09:43:32
author: JiangBiao
categories: Kernel
---

#  前言

**内存屏障**是一项与硬件架构紧密相关的关键技术，直接决定了多核场景中，并发数据访问的安全性。由于不同硬件架构的memory order实现不同，在需要保证内存访问顺序的场景中，内存屏障能发挥关键作用。

而Linux内核中，很多关键场景中都需要使用内存屏障保证可靠，内核中也实现了多种不同类型的内存屏障，本文主要分析内核中Mips架构相关内存屏障的主要实现。

关于内存屏障的原理，见其它文章说明。

# CPU架构 & 内存屏障

CPU架构与内存屏障关系紧密，通常情况下，CPU为最大程度的发挥性能，会乱序执行指令，导致在一些场景中可能存在严重隐患，此时则需要内存屏障来保证内存访问顺序。

不同的CPU架构其memory order的强弱程度不同，X86架构CPU有很强的memory order，其提供很多隐式的内存屏障(implicit memory barrier)，比如：LD_LD | LD_ST  |ST_ST，而Mips架构是典型的weak memory order的架构，完全没有隐式内存屏障。

本文主要关注Mips架构，与之相关的是龙芯CPU。

# 内核中内存屏障

Linux内核中实现了8中基本的CPU内存屏障，具体如下：

        TYPE            MANDATORY               SMP CONDITIONAL
        =============== ======================= ===========================
        GENERAL         mb()                    smp_mb()
        WRITE           wmb()                   smp_wmb()
        READ            rmb()                   smp_rmb()
        DATA DEPENDENCY read_barrier_depends()  smp_read_barrier_depends()

分别属于4中内存屏障类型，关于内存屏障的分类及相关原理，请参见我的其它文章

其中MANDATORY即强制内存屏障，不应该用于控制SMP环境。因为其在SMP和UP上都包含额外的消耗。其仅用于控制MMIO效果，当涉及设备相关的memory操作时，即使是SMP环境，也需要使用强制内存屏障。

SMP CONDITIONAL屏障用于SMP(简单说即多核)场景的内存访问。

# Mips架构的内存屏障实现

## SMP CONDITIONAL屏障

### smp_mb()

内核中，最基本的SMP内存屏障的接口为`smp_mb()`， 看看其在Mips架构中的实现：

	#define smp_mb()	__asm__ __volatile__(__WEAK_ORDERING_MB : : :"memory")
	
定义为一个宏，一句嵌入式汇编代码，其中volatile属性决定了编译器不会对其进行优化，意味者其隐含了编译器内存屏障。再看看`__WEAK_ORDERING_MB`的实现：

	#if defined(CONFIG_WEAK_ORDERING) && defined(CONFIG_SMP)
	#define __WEAK_ORDERING_MB	"       sync	\n"
	#else
	#define __WEAK_ORDERING_MB	"		\n"

即当配置了CONFIG_WEAK_ORDERING时，其实现为 **sync** 指令，未定义时，定义为空。

龙芯环境中，查看内核配置文件(/boot/config*)，确认该配置为y：

	CONFIG_WEAK_ORDERING=y
	
也就是说，龙芯3A3000环境中，`smp_mb()`内存屏障的实现，其本质就是一条 **sync** 指令，看似有点暴力。

### smp_wmb() & smp_rmb()

`smp_wmb()` 和 `smp_rmb()` 分别为基本的读写屏障，分别作用于load和store的场景。在Mips架构中，其实现与 `smp_mb()` 没有区别：

	#define smp_rmb()	__asm__ __volatile__(__WEAK_ORDERING_MB : : :"memory")
	#define smp_wmb()	__asm__ __volatile__(__WEAK_ORDERING_MB : : :"memory")
 
### smp_read_barrier_depends()
 
 smp_read_barrier_depends()是数据依赖屏障，在Mips架构中没有实现：
 
 	#define smp_read_barrier_depends()	do { } while(0)
 
##  MANDATORY屏障
 
### mb() & rmb() & wmb()
 
 这3个是最基本的强制内存屏障，分别对应于通用、读和写类型的内存屏障，其实现如下：
	 
	#ifdef CONFIG_CPU_HAS_WB
	
	#define wmb()		fast_wmb()
	#define rmb()		fast_rmb()
	#define mb()		wbflush()

	#else /* !CONFIG_CPU_HAS_WB */
	
	#define wmb()		fast_wmb()
	#define rmb()		fast_rmb()
	#define mb()		fast_mb()

 
 即当有内核配置CONFIG_CPU_HAS_WB时，其mb()实现不同。而龙芯环境的内存中，CONFIG_CPU_HAS_WB参数配置为y：
 
 	CONFIG_CPU_HAS_WB=y
 	
 所以，实现对应为：
 
 	#define wmb()		fast_wmb()
	#define rmb()		fast_rmb()
	#define mb()		wbflush()

#### wmb()
	
看看`fast_wmb()`的实现：

	#define fast_wmb()	__sync()
	
看看`__sync()`实现：

	#ifdef CONFIG_CPU_HAS_SYNC
	#define __sync()				\
		__asm__ __volatile__(			\
			".set	push\n\t"		\
			".set	noreorder\n\t"		\
			".set	mips2\n\t"		\
			"sync\n\t"			\
			".set	pop"			\
			: /* no output */		\
			: /* no input */		\
			: "memory")
	#else
	#define __sync()	do { } while(0)
	
龙芯环境中CONFIG_CPU_HAS_SYNC配置为y，所以其实现为一段汇编，其中，

- .set用于设置Mips汇编器的汇编指示。

- `.set push`和`.set pop`用于保存和恢复现有的汇编指示，目的就是想在不破坏现有的汇编指示环境的前提下执行一些命令，类似于函数调用过程中的压栈和出栈的操作。

- `.set	noreorder`。设置汇编器为noreorder模式，该模式下，不会对指令进行重新排序，也不会做任何相关优化，指令执行顺序完全按原程序编译后的指令顺序。而默认汇编器处在reorder的模式下，该模式允许汇编器对指令进行重新排序，以避免流水线堵塞并获得更好的性能。

- `.set	mips2`，设置汇编器使用的指令集级别，为后面的sync指令执行做准备，sync指令应该是属于mips2中的。

- `sync`，就是sync指令了

所以，`__sync()`实现的本质还是一条 **sync** 指令。

#### rmb()

再看看 `fast_rmb()` 的实现：

	#define fast_rmb()	__sync()
	
还是`__sync()`，与`wmb()`实现一样。

#### mb()

再看看`wbflush()`的实现：

	#ifdef CONFIG_CPU_HAS_WB
	
	#define wbflush()			\
		do {				\
			__sync();		\
			__wbflush();		\
		} while (0)
	
	#else /* !CONFIG_CPU_HAS_WB */
	...
	
实现为两个部分，第一部分还是一个`__sync()`(即sync指令)，另一部分 `__wbflush()`，其实现与架构相关，龙芯中的实现为：

	__wbflush = wbflush_loongson;
	
	static void wbflush_loongson(void)
	{
		asm(".set\tpush\n\t"
		    ".set\tnoreorder\n\t"
		    ".set mips3\n\t"
		    "sync\n\t"
		    "nop\n\t"
		    ".set\tpop\n\t"
		    ".set mips0\n\t");
	}
	
 其中，`.set`相关的前面已经说过了，看起来，其本质上就两条指令`sync`和`nop`
 
### read_barrier_depends() 

这是数据依赖屏障，Mips架构中依然没有实现：

	#define read_barrier_depends()		do { } while(0)
 
# 总结
 
 总的来看，龙芯(Mips)架构中，内存屏障基本都通过`sync`指令实现，这个估计会对影响有较大影响，是否有更好的实现方式尚待探讨。
 
# 附：X86的mb()实现

X86-64中的实现如下：

	/* Copied from include/asm-x86_64 for use by userspace. */
	#define mb() 	asm volatile("mfence":::"memory")
	
	#endif

就是mfence指令。

i386中实现如下：

	/* Copied from include/asm-i386 for use by userspace.  i386 has the option
	 * of using mfence, but I'm just using this, which works everywhere, for now.
	 */
	#define mb() asm volatile("lock; addl $0,0(%esp)")
	
	#endif

可以自己理解下。