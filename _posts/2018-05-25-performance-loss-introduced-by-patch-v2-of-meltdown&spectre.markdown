---
layout: post
title:  "Meltdown & Spectre patch v2-关闭开关后仍有40%的性能差异？"
date:   2018-05-25 16:45:25
author: JiangBiao
categories: Kernel
---

#  问题

在给环境中打上了针对Meltdown&Spectre漏洞的V2版本补丁后，同时通过grub参数关闭掉所有的开关，通过unixbench工具测试系统调用，发现性能损耗达40%+？开始不相信，原因很简单，权当记笔记了。

# V2版本patch

针对Meltdown&Spectre漏洞的V2版本patch，其实就是引入retpoline后的patch。主要针对Spectre V2漏洞，之前的V1版本使用的是微码(固件)+IBRS+IBPB的方式来规避的，这种方式有2个大问题：

1、IBRS带来的性能损耗太大，实测超过20%。难以接受。
2、需要更新固件(微码)，操作实在不便。如果涉及到Guest中打补丁，还需要同时对Qemu打补丁，更是麻烦。

retpoline又是Google的大师提出的，具体原理有空单独写文章说明，主要思想是：利用编译器的新特性，在编译时在关键代码处加入不会执行的死循环，如此通过结合编译器和CPU，可以防止CPU在这些节点进行Speculation操作，从而规避掉Spectre V2漏洞。对于内核中的汇编代码，尤其是系统调用、中断、异常时使用的JMP、CALL指令，由于无法利用编译器，这类代码需要修改内核代码，对JMP和CALL进行封装，原理跟编译器中的新特性类似，如此可以规避掉内核中相关关键路径的Spectre漏洞(即使在没有升级编译器的情况下～)。

retopline引入后，就可以基本代替原有的IBRS功能(IBPB还需要，但是IBPB对性能影响比较小)。但对内核的性能影响比IBRS小很多，据官方数据：0.5-2%，好像有点太低了。同时对固件的依赖更小了(只有IBPB还依赖固件)。

# 测试数据

硬件环境：

	Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz 32核	

测试工具：

	Unixbench-5.1.2

测试方法：

	# ./Run syscall -c 1

## 补丁前

在未打补丁前，原生内核。结果

	Benchmark Run: Fri May 25 2018 15:16:55 - 15:19:05
	32 CPUs in system; running 1 parallel copy of tests

	System Call Overhead                        1552369.8 lps   (10.0 s, 7 samples)

	System Benchmarks Partial Index              BASELINE       RESULT    INDEX
	System Call Overhead                          15000.0    1552369.8   1034.9
		                                                           ========
	System Benchmarks Index Score (Partial Only)                         1034.9

## 补丁后

启动参数：

	nopti noibrs noibpb

结果：

	System Call Overhead                        1017376.4 lps   (10.0 s, 7 samples)

	System Benchmarks Partial Index              BASELINE       RESULT    INDEX
	System Call Overhead                          15000.0    1017376.4    678.3
		                                                           ========
	System Benchmarks Index Score (Partial Only)                          678.3
	
# 原因

很简单，内核启动参数加错了。

V1版本关闭所有功能的内核启动参数为：

	nopti noibrs noibpb

但V2由于引入了retpoline，启动参数加错了，关闭所有功能的参数应该为：

	nopti nospectre_v2

或者

	nopti spectre_v2=off
	
参数设置正确后，性能差距在5%左右，为啥还有？

因为上游的补丁中有些汇编代码没有通过开关控制，还有优化空间。
