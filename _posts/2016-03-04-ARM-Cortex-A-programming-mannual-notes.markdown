---
layout: post
title:  "ARM Cortex-A 编程手册学习笔记"
date:   2016-03-05 6:17:13
author: HappySeeker
categories: Kernel
---

# 闲话

从前都在X86上分析内核，做开发、trouble shooting，对于其他架构了解较少，对于新架构的学习，甚至还有些抵触，这次趁分析问题的机会，顺便学习了一下ARM架构的基础知识，权当笔记。

这里主要是AArch32架构(即32位，后面就都简写成ARM了)，相对比较简单，入门必备。

# ARM处理器模式

在引入安全扩展之前，ARM有7中处理器模式，其中6种是特权模式，剩下一种为用户程序运行的非特权模式。特权模式下，可以做一些user模式下不能做的操作，比如mmu配置和cache操作。

| Mode       | Function         | Privilege |
| ---------- |------------------| --------- |
| User (USR) | Mode in which most programs and applications run | Unprivileged |

| FIQ | Entered on an FIQ interrupt exception IRQ Entered on an IRQ interrupt exception |     |
|Supervisor (SVC) | Entered on reset or when a Supervisor Call instruction (SVC)
is executed |    |
| Abort (ABT) | Entered on a memory access exception Undef (UND) Entered when an undefined instruction executed |   |
| System (SYS) | Mode in which the OS runs, sharing the register view |    |

TrustZone Security Extensions引入了独立于模式的两种安全状态，和一种新的Monitor模式。如此就有8种CPU模式了。

TrustZone Security Extensions的环境中，通过将所有的硬件和软件资源按设备来划分，来提升系统安全性。 在CPU处于Normal (Non-secure)  state，其不能访问在Secure state下分配的内存。

当前处理器所处模式，由CPSR(当前程序状态寄存器)寄存器的值决定，改变cpu模式可以通过特权软件显示设置该寄存器，也可以由异常触发(发生异常后，CPU就会自动切换到对应的模式)。

# 寄存器

ARM架构提供16个32位的通用寄存器(R0-R15),R0-R14可用作普通的数据存储。

R15为PC寄存器，修改R15可以改变程序的执行流程。PC值指向当前执行指令的地址加8处，PC是有读写限制的。

R14也称LR(Link register)，通常用于保存当前函数的返回地址(上一级函数)，当其空闲时，也可以用作普通寄存器用。每种模式下都有自己独立(物理上)的R14。

R13也称SP，即用于保存堆栈的栈顶指针，当其空闲时，也可以用作普通寄存器用。每一种异常模式都有其自己独立的r13，它通常指向异常模式所专用的堆栈。当ARM进入异常模式的时候，程序就可以把一般通用寄存器压入堆栈，返回时再出栈，保证了各种模式下程序的状态的完整性。

程序也可以访问CPSR，SPSR是前一个执行模式下CPSR的副本。

虽然软件可以访问这些寄存器，但是不同模式下，部分寄存器实际对应的物理存储位置可能不同，也就是说，不同模式下，部分寄存器(除R0-7和R15外)实际物理上是不同(虽然对软件来说是透明的，软件看到的是相同的逻辑上的寄存器)，这些寄存器只能在相应模式下才能访问。

Program Status Registers。CPSR用于存储：flags、当前CPU模式、中断禁用标记、当前CPU状态等。在user模式下，CPSR对应的寄存器为APSR(限制版本的CPSR)。

# Coprocessor 15

CP15，系统控制协处理器，用于控制Core的需要特性，包括c0-c15主要的32位寄存器，这些寄存器通常也通过名称访问，如CP15.SCTLR

# Cache

ARM中的常见Cache架构如下：

![ARM中Cache架构](https://github.com/HappySeeker/happyseeker.github.io/raw/master/Cache-arrangement-in-AArch32.png)

## cache问题

让程序执行时间变得不可知。

对于实时性要求高的系统来说有点不可接受。

外设需要用up-to-date的数据，需要cache控制。

## 其他

tag占物理空间，但不计算在cache size内

invalidate操作：丢掉cache。

clean操作：将脏数据刷入内存，保证一致。

## Point of coherency and unification

For set/way based clean and invalidate, the operation is performed on a specific level of cache.For operations that use a virtual address, the architecture defines two conceptual points:

### Point of Coherency (PoC)
For a particular address, the PoC is the point at which all blocks, for example,
cores, DSPs, or DMA engines, that can access memory are guaranteed to see the
same copy of a memory location. Typically, this will be the main external system
memory.

### Point of Unification (PoU)
The PoU for a core is the point at which the instruction and data caches of the core
are guaranteed to see the same copy of a memory location. For example, a unified
level 2 cache would be the point of unification in a system with Harvard level 1
caches and a TLB for cacheing translation table entries. If no external cache is
present, main memory would be the Point of unification.







ARM64中包含如下几种类型的异常：

1. 中断(Interrupts)，就是我们平常理解的中断，主要由外设触发，是典型的异步异常。 ARM64中主要包括两种类型的中断：IRQ(普通中断)和FIQ(高优先级中断，处理更快)。 Linux内核中好像没有使用FIQ，还没有仔细看代码，具体不详述了。
2. Aborts。 可能是同步或异步异常。包括指令异常(取指令时产生)、数据异常(读写内存数据时产生)，可以由MMU产生(比如典型的缺页异常)，也可以由外部存储系统产生(通常是硬件问题)。
3. Reset。复位被视为一种特殊的异常。
4. Exception generating instructions。由异常触发指令触发的异常，比如Supervisor Call (SVC)、Hypervisor Call (HVC)、Secure monitor Call (SMC)

## 异常级别(EL)

ARM中，异常由级别之分，具体如下图所示，只要关注：

普通的用户程序处于EL0，级别最低

内核处于EL1，HyperV处于EL2，EL1-3属于特权级别。

![Arm64中的异常级别](https://github.com/HappySeeker/happyseeker.github.io/raw/master/_posts/Exception-Level-in-AArch64.png)

## 异常处理

Arm中的异常处理过程与X86比较相似，同样包括硬件自动完成部分和软件部分，同样需要设置中断向量，保存上下文，不同的异常类型的处理方式可能有细微差别。 这里不详述了。

需要关注：用户态(EL0)不能处理异常，当异常发生在用户态时，异常级别(EL)会发生切换，默认切换到EL1(内核态)。

## 中断向量表

Arm64架构中的中断向量表有点特别(相对于X86来说~)，包含16个entry，这16个entry分为4组，每组包含4个entry，每组中的4个entry分别对应4种类型的异常：

1. SError
2. FIQ
3. IRQ
4. Synchronous Aborts

4个组的分类根据发生异常时是否发生异常级别切换、和使用的堆栈指针来区别。分别对应于如下4组：

1. 异常发生在当前级别且使用SP_EL0(EL0级别对应的堆栈指针)，即发生异常时不发生异常级别切换，可以简单理解为异常发生在内核态(EL1)，且使用EL0级别对应的SP。 这种情况在Linux内核中未进行实质处理，直接进入bad_mode()流程。
2. 异常发生在当前级别且使用SP_ELx(ELx级别对应的堆栈指针，x可能为1、2、3)，即发生异常时不发生异常级别切换，可以简单理解为异常发生在内核态(EL1)，且使用EL1级别对应的SP。 这是比较常见的场景。
3. 异常发生在更低级别且在异常处理时使用AArch64模式。 可以简单理解为异常发生在用户态，且进入内核处理异常时，使用的是AArch64执行模式(非AArch32模式)。 这也是比较常见的场景。
4. 异常发生在更低级别且在异常处理时使用AArch32模式。 可以简单理解为异常发生在用户态，且进入内核处理异常时，使用的是AArch32执行模式(非AArch64模式)。 这中场景基本未做处理。


# 代码分析
## 中断向量表

Linux内核中，中断向量表实现在entry.S文件中，代码如下：

		/*
		 * Exception vectors.
		 */
		
			.align	11
		/*el1代表内核态，el0代表用户态*/
		ENTRY(vectors)
			ventry	el1_sync_invalid		// Synchronous EL1t 
			ventry	el1_irq_invalid			// IRQ EL1t
			ventry	el1_fiq_invalid			// FIQ EL1t
			/*内核态System Error ，使用SP_EL0(用户态栈)*/
			ventry	el1_error_invalid		// Error EL1t
		
			ventry	el1_sync			// Synchronous EL1h
			ventry	el1_irq				// IRQ EL1h
			ventry	el1_fiq_invalid			// FIQ EL1h
			/*内核态System Error ，使用SP_EL1(内核态栈)*/
			ventry	el1_error_invalid		// Error EL1h
		
			ventry	el0_sync			// Synchronous 64-bit EL0
			ventry	el0_irq				// IRQ 64-bit EL0
			ventry	el0_fiq_invalid			// FIQ 64-bit EL0
			/*用户态System Error ，使用SP_EL1(内核态栈)*/
			ventry	el0_error_invalid		// Error 64-bit EL0
		
		#ifdef CONFIG_COMPAT
			ventry	el0_sync_compat			// Synchronous 32-bit EL0
			ventry	el0_irq_compat			// IRQ 32-bit EL0
			ventry	el0_fiq_invalid_compat		// FIQ 32-bit EL0
			ventry	el0_error_invalid_compat	// Error 32-bit EL0
		#else
			ventry	el0_sync_invalid		// Synchronous 32-bit EL0
			ventry	el0_irq_invalid			// IRQ 32-bit EL0
			ventry	el0_fiq_invalid			// FIQ 32-bit EL0
			ventry	el0_error_invalid		// Error 32-bit EL0
		#endif
		END(vectors)


可以明显看出分组和分类的情况。 

## invalid类处理
带invalid后缀的向量都是Linux做未做进一步处理的向量，默认都会进入bad_mode()流程，说明这类异常Linux内核无法处理，只能上报给用户进程(用户态，sigkill或sigbus信号)或die(内核态)

带invalid后缀的向量最终都调用了inv_entry，inv_entry实现如下：

	/*
	 * Invalid mode handlers
	 */
 	/*Invalid类异常都在这里处理，统一调用bad_mode函数*/
	.macro	inv_entry, el, reason, regsize = 64
	kernel_entry el, \regsize
	/*传入bad_mode的三个参数*/
	mov	x0, sp
	/*reason由上一级传入*/
	mov	x1, #\reason
	/*esr_el1是EL1(内核态)级的ESR(异常状态寄存器)，用于记录异常的详细信息，具体内容解析需要参考硬件手册*/
	mrs	x2, esr_el1
	/*调用bad_mode函数*/
	b	bad_mode
	.endm


调用bad_mode，是C函数，通知用户态进程或者panic。

	/*
	 * bad_mode handles the impossible case in the exception vector.
	 */
	asmlinkage void bad_mode(struct pt_regs *regs, int reason, unsigned int esr)
	{
		siginfo_t info;
		/*获取异常时的PC指针*/
		void __user *pc = (void __user *)instruction_pointer(regs);
		console_verbose();
		/*打印异常信息，messages中可以看到。*/
		pr_crit("Bad mode in %s handler detected, code 0x%08x -- %s\n",
			handler[reason], esr, esr_get_class_string(esr));
		/*打印寄存器内容*/
		__show_regs(regs);
		/*如果发生在用户态，需要向其发送信号，这种情况下，发送SIGILL信号，所以就不会有core文件产生了*/
		info.si_signo = SIGILL;
		info.si_errno = 0;
		info.si_code  = ILL_ILLOPC;
		info.si_addr  = pc;
		/*给用户态进程发生信号，或者die然后panic*/
		arm64_notify_die("Oops - bad mode", regs, &info, 0);
	}


arm64\_notify_die：

	void arm64_notify_die(const char *str, struct pt_regs *regs,
			      struct siginfo *info, int err)
	{
		/*如果发生异常的上下文处于用户态，则给相应的用户态进程发送信号*/
		if (user_mode(regs)) {
			current->thread.fault_address = 0;
			current->thread.fault_code = err;
			force_sig_info(info->si_signo, info, current);
		} else {
			/*如果是内核态，则直接die，最终会panic*/
			die(str, regs, err);
		}
	}

## IRQ中断处理

场景的场景中(不考虑EL2和EL3)，IRQ处理分两种情况：用户态发生的中断和内核态发生的中断，相应的中断处理接口分别为：

	el0_sync
	el1_sync

相应代码如下：

el1_sync：

		.align	6
	el1_irq:
		/*保存中断上下文*/
		kernel_entry 1
		enable_dbg
	#ifdef CONFIG_TRACE_IRQFLAGS
		bl	trace_hardirqs_off
	#endif
		/*调用中断处理默认函数*/
		irq_handler
		/*如果支持抢占，处理稍复杂*/
	#ifdef CONFIG_PREEMPT
		get_thread_info tsk
		ldr	w24, [tsk, #TI_PREEMPT]		// get preempt count
		cbnz	w24, 1f				// preempt count != 0
		ldr	x0, [tsk, #TI_FLAGS]		// get flags
		tbz	x0, #TIF_NEED_RESCHED, 1f	// needs rescheduling?
		bl	el1_preempt
	1:
	#endif
	#ifdef CONFIG_TRACE_IRQFLAGS
		bl	trace_hardirqs_on
	#endif
		/*恢复上下文*/
		kernel_exit 1
	ENDPROC(el1_irq)

代码非常简单，主要就是调用了irq_handler()函数，不做深入解析了，有兴趣可以自己再看看代码。

el0_sync处理类似，主要区别在于：其涉及用户态和内核态的上下文切换和恢复。

		.align	6
	el0_irq:
		kernel_entry 0
	el0_irq_naked:
		enable_dbg
	#ifdef CONFIG_TRACE_IRQFLAGS
		bl	trace_hardirqs_off
	#endif
		/*退出用户上下文*/
		ct_user_exit
		irq_handler
	
	#ifdef CONFIG_TRACE_IRQFLAGS
		bl	trace_hardirqs_on
	#endif
		/*返回用户态*/
		b	ret_to_user
	ENDPROC(el0_irq)