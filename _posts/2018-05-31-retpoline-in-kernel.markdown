---
layout: post
title:  "Retpoline in kernel"
date:   2018-05-31 7:02:28
author: JiangBiao
categories: Kernel
---

#  背景

继续Meltdown && Spectre安全漏洞的相关分析，本文关注V2 patch中引入的retpoline技术。

# 基本原理

## IBRS/IBPB

在原有的V1版本的patch中，针对Spectre V2漏洞(这个漏洞也是一同的3个漏洞中，解决起来最麻烦的一个)的解决方案是：

	IBRS/IBPB + 固件升级
	
思路为：升级固件，让CPU提供(通过MSR寄存器)新的硬件特性(SPEC_CTRL)，用来：

1. 限制间接分支预测。Indirect Branch Restricted Speculation，即IBRS。通过相关接口(MSR寄存器中的特殊位)，可以限制间接分支预测，达到规避漏洞的目的。IBRS主要用在用户态项内核态切换时，防止用户态窃取到内核态的非授权数据。IBRS实现比较厚重，对影响比较大，实测unixbench，对性能影响在20%以上。
2. 间接分支预测屏障。Indirect Branch Prediction Barriers，即IBPB。也是通过类似接口实现。其本质类似于一个内存屏障，用来防止CPU跨过该屏障进行分支预测。IBPB主要用在进程上下文切换时，防止上一个进程通过预测执行，窃取到下一个进程的私密数据，在云化场景中，即能避免虚拟机之间的数据泄露。IBPB实现相对轻量，对性能影响相对较小。

## Retpoline

Retpoline是针对Spectre V2漏洞(CVE-2017-5715)提出的规避方案。在V2版本中有google的工程师提出，主要用于解决IBRS带来的难以接受的性能损耗问题。

Retpoline的基本原理：

在间接指令跳转时，在原有代码中加入不会被执行到的死循环代码，于时在CPU执行间接跳转指令(JMP/CALL)时，进行预测执行时，只能预取到被加入的死循环代码，不会预取到内核中的其他数据，从而防止了数据泄露，规避掉了Spectre V2。

由于没有几乎没有添加额外的执行流程(添加的死循环代码永远不会被执行)，所以对性能影响比较小，官方给出的数据是：0.5%-2%。

原理比较简单，一些关键的知识点：

1、只针对间接跳转指令。即JMP和CALL指令。
2、对于C代码，包括内核代码和应用程序代码。需要借助新的编译器选项-mindirect-branch=thunk-extern，通过新的编译器(gcc)来实现死循环代码的插入和跳转。
3、对于内核中的汇编代码。由于不能借助编译器完成，则只能自己实现，其原理跟编译器中的类似，如前面所述。本文主要关注这部分内容。

## 内核中retpoline的实现原理

内核中，存在泄密风险的关键点在于用户态向内核态切换时，而这部分代码都是用汇编实现，需要内核自己实现retpoline来规避风险。原理如前面所述，比较简单。其主要是对原有的JMP和CALL指令进行了封装，封装后的指令中包含了跳转和死循环代码。后面具体分析代码。

# 代码分析

## JMP封装

在系统调用的典型场景中，retpoline补丁在用户态->内核态切换时，将原有的JMP指令，换成了JMP_NOSPEC指令：

	ENTRY(entry_SYSCALL_64_trampoline)
		UNWIND_HINT_EMPTY
		swapgs

		/* Stash the user RSP. */
		movq	%rsp, RSP_SCRATCH

		/* Note: using %rsp as a scratch reg. */
		SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp

		/* Load the top of the task stack into RSP */
		movq	CPU_ENTRY_AREA_tss + TSS_sp1 + CPU_ENTRY_AREA, %rsp

		/* Start building the simulated IRET frame. */
		pushq	$__USER_DS			/* pt_regs->ss */
		pushq	RSP_SCRATCH			/* pt_regs->sp */
		pushq	%r11				/* pt_regs->flags */
		pushq	$__USER_CS			/* pt_regs->cs */
		pushq	%rcx				/* pt_regs->ip */

		/*
		 * x86 lacks a near absolute jump, and we can't jump to the real
		 * entry text with a relative jump.  We could push the target
		 * address and then use retq, but this destroys the pipeline on
		 * many CPUs (wasting over 20 cycles on Sandy Bridge).  Instead,
		 * spill RDI and restore it in a second-stage trampoline.
		 */
		pushq	%rdi
		movq	$entry_SYSCALL_64_stage2, %rdi
		//这里进行了指令替换，使用了封装后的指令
		JMP_NOSPEC %rdi
	END(entry_SYSCALL_64_trampoline)

继续看看JMP_NOSPEC的实现：

	/*
	 * JMP_NOSPEC and CALL_NOSPEC macros can be used instead of a simple
	 * indirect jmp/call which may be susceptible to the Spectre variant 2
	 * attack.
	 */
	 //  JMP_NOSPEC宏入口，前面的注释写的很明白
	.macro JMP_NOSPEC reg:req
	// 仅当内核配置选项打开时
	#ifdef CONFIG_RETPOLINE
		ANNOTATE_NOSPEC_ALTERNATIVE
		/*
		 * 这里相当于一个条件分支，意思是：
		 * 1. 当X86_FEATURE_RETPOLINE设置时，执行RETPOLINE_JMP \reg，这是我们主要关注的分支
		 * 2. 当X86_FEATURE_RETPOLINE_AMD设置时，执行lfence; ANNOTATE_RETPOLINE_SAFE; jmp *\reg
		 * 3. 当前面两个都没有设置时，执行ANNOTATE_RETPOLINE_SAFE; jmp *\reg
		 */
		ALTERNATIVE_2 __stringify(ANNOTATE_RETPOLINE_SAFE; jmp *\reg),	\
			__stringify(RETPOLINE_JMP \reg), X86_FEATURE_RETPOLINE,	\
			__stringify(lfence; ANNOTATE_RETPOLINE_SAFE; jmp *\reg), X86_FEATURE_RETPOLINE_AMD
	#else
		//当内核选项没有配置时，就是原来的jmp指令，对性能没有任何影响。
		jmp	*\reg
	#endif
	.endm

### X86_FEATURE_RETPOLINE场景

当X86_FEATURE_RETPOLINE设置时(Intel CPU使用)，JMP_NOSPEC reg执行的指令为：

	RETPOLINE_JMP \reg

需要继续看RETPOLINE_JMP：

	/*
	 * These are the bare retpoline primitives for indirect jmp and call.
	 * Do not use these directly; they only exist to make the ALTERNATIVE
	 * invocation below less ugly.
	 */
	.macro RETPOLINE_JMP reg:req
		// 跳转到后面的标签，跳过了中间的死循环代码，意味者这段死循环代码不会被执行
		call	.Ldo_rop_\@
	// 死循环代码，不会被实际调用，当CPU预测执行时，可能会预取到这里，但是什么也做不了
	.Lspec_trap_\@:
		pause
		lfence
		jmp	.Lspec_trap_\@
	.Ldo_rop_\@:
		/* 
		 * 将reg寄存器内容压栈，由于前面调用的是call指令，所以在ret时会进行出栈操作，
		 * 会将刚压栈的寄存器内容作为目标地址，进行跳转，跳转到reg内容地址继续执行。
		 * 所以，整个RETPOLINE_JMP reg代码本质上就是一个jmp reg指令。
		mov	\reg, (%_ASM_SP)
		ret
	.endm

从上面代码可以看出，当X86_FEATURE_RETPOLINE设置时，JMP_NOSPEC reg本质上就是一个jmp reg指令。只是加入了一些不会被执行到的死循环。

### 未设置retpoline场景

当X86_FEATURE_RETPOLINE和X86_FEATURE_RETPOLINE_AMD都没有设置时，JMP_NOSPEC reg执行的指令为：

	ANNOTATE_RETPOLINE_SAFE; jmp *\reg

继续看ANNOTATE_RETPOLINE_SAFE：

	/*
	 * This should be used immediately before an indirect jump/call. It tells
	 * objtool the subsequent indirect jump/call is vouched safe for retpoline
	 * builds.
	 */
	.macro ANNOTATE_RETPOLINE_SAFE
		.Lannotate_\@:
		.pushsection .discard.retpoline_safe
		_ASM_PTR .Lannotate_\@
		.popsection
	.endm

个人理解，这段代码只是在最终的elf文件头中加入了一个特殊的section：.discard.retpoline_safe，供objtool识别，告诉objtool随后的jmp和call指令是安全的。
本质上没有任何影响代码执行流的功能。基本可以忽略。

所以，这种场景下，其本质为jmp reg。相当于原来的语义。

### X86_FEATURE_RETPOLINE_AMD场景

当X86_FEATURE_RETPOLINE_AMD设置时(AMD CPU使用)，JMP_NOSPEC reg执行的指令为：

	lfence; ANNOTATE_RETPOLINE_SAFE; jmp *\reg

结合前面的分析，这里其实就是在jmp指令之前添加了一条：lfence指令，即在AMD的硬件环境中，retpoline本质上就是加了一个内存屏障，通过内存屏障来实现，并没有使用前面说的“加入死循环”。这是由硬件特性决定的。AMD的服务器硬件比较少，没有实际测试性能影响的差异。

## CALL封装

CALL封装后的宏为CALL_NOSPEC，其本质上是基于RETPOLINE_JMP实现的，所以相关逻辑基本类似，这里看看主要代码即可：

	.macro CALL_NOSPEC reg:req
	#ifdef CONFIG_RETPOLINE
		// 与前面分析类似
		ANNOTATE_NOSPEC_ALTERNATIVE
		ALTERNATIVE_2 __stringify(ANNOTATE_RETPOLINE_SAFE; call *\reg),	\
			__stringify(RETPOLINE_CALL \reg), X86_FEATURE_RETPOLINE,\
			__stringify(lfence; ANNOTATE_RETPOLINE_SAFE; call *\reg), X86_FEATURE_RETPOLINE_AMD
	#else
		call	*\reg
	#endif
	.endm

RETPOLINE_CALL：

	/*
	 * This is a wrapper around RETPOLINE_JMP so the called function in reg
	 * returns to the instruction after the macro.
	 */
	 // 通过封装RETPOLINE_JMP实现
	.macro RETPOLINE_CALL reg:req
		// 跳转到后面
		jmp	.Ldo_call_\@
	.Ldo_retpoline_jmp_\@:
		// 调用RETPOLINE_JMP实现
		RETPOLINE_JMP \reg
	.Ldo_call_\@:
		// 然后再跳转到前面
		call	.Ldo_retpoline_jmp_\@
	.endm
