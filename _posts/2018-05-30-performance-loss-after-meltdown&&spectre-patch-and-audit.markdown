---
layout: post
title:  "合入Meltdown && Spectre patch V2补丁后莫名的性能差异"
date:   2018-05-30 7:17:09
author: JiangBiao
categories: Kernel
---

#  背景

最近测试Meltdown && Spectre V2 patch对虚拟机的性能影响，遇到一个莫名的性能问题，问题比较简单，但需要记录下。

# 现象

在合入补丁的Host环境中，加入内核启动参数：nopti nospectre_v2，也就是关闭了所有的补丁功能。测试系统调用性能。结果为：1020.

在合入补丁的Guest环境中，也加入内核启动参数：nopti nospectre_v2，测试系统调用性能，结果为：964，Guest的系统调用性能于Host差距在5%以内，正常。

在未合入补丁的另一个标准发行版的Guest环境中，用相同的方法测试系统调用性能，结果：该Guest中的系统调用性能为2020，其性能几乎是Host的两倍。这个结果显然不太正常，Guest的系统调用性能居然比同样物理硬件环境下的Host性能高出100%。


# 分析

分析过程也比较简单直接。

## 确认环境

通过/proc/cmd、/sys其他接口确认了主机中的参数，确认已经生效，未发现疑点。

确定两台虚拟机的相关参数(CPU数、mem等)，确认一致，未发现疑点。

## perf

通过perf top，对比两台虚拟机在执行性能测试时热点，发现：

合入补丁的Guest热点如下：

	  PerfTop:    3972 irqs/sec  kernel:91.4%  exact:  0.0% [4000Hz cycles],  (all, 32 CPUs)
	-------------------------------------------------------------------------------

	    40.67%  [kernel]       [k] audit_filter_syscall
	     7.06%  [kernel]       [k] audit_filter_rules.isra.8
	     6.40%  [kernel]       [k] system_call
	     4.38%  [kernel]       [k] system_call_after_swapgs
	     3.83%  [kernel]       [k] __audit_syscall_exit
	     3.42%  [kernel]       [k] __audit_syscall_entry
	     3.07%  [kernel]       [k] _raw_qspin_lock

未合入补丁的Guest热点如下：

	   PerfTop:    4156 irqs/sec  kernel:83.6%  exact:  0.0% [4000Hz cycles],  (all, 32 CPUs)
	-------------------------------------------------------------------------------

	    13.38%  [kernel]       [k] system_call
	    10.30%  [kernel]       [k] system_call_after_swapgs
	     9.49%  [kernel]       [k] __audit_syscall_entry
	     7.80%  [kernel]       [k] __audit_syscall_exit
	     6.28%  [kernel]       [k] _raw_qspin_lock

很明显，最大的区别在于audit_filter_syscall和audit_filter_rules，也就是说性能较低的虚拟机中有多余的audit_filter_syscall调用，显然与audit相关，后面看看相关代码：

## 代码分析

audit_filter_syscall调用于audit_take_context函数中：

	/* Transfer the audit context pointer to the caller, clearing it in the tsk's struct */
	static inline struct audit_context *audit_take_context(struct task_struct *tsk,
							      int return_valid,
							      long return_code)
	{
		...
		/* 
		 * 调用audit_filter_syscall的条件：1. context->in_syscall，在__audit_syscall_entry中设置
		 * 条件1通常是满足的，因为syscall entry时就会设置这个变量，而这里的流程是在syscall exit中调用的
		 * 2. context->dummy，也在__audit_syscall_entry中设置，意思是是否存在audit rules，见后面分析
		 * 所以，就说当存在audit规则时，就会调用audit_filter_syscall。
		 */
		if (context->in_syscall && !context->dummy) {
			audit_filter_syscall(tsk, context, &audit_filter_list[AUDIT_FILTER_EXIT]);
			audit_filter_inodes(tsk, context);
		}

	}

audit_take_context的调用链：
	
	__audit_syscall_exit
	  audit_take_context
	    audit_filter_syscall

再看看__audit_syscall_entry的关键点：

	void __audit_syscall_entry(int major, unsigned long a1, unsigned long a2,
				   unsigned long a3, unsigned long a4)
	{
		...
		// 设置dummy，意思是当audit_n_rules不存在时，设置dummy
		context->dummy = !audit_n_rules;
		...
		// 设置in_syscall ，表示当前再syscall的过程中
		context->in_syscall = 1;
	}
	
继续看看audit_n_rules的设置代码(audit_add_rule)：

	/* Add rule to given filterlist if not a duplicate. */
	static inline int audit_add_rule(struct audit_entry *entry)
	{
		...
		#ifdef CONFIG_AUDITSYSCALL
		if (!dont_count)
			audit_n_rules++;
		...
	}

其调用链为：

	audit_receive_msg
	  audit_rule_change
	    audit_add_rule

# 原因

从上述代码可以看出，当系统中存在audit规则时，就会调用audit_filter_syscall。

通过对比两台虚拟机中的audit规则配置：/etc/auditd/audit.rules文件，确认确实在性能较低的环境中存在audit规则，而另一台虚拟机中不存在。

# 解决

audit确实在一些场景中对系统性能有一定影响，之前也遇到过多次类似的案例，如下几种方法均可控制audit开关：

1. 执行命令 auditctl -D，可以临时清空所有的audit规则，立即生效，重启后失效，但是简单直接。使用这种方法清除规则后，即发现两台虚拟机系统调用性能一致了。

2. 删除/etc/auditd/audit.rules文件中的所有内容。重启后生效，永久有效。

3. 关闭auditd服务，systemctl stop auditd。 

4. 加入内核启动参数中加入audit=0，如此可以彻底关闭内核中的audit功能，那么用户态中的啥规则都没有效了。这种方法是关闭audit的最根本的方法。上述1、2、3方法都只能关闭用户态的auditd部分功能，即使关闭，内核中的audit仍在运行，这种情况下，上述__audit_syscall_entry和__audit_syscall_exit 热点依然会存在，依然对性能有一定影响(应该很小)，方法4是one for all的效果最好的方法。