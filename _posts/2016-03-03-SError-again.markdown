---
layout: post
title:  "又见异常：SError(System Error)"
date:   2016-03-03 18:35:01
author: HappySeeker
categories: Kernel
---

# 闲话

前面介绍了Arm64环境中的System Error，这篇文章讲一下我遇到的具体的SError及相关分析。

# 问题现象

这次故障出现在用户态，出现时的打印如下：

	[ 1259.654597] Bad mode in Error handler detected, code 0xbf000002 -- SError
	[ 1259.661357] CPU: 12 PID: 2293 Comm: mate-settings-d Not tainted 4.1.15-1.el7.aarch64 #2
	[ 1259.669320] Hardware name: xxxx
	[ 1259.675209] task: ffffffc8c9bd1700 ti: ffffffc8c9e4c000 task.ti: ffffffc8c9e4c000
	[ 1259.682665] PC is at 0x7f9c51abd8
	[ 1259.685961] LR is at 0x7f942f9828
	[ 1259.689259] pc : [<0000007f9c51abd8>] lr : [<0000007f942f9828>] pstate: 80000000
	[ 1259.696616] sp : ffffffc8c9e4fff0
	[ 1259.699913] x29: 0000007fc5ef6be0 x28: 0000000000000000
	[ 1259.705220] x27: 0000007f8dfc92c0 x26: 0000007f8de63000
	[ 1259.710525] x25: 0000007f8de47e48 x24: 0000007f8de47e40
	[ 1259.715832] x23: 0000007fc5ef6d60 x22: 0000007f7b7ff000
	[ 1259.721137] x21: 0000000004001027 x20: 0000007f94344000
	[ 1259.726441] x19: 0000000013e20a10 x18: 0000000000000004
	[ 1259.731746] x17: 0000007f9c51abd0 x16: 0000007f943448c8
	[ 1259.737050] x15: 0000000000000028 x14: 0000000000000000
	[ 1259.742355] x13: 0000000000000004 x12: 0000000000000010
	[ 1259.747659] x11: 0101010101010101 x10: 0000000013f52280
	[ 1259.752965] x9 : 0000007f9c5c1570 x8 : 00000000000000d7
	[ 1259.758269] x7 : 0000007fc5ef6b78 x6 : 0000000000000000
	[ 1259.763574] x5 : 0000000000000000 x4 : 0000000000000001
	[ 1259.768878] x3 : 0000007f9c5bf120 x2 : 0000000000020000
	[ 1259.774184] x1 : 0000000004001000 x0 : 0000000000000000


# 分析

分析错误打印：
	Bad mode in Error handler detected, code 0xbf000002 -- SError

可以知道，发生了SError(也就是System Error异常)，错误码(ESR寄存器内容)为：0xbf000002

具体分析错误码：

- 前六位bit[31:26]为101111，对应具体的异常类型，查看ArmV8手册：

	101111 SError interrupt
	
	即系统异常中断。与打印一致。


再看看code后面bit的含义：

- ISV, bit [24]
Instruction syndrome valid. Indicates whether the rest of the syndrome information in this register
is valid.
0 No valid instruction syndrome. ISS[23:0] are RES0.
1 ISS[23:0] hold a valid instruction syndrome.

	本code中，该位为1，说明bit[23:0]中存放了instruction syndrome(出错指令的具体信息)

- IS, bits [23:0]
IMPLEMENTATION DEFINED syndrome information that can be used to provide additional
information about the SError interrupt. Only valid if bit[24] of this register is 1. If bit[24] is 0, this
field is RES0.

	表示IMPLEMENTATION DEFINED(即各硬件厂商根据自己的实现定义)的异常信息，也就是说这部分信息需要具体的硬件厂商来翻译，或者需要根据具体的芯片手册来。我的环境中暂时无法确认。