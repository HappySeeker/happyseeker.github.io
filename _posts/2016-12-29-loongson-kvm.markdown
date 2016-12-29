---
layout: post
title:  "闲聊龙芯KVM-内存虚拟化"
date:   2016-12-29 7:01:57
author: HappySeeker
categories: Kernel
---

最近分析龙芯KVM的实现，重点在内存虚拟化，影响性能的关键，本文主要讲述龙芯硬件平台上，现有的几种内存虚拟化方式的实现原理。

# 硬件现状

当前龙芯3A硬件上已经提供了部分虚拟化需要的硬件，比如Guest、host模式，VPID的支持，但是对于内存虚拟化来说，还没有类似X86上的EPT之类的硬件，所以内存虚拟化实现比较困难，性能不能充分发挥。

# 现有的可行方式

根据龙芯的几篇主要博士论文，目前龙芯KVM中内存虚拟化可行的实现方式有如下几种：

1. 软件模拟TLB(同构虚拟化)。一种全虚拟化方式，也是目前Mips采用的标准软件虚拟化方式，不需要修改Guest OS。所谓同构，指Guest OS和Host OS使用相同的内存管理方式，即都基于结构化的TLB。这种方式完全通过软件模拟Guest OS中使用的TLB，Guest OS中所有与TLB相关的操作和异常都将导致VM-Exit。性能比较差。
2. 异构内存虚拟化方法。半虚拟化方式，也是目前龙芯实现的一种主要方式。所谓异构，指Guest OS和Host OS使用不同的内存管理方式，主要是通过修改Guest OS的内核，去除所有TLB相关的操作，让Guest OS不再直接使用TLB，即让Guest OS基于机构化页表方式；而Host OS必须基于结构化TLB(硬件决定)，所以称为异构。这种方式消除了Guest OS因操作TLB导致的VM-Exit。而且由VMM统一管理TLB，效率比较高，整体来说性能比较好。论文测试数据能达到物理机的85%以上，已经很不错了。但这种方式，对Guest OS需要做比较大的改动，局限性也很明显。
3. 硬件双TLB方式。全虚拟化方式，也是Mips虚拟化标准中推荐的标准硬件虚拟化方式，不需要修改Guest OS。主要是通过在硬件上增加一个TLB，供Guest OS使用(类似于X86的EPT)。但目前龙芯硬件尚未实现，由于实现难度和风险都比较大，还在评估中，从论文中的模拟和描述看，龙芯当前认为这种方式性能并不好，还不及异构内存虚拟化方式，但实际并不确认。性能不好由于额外的如下开销：增加Guest TLB带来的访存延迟、增加Guest TLB带来的额外的TLB缺失异常开销、VMM维护GPA到HPA映射表的开销。
4. 基于影子缓冲的内存虚拟化方法。全虚拟化方法，是一篇论文中推荐的方法，需要对硬件做部分改造(但工作量和风险不及双TLB方式)，同时软件也需要同步实现。其目的是尽量减少TLB相关的异常(将导致VM-Exit)，据论文中的数据，其性能略低于异构内存虚拟化方法。

总的来说，方式1和3是Mips使用的标准的软件和硬件虚拟化方式，都是全虚拟化。

方式2和4是龙芯当前(或推荐)使用的方式，跟Mips标准不一样。方式2是龙芯当前实现的方案，测试性能最优。

# 同构虚拟化原理

该方式定义了3中TLB：

- Guest TLB。有Guest OS维护，用于映射GVA->GPA，该TLB不会被Guest OS直接使用，只用于生成Shadow TLB。Guest OS 在维护Guest TLB时,对 TLB 的读写指令会被虚拟机管理器(virtual machine monitor, VMM)捕捉并模拟,使 Guest OS 认为成功读写了硬件 TLB。
- Shadow TLB。用于映射GVA->HPA，即可以直接被硬件TLB使用的格式。由VMM维护。
- Host TLB。用于映射HVA->HPA，由Host OS维护(KVM中，VMM运行于Host OS的内核态)，是唯一的真正的硬件TLB。

Guest最终通过Host TLB装载Shadow TLB中的条目实现GVA到HPA的转换。

Guest中，任何TLB相关的操作都将触发异常。

Guest OS做访存操作时有如下集中情况发生：

1. 硬件TLB中刚好有该GVA到HPA的映射，此时不会发生异常，硬件TLB直接翻译地址，正常访问。
2. 如果硬件TLB中不存在相应映射条目，触发TLB Miss异常。此时退出到VMM中进行处理。
   - VMM先遍历Guest TLB，如果找到相应条目，则将Guest TLB中翻译后的GPA转换为HPA，然后填入Shadow TLB、填入硬件TLB，并返回Guest OS继续运行。
   - 如果Guest TLB中没有找到，则向Guest OS中注入TLB Miss异常，然后由Guest OS负责重填Guest TLB(当然这其中可能涉及更复杂的过程，比如当Guest OS的页表中也没有相应条目时(缺页)的处理，可能会再次触发异常，这里不讨论)。当Guest重填TLB时，会调用类似TLBWR之类的之令，此时会再次触发VM-Exit，VMM通过捕获该异常，其中填入Guest TLB、相应的Shadow TLB条目和硬件TLB，然后返回Guest OS继续执行。(这个过程比较复杂，性能瓶颈就在这里了)

# 异构虚拟化原理

主要思想是：通过修改Guest OS内核，对Guest OS隐藏MMU(TLB)硬件结构，实现结构化页表，消除对Guest TLB的维护开销，加速TLB miss处理过程，相比方式1，性能显著提高。

该方式只维护一个TLB，即物理TLB，由Host和Guest共享使用。同时Guest OS中去除所有TLB操作，只有页表，而且页表中保存的直接是GVA->HPA的映射(显然需要对Guest做比较大的改动)，这种情况下，当Guest OS做访问操作触发TLB miss时，由VMM捕获，然后直接读取Guest页表(需要VMM能访问Guest页表)中的GVA->HPA的映射条目，填入物理TLB中，如此VMM中的TLB miss实现简单，处理快。

一种改进的实现方式为：将Guest的页表分为两种：VMM维护的内核态页表(用于映射内核态地址)和Guest维护的用户态页表(用户映射用户态地址)。当Guest中发生TLB miss时，VMM判断地址类型，如果是内核态地址，则直接查询VMM维护的内核态页表；如果是用户态地址，则查询Guest内部的页表。当Guest发生page fault(即TLB Load异常)时，VMM判断地址类型，如果时内核态地址，直接在VMM处理(分配内存，修改内核态页表)；如果是用户态地址，则将缺页异常注入Guest，由Guest自己完成处理(分配内存，修改Guest页表)。

这个方案看似性能很不错，但最大的问题在于Guest和Host的地址空间隔离问题，即Guest的虚拟地址很可能与Host中的冲突(即都有相同的地址)，而TLB只有一个，很可能导致混乱，目前的解决方式是：在Guest切换、Guest和Host切换时都做TLB全部flush操作，每次即切换后TLB都是空的，重新开始填，这样就不会冲突了，但性能损失是不可忽略的。论文中提到了多种改进方法，但是都难以完全避免这种情况。

# 硬件双TLB原理

这种方式类似于X86中的EPT。基本原理为：

硬件上新增一套TLB(Guest TLB)，供Guest专用，用于GVA->GPA转换，而原有的硬件TLB(Host TLB)用于：

- Host的HVA->HPA的转换
- Guest的GPA->HPA的转换

这种情况下，Guest中GVA地址可以直接通过两级硬件TLB转换为HPA，最大的优势是，能大大减少因TLB miss而VM-Exit的次数。
但由于新增了一级TLB，会导致更多的TLB miss。

# 基于影子缓冲的内存虚拟化原理

综合了已有的异构内存虚拟化的特点，并考虑尽量降低硬件改造难度，目的在于尽可能的减少TLB相关的异常。方法的主要原理如下：

硬件设计(当前具体硬件状态如何？):

1. Guest和Host共享物理TLB，但TLB相关异常在不同状态下处理:TLB缺失异常在Root Mode下处理；TLB其他异常在Guest Mode下处理,例如TLB load/store/modify异常等。
2. Guest可以任意修改除了VPID相关域之外的TLB相关控制寄存器,同时可以使用TLBWR、TLBWI和TLBP之外的指令访问TLB。虚拟机任意写TLB的行为和TLBP的行为都产生异常。

软件设计:

1. VMM凭借VPID和ASID缓存所有曾经填入TLB的表项,构建影子缓冲。
2. 处理Guest执行TLBW相关指令产生异常时,更新或无效影子缓冲表项。
3. 处理Guest执行TLBP指令产生异常时,查询影子缓冲。
4. 当发生Guest将TLB全部无效时,无效该Guest所有影子缓冲。
5. TLB缺失异常时,查询影子缓冲:如果是unmapped地址且缓冲地址无效,则构建unmapped地址的表项;否则直接将GVA->HPA映射填
入TLB。(这里不更新shadow TLB，Shadow TLB在Guest执行TLBW相关指令时通过异常更新)
