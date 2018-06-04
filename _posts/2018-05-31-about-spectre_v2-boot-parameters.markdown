---
layout: post
title:  "spectre_v2启动参数"
date:   2018-05-31 17:05:59
author: JiangBiao
categories: Kernel
---

#  背景

针对Meltdown && Spectre安全漏洞的V2 patch中引入了retpoline技术，启动参数也变了，主要通过spectre_v2控制，但是可选的参数众多，有点眼晕，本文分析相关原理和实现代码。

# 参数说明

内核文档：kernel-parameters.txt中针对spectre_v2参数有如下说明： 

	spectre_v2=	[X86] Control mitigation of Spectre variant 2 (indirect branch speculation) vulnerability.
			on   - unconditionally enable
			off  - unconditionally disable
			auto - kernel detects whether your CPU model is vulnerable
			Selecting 'on' will, and 'auto' may, choose a
			mitigation method at run time according to the
			CPU, the available microcode, the setting of the
			CONFIG_RETPOLINE configuration option, and the
			compiler with which the kernel was built.
			Specific mitigations can also be selected manually:
			retpoline	  - replace indirect branches
			retpoline,generic - google's original retpoline
			retpoline,amd     - AMD-specific minimal thunk
			Not specifying this option is equivalent to
			spectre_v2=auto. 

其实说的蛮清楚，但是就是有点复杂。我总结一下关键点：

1. on，表示无论硬件如何，都开启。
2. off，表示物理硬件如何，都关闭。
3. auto，表示内核根据当前CPU model自动选择mitigation方法。参考的因素包括
  - cpu model
  - 是否有可用微码(也就是针对IBRS和IBPB提供的固件)
  - 内核配置选项CONFIG_RETPOLINE是否配置
  - 编译内核用的编译器是否提供了新的编译选项：-mindirect-branch=thunk-extern
  情况比较复杂。
4. retpoline/retpoline,generic/retpoline,amd。这几种就让人有点眼晕了，啥？retpoline还有3种？

事实上要比你看到的还要晕。上述内核文档中对这几种参数的说明也比较糢糊，不容易理解，需要实际看看代码。

# 代码分析

相关的主要代码都在spectre_v2_select_mitigation()函数中：

	// 通过传入的内核启动参数，结合硬件以及相关配置，设置最终的migigation mode
	static void __init spectre_v2_select_mitigation(void)
	{
		// 解析cmdline
		enum spectre_v2_mitigation_cmd cmd = spectre_v2_parse_cmdline();
		// 默认mode 为 SPECTRE_V2_NONE
		enum spectre_v2_mitigation mode = SPECTRE_V2_NONE;

		/*
		 * If the CPU is not affected and the command line mode is NONE or AUTO
		 * then nothing to do.
		 */
		// 如果CPU不受Spectre_V2影响，则不处理，即mode为默认SPECTRE_V2_NONE
		if (!boot_cpu_has_bug(X86_BUG_SPECTRE_V2) &&
		    (cmd == SPECTRE_V2_CMD_NONE || cmd == SPECTRE_V2_CMD_AUTO)) {
			pr_info("Not affected\n");
			return;
		}
		// 判断参数
		switch (cmd) {
		// spectre_v2=off
		case SPECTRE_V2_CMD_NONE:
			return;
		// spectre_v2=on
		case SPECTRE_V2_CMD_FORCE:
		// spectre_v2=auto或者没有设置
		case SPECTRE_V2_CMD_AUTO:
			// 如果配置了内核选项CONFIG_RETPOLINE，则进入auto
			if (IS_ENABLED(CONFIG_RETPOLINE))
				goto retpoline_auto;
			break;
		// spectre_v2=retpoline,amd
		case SPECTRE_V2_CMD_RETPOLINE_AMD:
			if (IS_ENABLED(CONFIG_RETPOLINE))
				goto retpoline_amd;
			break;
		// spectre_v2=retpoline,generic
		case SPECTRE_V2_CMD_RETPOLINE_GENERIC:
			if (IS_ENABLED(CONFIG_RETPOLINE))
				goto retpoline_generic;
			break;
		// spectre_v2=retpoline, 进入auto，相当于spectre_v2=auto
		case SPECTRE_V2_CMD_RETPOLINE:
			if (IS_ENABLED(CONFIG_RETPOLINE))
				goto retpoline_auto;
			break;
		}
		pr_err("Mitigation: kernel not compiled with retpoline; no mitigation available!");
		return;
	// auto方式，根据不同的条件，决定最后的mode
	retpoline_auto:
		// 如果是AMD CPU
		if (boot_cpu_data.x86_vendor == X86_VENDOR_AMD) {
		retpoline_amd:
			// AMD主要通过硬件提供的lfence来实现repoline，如果没有，则使用传统的方式(AMD也可以使用传统方式哦)
			if (!boot_cpu_has(X86_FEATURE_LFENCE_RDTSC)) {
				pr_err("Mitigation: LFENCE not serializing, switching to generic retpoline\n");
				goto retpoline_generic;
			}
			/* 
			 * 判断当前编译器是否支持-mindirect-branch=thunk-extern选项，
			 * 如果支持，则mode为SPECTRE_V2_RETPOLINE_AMD，表示内核中C代码和汇编代码中包含CALL/JMP指令
			 * 都经过了封装，采用retpoline方式。
			 * 如果不支持，则mode为SPECTRE_V2_RETPOLINE_MINIMAL_AMD，表示只有汇编代码中的CALL/JMP指令
			 * 经过了封装，采用retpoline方式，C代码中由于没有编译器支持，不能使用retpoline方式封装。
			 */
			mode = retp_compiler() ? SPECTRE_V2_RETPOLINE_AMD :
						 SPECTRE_V2_RETPOLINE_MINIMAL_AMD;
			setup_force_cpu_cap(X86_FEATURE_RETPOLINE_AMD);
			setup_force_cpu_cap(X86_FEATURE_RETPOLINE);
		} else {// 如果不是AMD CPU
		retpoline_generic:
			// 与上面类似，分为MINIMAL和普通两种方式，取决于编译器是否支持新选项
			mode = retp_compiler() ? SPECTRE_V2_RETPOLINE_GENERIC :
						 SPECTRE_V2_RETPOLINE_MINIMAL;
			setup_force_cpu_cap(X86_FEATURE_RETPOLINE);
		}
		// 设置最终mode
		spectre_v2_enabled = mode;
		pr_info("%s\n", spectre_v2_strings[mode]);

		/*
		 * If neither SMEP nor PTI are available, there is a risk of
		 * hitting userspace addresses in the RSB after a context switch
		 * from a shallow call stack to a deeper one. To prevent this fill
		 * the entire RSB, even when using IBRS.
		 *
		 * Skylake era CPUs have a separate issue with *underflow* of the
		 * RSB, when they will predict 'ret' targets from the generic BTB.
		 * The proper mitigation for this is IBRS. If IBRS is not supported
		 * or deactivated in favour of retpolines the RSB fill on context
		 * switch is required.
		 */
		if ((!boot_cpu_has(X86_FEATURE_PTI) &&
		     !boot_cpu_has(X86_FEATURE_SMEP)) || is_skylake_era()) {
			setup_force_cpu_cap(X86_FEATURE_RSB_CTXSW);
			pr_info("Mitigation: Filling RSB on context switch\n");
		}

		/* Initialize Indirect Branch Prediction Barrier if supported */
		// 如果CPU支持IBPU，这设置相关feature
		if (boot_cpu_has(X86_FEATURE_IBPB)) {
			setup_force_cpu_cap(X86_FEATURE_USE_IBPB);
			pr_info("Mitigation: Enabling Indirect Branch Prediction Barrier\n");
		}

		/*
		 * Retpoline means the kernel is safe because it has no indirect
		 * branches. But firmware isn't, so use IBRS to protect that.
		 */
		// 针对firmware设置IBRS，仅在firmware中使用。内核中优先使用retpoline
		if (boot_cpu_has(X86_FEATURE_IBRS)) {
			setup_force_cpu_cap(X86_FEATURE_USE_IBRS_FW);
			pr_info("Enabling Restricted Speculation for firmware calls\n");
		}
	}

从代码可以看出，内核启动时传入的参数并非直接对应最终的mode，之间有一定的“换算”关系，先看看入参类型的定义：

	/* The kernel command line selection */
	enum spectre_v2_mitigation_cmd {
		SPECTRE_V2_CMD_NONE,
		SPECTRE_V2_CMD_AUTO,
		SPECTRE_V2_CMD_FORCE,
		SPECTRE_V2_CMD_RETPOLINE,
		SPECTRE_V2_CMD_RETPOLINE_GENERIC,
		SPECTRE_V2_CMD_RETPOLINE_AMD,
	};
	
	static const struct {
		const char *option;
		enum spectre_v2_mitigation_cmd cmd;
		bool secure;
	} mitigation_options[] = {
		{ "off",               SPECTRE_V2_CMD_NONE,              false },
		{ "on",                SPECTRE_V2_CMD_FORCE,             true },
		{ "retpoline",         SPECTRE_V2_CMD_RETPOLINE,         false },
		{ "retpoline,amd",     SPECTRE_V2_CMD_RETPOLINE_AMD,     false },
		{ "retpoline,generic", SPECTRE_V2_CMD_RETPOLINE_GENERIC, false },
		{ "auto",              SPECTRE_V2_CMD_AUTO,              false },
	};

这些分别对应了上述内核文档中说明的参数。共有6种情况。

再看看mode的定义：

	/* The Spectre V2 mitigation variants */
	enum spectre_v2_mitigation {
		SPECTRE_V2_NONE,
		SPECTRE_V2_RETPOLINE_MINIMAL,
		SPECTRE_V2_RETPOLINE_MINIMAL_AMD,
		SPECTRE_V2_RETPOLINE_GENERIC,
		SPECTRE_V2_RETPOLINE_AMD,
		SPECTRE_V2_IBRS,
	};

mode也有6种情况，但他们之间却不是一一对应关系。根据前面的代码分析，我们可以总结下他们之间的对应关系。

# 参数与mode的对应关系

1. off --> SPECTRE_V2_NONE

2. on，force，retpoline，这3个参数，从文档上看，看似各不相同，但是从代码实现上看，都是一样的。最终对应哪个mode，分5种情况：
	  - if (!CONFIG_RETPOLINE) --> SPECTRE_V2_NONE
	  - if (AMD CPU && 支持lfence && 新GCC) -->  SPECTRE_V2_RETPOLINE_AMD
	  - if (AMD CPU && 支持lfence && 旧GCC) -->  SPECTRE_V2_RETPOLINE_MINIMAL_AMD
	  - if ((INTEL CPU || 不支持lfence) && 新GCC) -->  SPECTRE_V2_RETPOLINE_GENERIC
	  - if ((INTEL CPU || 不支持lfence) && 旧GCC) -->  SPECTRE_V2_RETPOLINE_MINIMAL
  
3. retpoline,generic，分2种情况
	  - if (新GCC) -->  SPECTRE_V2_RETPOLINE_GENERIC
	  - if (旧GCC) -->  SPECTRE_V2_RETPOLINE_MINIMAL
  
4. retpoline,amd，分4种情况：
	  - if (支持lfence && 新GCC) -->  SPECTRE_V2_RETPOLINE_AMD
	  - if (支持lfence && 旧GCC) -->  SPECTRE_V2_RETPOLINE_MINIMAL_AMD
	  - if (不支持lfence && 新GCC) -->  SPECTRE_V2_RETPOLINE_GENERIC
	  - if (不支持lfence && 旧GCC) -->  SPECTRE_V2_RETPOLINE_MINIMAL

这部分逻辑其实比较混乱，有不少的优化空间～