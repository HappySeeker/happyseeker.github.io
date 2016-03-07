---
layout: post
title:  "Core Dump流程分析"
date:   2016-03-04 12:10:09
author: HappySeeker
categories: Kernel
---

# 闲话

最近分析问题时，发现我的环境中，经常有用户态进程异常退出，但是却没有core文件生成，简单看了一下相关的内核流程，mark一下。

# Core Dump基本原理

当应用程序在用户态发生异常时，比如常见的段错误，通常会生成core文件，通过gdb分析core文件，基本就能定位问题。

core文件实际是当前进程地址空间的一个镜像文件，其中包含了该进程在内存中的所有信息。

Linux系统中，Core Dump的主要流程描述为：

1. 进程运行过程中出现异常(比如段错误)，该异常为硬件上报，可理解为一个中断，异常上报后，CPU在执行完当前的指令后会立刻停止执行当前上下文中的指令，而进入异常处理流程：常规的保存上下文，跳转到相应的中断向量处执行等，具体不详述了。
2. 进入异常处理流程后，会判断发生异常的CPU模式(X86中对应相应的Ring)，如果发生异常时，处于用户态，则说明该异常只会影响当前进程，于是向用户态进程发送相应的信号，如果是段错误，发送的信号为SIGSEGV,如果为Aborts，通常发送的信号为SIGBUS。如果发生异常时处于内核态，则说明该异常对系统是致命的，则会进入内核die流程，最终会panic，这种情况不会生成core文件，不是这里讨论的重点。
3. 当用户态进程有处理信号的时机时(比如得到调度)，其会检查pending的信号，然后进行处理，如果发现pending的信号为`SIG_KERNEL_COREDUMP_MASK`中的一个，则会进入core文件搜集流程，最终根据用户配置生成core文件。

# 代码分析

这里以ARM64架构中，非对齐异常的流程来说明core dump的流程。

## 信号发送

当硬件检测到alignment fault时，会进入相应的异常处理函数(其他文章已经分析过ARM64的异常处理流程了，可以参考)处理，对于alignment fault，最终会进入`do_mem_abort()`函数：

	/*
	 * Dispatch a data abort to the relevant handler.
	 */
	asmlinkage void __exception do_mem_abort(unsigned long addr, unsigned int esr,
						 struct pt_regs *regs)
	{
		/*从ESR中提取fault信息*/
		const struct fault_info *inf = fault_info + (esr & 63);
		struct siginfo info;
	
		if (!inf->fn(addr, esr, regs))
			return;
		/*我们在messages中看到的打印*/
		pr_alert("Unhandled fault: %s (0x%08x) at 0x%016lx\n",
			 inf->name, esr, addr);
		/*填充信号信息*/
		info.si_signo = inf->sig;
		info.si_errno = 0;
		info.si_code  = inf->code;
		info.si_addr  = (void __user *)addr;
		/*通知用户态程序或者是内核die*/
		arm64_notify_die("", regs, &info, esr);
	}

这里的填充的信号是来自ESR，对于alignment fault，默认为SIGBUS信号，然后还是进入了`arm64_notify_die（）`：

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

对于用户态发生的异常，会发送相应的信号，后续的信号发送主要代码流程为(不详述)：

	force_sig_info -> 
	  specific_send_sig_info ->
	    send_signal ->
		  __send_signal

## 信号处理

信号处理是异步过程，内核向用户态进程发送信号后，信号并不会得到立即处理，信号的处理时机有几个，最主要的就是就进程得到调度的时候，当进程得到调度后，就会处理相应的信号，信号处理的主要代码流程如下：

	do_signal ->
	  get_signal ->
		sig_kernel_coredump ->
		  do_coredump

具体代码就不列了，有点乏味~，这里主要强调一下，什么信号会最终导致core文件的生成，关键就在于`SIG_KERNEL_COREDUMP_MASK`宏：

	#define SIG_KERNEL_COREDUMP_MASK (\
	        rt_sigmask(SIGQUIT)   |  rt_sigmask(SIGILL)    | \
		rt_sigmask(SIGTRAP)   |  rt_sigmask(SIGABRT)   | \
	        rt_sigmask(SIGFPE)    |  rt_sigmask(SIGSEGV)   | \
		rt_sigmask(SIGBUS)    |  rt_sigmask(SIGSYS)    | \
	        rt_sigmask(SIGXCPU)   |  rt_sigmask(SIGXFSZ)   | \
		SIGEMT_MASK	

这个宏中列出的信号都会产生core文件，信号不只是内核可以发送，用户态当然也可以发送，最简单的就是kill命令，如果向让某进程产生core文件，可以直接向其发送信号(比如SIGSEGV、SIGBUS)，当然最简单的可能就是gcore命令了，损伤最小，扯远了~

# 相关配置

最后，再补充一下core文件的相关配置，主要就是两个地方：

1. /proc/sys/kernel/core_pattern，也可以通过sysctl.conf配置，用于控制core文件的生成规则。
2. ulimit -c控制core文件的大小，如果设置为0，则不会生成core文件。

不罗嗦其他的了。


