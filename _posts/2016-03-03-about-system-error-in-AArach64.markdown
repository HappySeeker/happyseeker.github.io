---
layout: post
title:  "闲聊System Error(SError) in AArch64"
date:   2016-03-03 6:25:12
author: HappySeeker
categories: Kernel
---

# 闲话

自己的AArach64(ARMv8)环境中，由出现了SError异常，于是又做了一番了解和研究，再次记录，权当闲聊。

# 什么SError？

SError全称为：System Error，是ARM架构中的一种类型的异常，从字面上看，好像太宽泛，无法得知更多的信息。

AArch64(ARM64)架构中，主要包括如下4中类型的异常：

1. Synchronous exception(同步异常)，“同步”可以理解为：发生异常的指令为导致异常的指令，即当导致异常发生的指令执行时能立即触发异常。 包括ARM架构中定义的所有Aborts异常，如：指令异常、数据异常、对齐异常等。
2. SError，System Error，是一种异步异常，后面再仔细说明。
3. IRQ，普通的中断，是异步异常。
4. FIQ，高优先级的中断，是异步异常。

# SError产生的原因？

到了大家最关心的问题，什么原因可能导致SError。 这个名字实在太宽泛，根据Arm相关手册的说明，其可能原因有很多，比较常见的原因包括：

- asynchronous Data Aborts，异步数据异常，数据异常即CPU读写内存数据时产生的异常。 比如：如果误将ROM对应的区域的页表项设置为RW，同时该内存熟悉为write-back cacheable，当尝试写这段区域内存时，数据会先写入cache，而在后面的某个时间点，当其他操作触发相应的脏的cacheline回写内存时，此时内存系统就会返回错误(因为ROM是只读的)，从而触发一个异步数据异常。 在Armv8环境中，这就是一个标准的SError。
- 外部引脚触发。 部分处理器，比如Cortex-A5x，拥有独立的引脚来触发SError，也就是说此时的SError是由SoC设计者自己定义的，比如：ECC和奇偶校验错误就常作为SError来触发。具体应该参考SoC的参考手册来确认SError可能的对应的错误类型。


# 内核如何处理SError？

Linux内核中，对SError进行了捕获，设置了相应的中断向量，当并未做实际的处理，只是上报异常，并终止进程会内核，因为对于内核来说，SError是致命的，内核自身无法做相应的修复操作，内核不知道具体原因，也不知道如何修复。

内核中相应的处理具体包括：

- 设置了相应的中断向量，对应的中断向量设置在entry.S汇编代码中，AArch64对应的四种状态下的SError对应中断处理函数都一样：

		/\*el1代表内核态，el0代表用户态*/
	
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
		
			...
		END(vectors)

el0\_error_invalid实现：

	el0_error_invalid:
		inv_entry 0, BAD_ERROR
	ENDPROC(el0_error_invalid)

el1\_error_invalid实现：

	el1_error_invalid:
		inv_entry 1, BAD_ERROR
	ENDPROC(el1_error_invalid)

可见，最终都调用了inv_entry，inv_entry实现如下：

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


- 调用bad_mode，是C函数，通知用户态进程或者panic。

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
