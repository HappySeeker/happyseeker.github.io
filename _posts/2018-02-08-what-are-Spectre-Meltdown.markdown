---
layout: post
title:  "CPU漏洞-Spectre&Meltdown原理分析及修复原理"
date:   2018-02-08 16:16:34
author: JiangBiao
categories: Kernel
---

#  漏洞

席卷全球的CPU漏洞Spectre&Meltdown让全世界的攻城师们着实忙碌了一把，我们也不例外，相关漏洞有3个：

1. Spectre varant1 CVE-2017-5753：bound-check bypass
2. Spectre varant2 CVE-2017-5715：Branch Target Injection
3. Meltdown CVE-2017-5754：Rogue Data Cache Load

3个漏洞的基础都来自于现代CPU的Specutlation特性，即预测执行，这也是我们熟悉的概念“乱序执行”、“指令预取”、“分支预测”的基础。共性：

1. 均基于现代CPU提供的预测执行功能
2. 需要能在主机中运行恶意程序，不能远程攻击。
3. 只能窃取数据，不能修改数据。

# 漏洞原理及影响

## Spectre varant1 CVE-2017-5753：bound-check bypass

### 基本原理

由于CPU的预测执行，在有条件分支(Conditional branch)的场景中(比如if语句)，可能会在条件分支判断操作执行之前(或者同时)执行(或预取)分支后的数据(可能是未授权的内核隐私数据)，预取的数据会在CPU Cache中存在一段时间，此时，恶意攻击程序可能通过side-channel的攻击方式推算出Cache中的数据，如此可能导致非授权的关键数据的泄露。

### 理论场景

如果条件分支中检查的是数组(或其他数据结构)的边界，而恶意攻击者通过传入超过边界的值，此时，由于CPU的预测执行，可能会将超过数组边界的数据预取到Cache中，攻击者可以通过side-channel的方式窃取到相关数据。

示例代码1：

struct array {
 unsigned long length;
 unsigned char data[];
};
struct array *arr1 = ...;
unsigned long untrusted_offset_from_caller = ...;
if (untrusted_offset_from_caller < arr1->length) {
 unsigned char value = arr1->data[untrusted_offset_from_caller];
 ...
}

此时，如果arr1->length在Cache中没有缓存，则CPU可能会预取到data[]数组中的越界数据。但是，测试影响并不大，因为一旦条件判断语句得到执行，CPU将会立刻回滚(roll back)掉之前预取到Cache中的数据(因为条件不满足)，预取的数据在Cache中存在的时间应该非常短，风险很小。

但如下示例代码2中的场景就不同了。如果arr1->length、arr2->data[0x200] 和 arr2->data[0x300]在Cache中都没有缓存，而代码中其他的所有数据都缓存了，并且判断条件被CPU预测为true，此时，CPU可能会在条件判断执行之前，预先执行如下操作：
1. 加载数据value = arr1->data[untrusted_offset_from_caller]
2. 从arr2->data数组中加载对应索引的数据到 L1 cache，此处的索引依赖于从arr1中读取到的数据，即这两个操作有依赖关系。

struct array {
 unsigned long length;
 unsigned char data[];
};
struct array *arr1 = ...; /* small array */
struct array *arr2 = ...; /* array of size 0x400 */
/* >0x400 (OUT OF BOUNDS!) */
unsigned long untrusted_offset_from_caller = ...;
if (untrusted_offset_from_caller < arr1->length) {
 unsigned char value = arr1->data[untrusted_offset_from_caller];
 unsigned long index2 = ((value&1)*0x100)+0x200;
 if (index2 < arr2->length) {
   unsigned char value2 = arr2->data[index2];
 }
}

之后，当CPU执行条件判断时，发现条件不满足，于是会做相应的回滚和retire操作，但是已经加载到Cache中的arr2->data[index2]数据会依然存在。此时，通过side-channel攻击方式，即可推测出Cache中的数据值。比如，通过测量arr2->data[0x200] 和 arr2->data[0x300]的数据加载时间，即可确认arr1->data[untrusted_offset_from_caller]中的数据是0还是1(如果是0，则arr2->data[0x200] 存在与cache中，数据加载更快；反之，arr2->data[0x300]存在与Cache中，数据加载更快)

### 现实场景

1. Google其实并未在Linux kernel代码中找到相应的可以攻击的代码模型，从上面的示例看，条件还算蛮苛刻的。但由于内核支持eBPF和JIT，可以通过eBPF、JIT或类似的方式向内核中注入类似的恶意代码，通过这种方式，也可以窃取到内核中的敏感数据。
2. Linux社区中针对改漏洞的特征，在内核中找到了可能存在的风险的代码点，其具有如下特征：
  - 从用户态向内核态中传入参数
  - 内核中对传入的参数做检查(判断)，有条件分支。
  - 在条件分支中访问了内核中的数据
  对于具有上述特征的代码，被认定为风险代码，需要逐个处理。绝大多数类似存在与驱动中。

### 影响

通过利用该漏洞，可以绕过边界条件检查，窃取到越界的数据。内核场景中，用户可能通过从用户态传入非法参数窃取到内核中的隐私数据。

## Spectre varant2 CVE-2017-5715: Branch Target Injection
基本原理：利用CPU中的间接分支预测器(indirect branch predictors) 的预测执行操作进行攻击，原理类似，利用side-channel的方式窃取预测执行时预取到CPU Cahce中的数据。

理论场景：
该漏洞仅针对于间接分支指令(indirect branch instruction)，如jmp eax，这里“间接”的意思是jmp跳转的目的地址不是直接给出的，而是通过寄存器或者内存提供了，此时CPU无法预先知道跳转的目的地址，只能在运行时根据上下文得出。这样的指令在实际的场景中实在很多。

现实场景：
1、在Linux KVM主机中运行虚拟机，  在当虚拟机退出(VMExit)时，可能通过该漏洞窃取到Host主机内核态的隐私数据。
2、当同一台主机中的虚拟机发生切换的时候(即进程切换，当前CPU上运行的进程上下文发生切换)，被切换的虚拟机可能会窃取到新虚拟机中的隐私数据。
3、与2类似，同一台主机中运行的不同进程，可能会相互窃取到对方的隐私数据。

影响：
1、利用该漏洞可以突破内核态和用户态之间的边界，让用户态程序窃取到内核中的数据。
2、利用该漏洞可以突破进程地址空间之间的边界，让进程可以窃取其他进程的数据。而进程地址空间隔离恰好是Linux内核和硬件虚拟内存管理机制提供的最基本、最核心的保障。
3、在云计算(虚拟机)的场景中，该漏洞影响尤为明显。由于主机中运行的虚拟机不受控制(由用户使用，并决定运行的程序)，那么恶意用户完全可以利用自己的虚拟机窃取主机或者其他虚拟机中的隐私数据。相对而言，其他多数场景中，系统中运行的应用基本是可控的，风险较小。

## Meltdown CVE-2017-5754: Rogue Data Cache Load

基本原理：攻击方法是通过用户态应用程序直接访问内核中的私有数据，正常情况下，这会直接触发page fault，因为CPU会进行权限检查，ring3的程序显然不能访问ring0中的page。但是，在特定场景下，CPU可能进行预测执行，意味着访问数据的操作可能在权限检查之前执行，由于权限检查相对比较耗时的操作，而数据读取操作通常很快（尤其是在被访问的数据就在L1 Cache中时），这种情况下，数据读取和权限检查实际执行的时间差可能会达到数百个CPU cycle。这给攻击者通过side-channel方式攻击留下了比较充分的时间。
典型场景：
1、系统调用时，从用户态切换到内核态时，可能利用此漏洞。
2、在用户态下发生中断时，会从用户态切换到内核态，此时可能利用此漏洞
影响：
1、利用该漏洞，恶意用户态程序可以突破用户态和内核态边界，窃取内核隐私数据。
2、只能窃取在页表中已经映射的内核数据，不能窃取未映射的数据。这也是开源修复方案(kpti/kaiser)的基础。

# 内核修复原理
## Spectre varant1 CVE-2017-5753
针对该漏洞，修复方法比较简单直接。在存在风险的代码点，加上内存屏障即可(X86中对应为lfence指令)，内存屏障可以保证在相应的关键点，代码按顺序执行，防止乱序，本质上即限制了预测执行的操作。

 ## Spectre varant2 CVE-2017-5715
该漏洞相对比较麻烦，关键点在于：
1. 需要堵住从用户态到内核态切换时的数据泄露漏洞。
2. 需要堵住从进程上下文切换时的数据泄露漏洞。
3. 在虚拟化的场景中，需要同时堵住Guest和Host中的漏洞。
修复方式则主要针对上述3点，主要包括：
1、更新CPU固件/微码，提供分支预测限制功能。包括两种方式：IBRS和IBPB。
2、在用户态和内核态切换时，限制预测执行。
3、在进程上下文切换时，限制预测执行。
4、对qemu打补丁，通过qemu让guest能感知到CPU(固件)提供的新特性。如此，在Guest中部署Host中相同的内核补丁，即可在Guest中防止内核态和进程间的数据泄露。

总的来说，需要对Host和Guest的内核打补丁，同时还需要更新CPU的固件。

## Meltdown CVE-2017-5754
从前面的原理分析可知，该漏洞只能窃取在页表中已经映射的内核数据，不能窃取未映射的数据。而当前的Linux内核实现中，页表相关实现有如下特点：
1. 进程在内核态和用户态使用同一张页表，如此，在用户态和内核态间相互切换时，就不需要切换页表，效率高。
2. 所有进程共享内核地址空间。即当进程进入内核态后，使用的内表内容都是一样的。虽然页表内容相同，但是页表却是各自独立的。实现方式为：内核自己维护一个单独的页表，但并不直接使用，而是将该页表内容复制到所有的进程自己的页表中。但内核页表内容发生变化时，会在合适的时机同步到所有的进程页表中。
3. 内核地址空间中的大部分区域(除vmalloc区以外)都是在内核初始化时提前映射好的，即内核页表中的大部分内容是不会发生变化的，这部分内容已提前映射，不会发生page fault。

所以，在当前内核实现中，由于用户态和内核态使用同一张页表，同时内核地址空间大部分内容已提前映射，如此存在Meltdown的漏洞风险。

修复方式：将内核态和用户态使用的页表分离，各自使用自己独立的页表，在内核态和用户态相互切换时，同时切换相应的页表。这种修复方案的官方名称叫Kpti，其实相关方案早在16年就提出了，在单独的分支中维护，名称为KAISER。当前合入内核主分支后，更名为kpti。
页表分离的基本原理是：使用线性地址中的第12位来区分页表是内核态还是用户态，将PGD(页目录)由原来的一页增加为两页，分别对应内核态和用户态的页目录。
