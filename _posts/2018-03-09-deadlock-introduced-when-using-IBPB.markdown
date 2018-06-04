---
layout: post
title:  "Spectre V2漏洞-IBPB-死锁"
date:   2018-03-06 15:10:36
author: JiangBiao
categories: Kernel
---

#  背景

基于CPU硬件的Spectre漏洞影响范围极大，市面上几乎所有的现代CPU都未能幸免。之前也写过文章讲述相关原理和修复方式了。其中的Spectre V2(CVE-2017-5713)影响最大，而且修复最麻烦，需要同时修复内核、qemu并更新CPU硬件微码。

redhat官方也发布了相应的修复补丁集。但合入补丁后的内核在云测试环境中出现了死锁问题，现象为：

内核trigger了nmi_watchdog，有如下打印

	[ 850.136888] <NMI> [<ffffffff8163f2ba>] dump_stack+0x19/0x1b
	[ 850.143493] [<ffffffff81638b26>] panic+0xd8/0x1e7
	[ 850.148970] [<ffffffff8111f260>] ? restart_watchdog_hrtimer+0x50/0x50
	[ 850.156449] [<ffffffff8111f322>] watchdog_overflow_callback+0xc2/0xd0
	[ 850.163902] [<ffffffff81162b11>] __perf_event_overflow+0xa1/0x250
	[ 850.170982] [<ffffffff811635e4>] perf_event_overflow+0x14/0x20
	[ 850.177743] [<ffffffff81033888>] intel_pmu_handle_irq+0x1e8/0x470
	[ 850.184795] [<ffffffff812fa291>] ? ioremap_page_range+0x241/0x320
	[ 850.191849] [<ffffffff811a99c1>] ? unmap_kernel_range_noflush+0x11/0x20
	[ 850.199506] [<ffffffff8139c9f4>] ? ghes_copy_tofrom_phys+0x124/0x210
	[ 850.206865] [<ffffffff8139cb80>] ? ghes_read_estatus+0xa0/0x190
	[ 850.213731] [<ffffffff8164a4bb>] perf_event_nmi_handler+0x2b/0x50
	[ 850.220786] [<ffffffff81649c09>] nmi_handle.isra.0+0x69/0xb0
	[ 850.227352] [<ffffffff81649d20>] do_nmi+0xd0/0x340
	[ 850.232921] [<ffffffff81648ff9>] end_repeat_nmi+0x1e/0x7e
	[ 850.239185] [<ffffffff81646d72>] ? _raw_spin_lock+0x32/0x50
	[ 850.245668] [<ffffffff81646d72>] ? _raw_spin_lock+0x32/0x50
	[ 850.252130] [<ffffffff81646d72>] ? _raw_spin_lock+0x32/0x50
	[ 850.258589] <<EOE>> [<ffffffff810bbdd2>] try_to_wake_up+0x192/0x300
	[ 850.265983] [<ffffffff810bbfb2>] default_wake_function+0x12/0x20
	[ 850.272943] [<ffffffff811f7563>] pollwake+0x73/0x90
	[ 850.278614] [<ffffffff810bbfa0>] ? wake_up_state+0x20/0x20
	[ 850.284981] [<ffffffff810b2658>] __wake_up_common+0x58/0x90
	[ 850.291449] [<ffffffff810b4404>] __wake_up_sync_key+0x44/0x60
	[ 850.298112] [<ffffffff811ebd73>] pipe_write+0x383/0x5b0
	[ 850.304185] [<ffffffff811e26ad>] do_sync_write+0x8d/0xd0
	[ 850.310356] [<ffffffff811e2ecd>] vfs_write+0xbd/0x1e0
	[ 850.316228] [<ffffffff8151bd85>] ? __sys_recvmsg+0x85/0x90
	[ 850.322591] [<ffffffff811e396f>] SyS_write+0x7f/0xe0
 
  从堆栈可以看出是阻塞在了try_to_wake_up流程中，需要获取调度运行队列的spinlock：rq->lock。而通过其他CPU上的

 从vmcore中可以看到其他进程也阻塞到了相同的锁中
 
	>crash> bt 16359
	> PID: 16359 TASK: ffff88043d29f300 CPU: 24 COMMAND: "qemu-kvm"
	> #0 [ffff88181ff08e70] crash_nmi_callback at ffffffff81046bf2
	> #1 [ffff88181ff08e80] nmi_handle at ffffffff81649c09
	> #2 [ffff88181ff08ec8] do_nmi at ffffffff81649d20
	> #3 [ffff88181ff08ef0] end_repeat_nmi at ffffffff81648ff9
	>[exception RIP: _raw_spin_lock+55]
	> RIP: ffffffff81646d77 RSP: ffff88044171f5e0 RFLAGS: 00000002
	> RAX: 0000000000002af8 RBX: ffff88047b092280 RCX: 000000000000ae8c
	> RDX: 000000000000ae8e RSI: 000000000000ae8e RDI: ffff88181ff16500
	> RBP: ffff88044171f5e0 R8: 0000000000000000 R9: 0000000000000000
	> R10: 0000000000000020 R11: ffff880479426220 R12: ffff88047b092a84
	> R13: 0000000000000046 R14: ffff88181ff16500 R15: 0000000000000018
	> ORIG_RAX: ffffffffffffffff CS: 0010 SS: 0018
	>— <NMI exception stack> —
	>#4 [ffff88044171f5e0] _raw_spin_lock at ffffffff81646d77
	> #5 [ffff88044171f5e8] try_to_wake_up at ffffffff810bbdd2
	> #6 [ffff88044171f630] default_wake_function at ffffffff810bbfb2
	> #7 [ffff88044171f640] autoremove_wake_function at ffffffff810aa148
	> #8 [ffff88044171f668] __wake_up_common at ffffffff810b2658
	> #9 [ffff88044171f6b0] __wake_up at ffffffff810b43a9
	> #10 [ffff88044171f6e8] __call_rcu_nocb_enqueue at ffffffff81127718
	> #11 [ffff88044171f700] __call_rcu at ffffffff81128548
	> #12 [ffff88044171f738] call_rcu_sched at ffffffff8112863d
	> #13 [ffff88044171f748] avc_node_delete at ffffffff8128d36b
	> #14 [ffff88044171f758] avc_alloc_node at ffffffff8163e2f5
	> #15 [ffff88044171f798] avc_compute_av at ffffffff8163e3b4
	> #16 [ffff88044171f7e8] avc_has_perm_noaudit at ffffffff8128d6d4
	> #17 [ffff88044171f830] cred_has_capability at ffffffff81290f0b
	> #18 [ffff88044171f8b8] selinux_capable at ffffffff81290fee
	> #19 [ffff88044171f8e0] security_capable_noaudit at ffffffff8128ac75
	> #20 [ffff88044171f8f0] has_ns_capability_noaudit at ffffffff8108b8f5
	> #21 [ffff88044171f900] ptrace_has_cap at ffffffff8108bbd5
	> #22 [ffff88044171f910] ___ptrace_may_access at ffffffff8108c661
	> #23 [ffff88044171f940] __schedule at ffffffff8164430e
	> #24 [ffff88044171f9c8] schedule at ffffffff81644ac9
	> #25 [ffff88044171f9d8] schedule_hrtimeout_range_clock at ffffffff81643ae2
	> #26 [ffff88044171fa70] schedule_hrtimeout_range at ffffffff81643b93
	> #27 [ffff88044171fa80] poll_schedule_timeout at ffffffff811f75d5
	> #28 [ffff88044171fab0] do_sys_poll at ffffffff811f8b5d
	> #29 [ffff88044171fed8] sys_ppoll at ffffffff811f8f63
	> #30 [ffff88044171ff50] system_call_fastpath at ffffffff816513fd

从堆栈可以看出是重复获取了同一把spinlock所：rq->lock，第1次获取锁位置在__schedule函数调用 context_switch前：

	raw_spin_lock_irq(&rq->lock);

# 原因

原因是在修复Spectre V2补丁时，redhat合入了如下补丁：

	From 26c260ea162852cb3961ff2411ac9604918dd723 Mon Sep 17 00:00:00 2001
	From: Josh Poimboeuf <jpoimboe@redhat.com>
	Date: Fri, 15 Dec 2017 23:41:06 +0100
	Subject: [x86] x86/mm: Only set IBPB when the new thread cannot ptrace current thread

	Message-id: <e5c83872cd0a161d3241cc2677443a18e69315ec.1513380150.git.jpoimboe@redhat.com>
	Patchwork-id: 6015
	O-Subject: [RHEL7.2.z PATCH CONFIDENTIAL 093/119] x86/mm: Only set IBPB when the new thread cannot ptrace current thread
	Bugzilla: 1526978
	CVE: CVE-2017-5715

	From: Tim Chen <tim.c.chen@linux.intel.com>

	To reduce overhead of setting IBPB, we only do that when
	the new thread cannot ptrace the current one.  If the new
	thread has ptrace capability on current thread, it is safe.

	Signed-off-by: Tim Chen <tim.c.chen@linux.intel.com>
	Signed-off-by: Andrea Arcangeli <aarcange@redhat.com>
	---
	 arch/x86/include/asm/mmu_context.h |  2 +-
	 arch/x86/include/asm/spec_ctrl.h   | 19 ++++++++++++++++++-
	 include/linux/ptrace.h             |  7 +++++++
	 kernel/ptrace.c                    | 27 +++++++++++++++++++++++----
	 4 files changed, 49 insertions(+), 6 deletions(-)

	Signed-off-by: Rado Vrbovsky <rvrbovsk@redhat.com>

	diff --git a/arch/x86/include/asm/mmu_context.h b/arch/x86/include/asm/mmu_context.h
	index 87d728c..52714b8 100644
	--- a/arch/x86/include/asm/mmu_context.h
	+++ b/arch/x86/include/asm/mmu_context.h
	@@ -48,7 +48,7 @@ static inline void switch_mm(struct mm_struct *prev, struct mm_struct *next,
	 #endif
		         cpumask_set_cpu(cpu, mm_cpumask(next));
	 
	-                spec_ctrl_ibpb();
	+                spec_ctrl_ibpb_if_different_creds(tsk);
	 
		         /* Re-load page tables */
		         load_cr3(next->pgd);
	diff --git a/arch/x86/include/asm/spec_ctrl.h b/arch/x86/include/asm/spec_ctrl.h
	index f7535cd..16245a8 100644
	--- a/arch/x86/include/asm/spec_ctrl.h
	+++ b/arch/x86/include/asm/spec_ctrl.h
	@@ -120,6 +120,7 @@
	 
	 #else /* __ASSEMBLY__ */
	 
	+#include <linux/ptrace.h>
	 #include <asm/microcode.h>
	 
	 extern void set_spec_ctrl_pcp_ibrs(bool enable);
	@@ -141,11 +142,27 @@ static inline void spec_ctrl_disable_ibrs(void)
		 }
	 }
	 
	+static inline void __spec_ctrl_ibpb(void)
	+{
	+        native_wrmsrl(MSR_IA32_PRED_CMD, FEATURE_SET_IBPB);
	+}
	+
	 static inline void spec_ctrl_ibpb(void)
	 {
		 if (static_cpu_has(X86_FEATURE_IBPB_SUPPORT)) {
		         if (__this_cpu_read(spec_ctrl_pcp) & SPEC_CTRL_PCP_IBPB)
	-                        native_wrmsrl(MSR_IA32_PRED_CMD, FEATURE_SET_IBPB);
	+                        __spec_ctrl_ibpb();
	+        }
	+}
	+
	+static inline void spec_ctrl_ibpb_if_different_creds(struct task_struct *next)
	+{
	+        struct task_struct *prev = current;
	+
	+        if (static_cpu_has(X86_FEATURE_IBPB_SUPPORT)) {
	+                if (__this_cpu_read(spec_ctrl_pcp) & SPEC_CTRL_PCP_IBPB && next &&
	+                    ___ptrace_may_access(next, NULL, prev, PTRACE_MODE_IBPB))
	+                        __spec_ctrl_ibpb();
		 }
	 }
	 
	diff --git a/include/linux/ptrace.h b/include/linux/ptrace.h
	index 822d877..16157f9 100644
	--- a/include/linux/ptrace.h
	+++ b/include/linux/ptrace.h
	@@ -55,8 +55,15 @@ extern void exit_ptrace(struct task_struct *tracer);
	 #define PTRACE_MODE_READ        0x01
	 #define PTRACE_MODE_ATTACH        0x02
	 #define PTRACE_MODE_NOAUDIT        0x04
	+#define PTRACE_MODE_NOACCESS_CHK 0x20
	+#define PTRACE_MODE_IBPB (PTRACE_MODE_ATTACH | PTRACE_MODE_NOAUDIT        \
	+                          | PTRACE_MODE_NOACCESS_CHK)
	 /* Returns true on success, false on denial. */
	 extern bool ptrace_may_access(struct task_struct *task, unsigned int mode);
	+extern int ___ptrace_may_access(struct task_struct *tracer,
	+                                const struct cred *cred, /* tracer cred */
	+                                struct task_struct *task,
	+                                unsigned int mode);
	 
	 static inline int ptrace_reparented(struct task_struct *child)
	 {
	diff --git a/kernel/ptrace.c b/kernel/ptrace.c
	index f0477bc..44f81d4 100644
	--- a/kernel/ptrace.c
	+++ b/kernel/ptrace.c
	@@ -222,9 +222,12 @@ static int ptrace_has_cap(struct user_namespace *ns, unsigned int mode)
	 }
	 
	 /* Returns 0 on success, -errno on denial. */
	-static int __ptrace_may_access(struct task_struct *task, unsigned int mode)
	+int ___ptrace_may_access(struct task_struct *tracer,
	+                         const struct cred *cred, /* tracer cred */
	+                         struct task_struct *task,
	+                         unsigned int mode)
	 {
	-        const struct cred *cred = current_cred(), *tcred;
	+        const struct cred *tcred;
	 
		 /* May we inspect the given task?
		  * This check is used both for attaching with ptrace
	@@ -236,9 +239,17 @@ static int __ptrace_may_access(struct task_struct *task, unsigned int mode)
		  */
		 int dumpable = 0;
		 /* Don't let security modules deny introspection */
	-        if (task == current)
	+        if (task == tracer)
		         return 0;
		 rcu_read_lock();
	+        if (!cred) {
	+                WARN_ON_ONCE(tracer == current);
	+                WARN_ON_ONCE(task != current);
	+                cred = __task_cred(tracer);
	+        } else {
	+                WARN_ON_ONCE(tracer != current);
	+                WARN_ON_ONCE(task == current);
	+        }
		 tcred = __task_cred(task);
		 if (uid_eq(cred->uid, tcred->euid) &&
		     uid_eq(cred->uid, tcred->suid) &&
	@@ -264,7 +275,15 @@ ok:
		 }
		 rcu_read_unlock();
	 
	-        return security_ptrace_access_check(task, mode);
	+        if (!(mode & PTRACE_MODE_NOACCESS_CHK))
	+                return security_ptrace_access_check(task, mode);
	+
	+        return 0;
	+}
	+
	+static int __ptrace_may_access(struct task_struct *task, unsigned int mode)
	+{
	+        return ___ptrace_may_access(current, current_cred(), task, mode);
	 }
	 
	 bool ptrace_may_access(struct task_struct *task, unsigned int mode)
	
关键就是在switch_mm()(上下文切换时都会调用)中加了一个条件(大概意思是只有在切换后的进程没有ptrace前一个进程的能力时，才加屏障）。目的是想减少在switch_mm中对IBPB屏障的使用。但是正是这个条件中可能会用到call_rcu()，这个call_rcu可能会去获取运行队列的锁，最终导致了死锁。显然这个补丁有问题，call_rcu()显然不能在schedule()时用，但是redhat一直没有修正。

# 相关原理

IBPB本质上是一个内存屏障，设置后在屏障后的指令就无法“预取”到屏障前了，这样可以防止spectre漏洞，于是改屏障自然的被放入了
switch_mm()中(上下文切换)，防止上下文切换时，前一个上下文“窃取”后一个的数据。

但问题是，这个屏障带来的整体代价非常“贵”，要多耗用5000-10000个cycle，而switch_mm的调用频率又实在太高(可以想象~)，所以这里带来的性能损耗估计会很大。

因此，有人提了这个补丁，目的是想减少在switch_mm中对IBPB屏障的使用，虽然这个补丁有问题，但是其方向是对的，确实在switch_mm中加IBPB的代价有点大，每次上下文切换都会用到。

对比了一下master kernel分支中的实现，也提了一个类似的补丁，但判断条件不一样，也是同一个人提的，从代码上看master分支中的条件看起来没有问题，是通过判断SUID_DUMP_USER标记，但这个补丁需要结合google提供的新方案retpoline来一起用。这样能最大程度的降低对性能的影响。

我们知道这次修复Spectre和Meltdown的补丁对OS的性能有非常大的影响，我想其中的很大一部分影响应该都是在switch_mm中使用IBPB“贡献”的。

关于retpoline又是一个新的话题，改天另写文章说明了。