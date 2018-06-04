---
layout: post
title:  "EPT misconfig"
date:   2018-05-29 6:32:05
author: JiangBiao
categories: Kernel
---

#  背景

前不久被问到EPT misconfig的问题，没反应过来，顺手了解了一下，记录一下。

# 基本概念

## EPT Voilation

这个是基础了，相当于EPT(扩展页表)的page fault，是一种特殊的硬件异常。当EPT中不存在指定GPA->HPA的映射时触发，硬件触发。

## EPT misconfig

本质上也是一种page fault，与EPT voilation不同(当page not present时触发)，EPT misconfig类似于 reserve bit set page fault，也就是说当页表项中的保留位设置时触发，也是硬件触发。

SDM中关于EPT misconfig的描述：

> An EPT misconfiguration occurs when, in the course of translating 
> a guest-physical address, the logical processor encounters an EPT 
> paging-structure entry that contains an unsupported value. An EPT 
> violation occurs when there is no EPT misconfiguration but the EPT 
> paging-structure entries disallow an access using the guest physical
> address.

# EPT misconfig用途

EPT misconfig通常用在处理mmio regions时，没有通过passed-through的方式透传给Guest的mmio区域通过EPT misconfig来处理。

当首次访问某mmio page时，会触发EPT violation，KVM在EPT violation的处理过程中设置相应EPT entry中的保留位，然后在下一次再访问该page时，即会触发EPT misconfig。
