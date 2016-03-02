---
layout: post
title:  "闲聊synchronous external abort"
date:   2016-03-01 6:15:32
author: HappySeeker
categories: Kernel
---

# 闲话

最近环境中又出现了synchronous external abort，ARM64架构环境，不得不感叹，X86架构看似稳定很多。自己对ARM架构没有深入研究，顺便了解了相关的内容，内容不多，记录存档。

# 什么synchronous external abort？

读了下ARM手册中的相关内容，对ARM架构了解有限，这里仅谈自己的理解，不对之处望指正！

这里面有3个概念：synchronous、external和aborts，需要分开说。

## Aborts

先说Aborts，熟悉X86架构的TX应该对这个也不陌生，其实就是一种异常。 ARM中包括多种异常类型，比如： 中断(也称为异步异常)、Aborts、Reset、Exception generating instructions(如SVC、HVC、SMC指令)。

所谓异常，从直观上，可以理解为硬件检测到了“something wrong”，或者外设有事件需要上报，需要上报给CPU。可以理解为一种硬件上的一种处理机制。

Aborts可以由两种方式触发：

1. 外部存储系统(external memory system,可以理解为内存硬件)触发。通常在进行指令预取(prefetch aborts)或数据访问(data aborts)时发现异常后触发，表明指定不是一个真实的可以访问的内存地址。

2. MMU(Memory Management Unit)触发。这就是我们最常见的缺页异常了，当指定的虚拟地址在页表中找不到相应的页表项时触发，内核中采用该缺页异常来分配新的物理内存。

## external(外部)

External Aborts即External memory error，即发生外部存储系统(external memory system)上的aborts异常，即上述Aborts触发方式1。

External Aborts被认为是非常少见的，且对应用程序是致命的。不可纠正的(通常内存都具有一位ECC错误纠正功能)ECC奇偶校验错误就是一种典型的External Aborts。

External Aborts可能发生在如下几种情况下：

1. 指令预取。此时其为“precise”(准确的)，即Instruction Fault Status Register(IFSR寄存器)中保存了准确的异常信息，R14_abt(abort模式下的LR寄存器)保存了异常地址信息。
2. 数据读写。此时可能是“imprecise”(非准确的)，即R14_abt(abort模式下的LR寄存器)保存地址信息可能不准确。
3. hardware page table walk。此时其为“precise”(准确的)。

## synchronous(同步)

Synchronous Aborts表示异常在指令(或指令流程)执行时产生，发生异常时返回的地址信息能提供详细的导致异常的指令信息。也就是说，这种情况下，返回的地址能用于分析定位故障。意味着，如果你遇到了这种abort，你还是幸运的，因为有相关的信息用于分析定位问题。

相反，Asynchronous Aborts表示异常不是在指令(或指令流程)执行时产生的，其返回的地址信息与异常没有什么关系，不能提供异常信息，也就不能用于分析定位故障，意味着，如果遇到这种abort，你就基本无计可施了。(不确定硬件层面是否有啥定位手段)

综合来看，synchronous external abort表示：由外部存储系统触发的，有异常地址信息可用的异常。个人理解，这个更可能是硬件问题导致，软件层面暂时想不到有什么原因会导致这样的问题。

对于目前遇到的问题，需要利用kdump搜集vmcore，然后再深入分析了。后面有机会再聊。






