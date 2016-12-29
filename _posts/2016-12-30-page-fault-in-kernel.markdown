---
layout: post
title:  "内核中的page fault & copy_from_user"
date:   2016-12-30 6:22:14
author: HappySeeker
categories: Kernel
---

# 内核态的page fault？

前段时间有同事问了个问题：内核中是否可能发生page fault？

一时没能给出准确答案，当即有种感觉：难道是对内核内存管理的理解还不够，之前在这方面还是比较自信的～

问题看似很简单，从之前的理解来看，以经典的32位X86为例，内核态低端地址都是线性映射的，页表都是事先(内核初始化时)创建好的，对于这段地址，应该不会发生page fault；但对于vmalloc区，是通过页表动态映射的，这部分显然是可能发生缺页的。

看似问题有答案了，但TX问的不是针对内核态地址发送的缺页，而是：在内核态下，对用户态地址的访问，是否发生page fault？

我追问了一句：什么情况下，内核态需要访问用户态地址？ 回答：比如copy_from_user.

问题清晰了，这种情况下，可能发生缺页吗？

# 硬件层面

从硬件层面上来说，当CPU做访问操作时，MMU硬件会遍历页表，查找匹配的映射条目，如果没有找到，就会自动触发page fault。

所以，从这个角度看，只要copy_from_user参数中的用户态地址对应的页表项不存在，就会产生page fault。

# 不应该发生吧？

换个角度，内核要使用copy_from_user从用户态拷贝数据时，按理数据应该都已经准备好了吧，也就是说相应的内存(物理)应该都已经分配好了把，不应该发生page fault了吧？

正常情况下，的确如此，但万一内核非要访问未映射的用户态地址呢？万一用户态数据此时没有准备好呢？谁能保证？

所以，虽然不应该发生，但还是可能发送的。

# 有什么不一样么？

这种情况下，在内核中发生的page fault，与平常我们见到的在用户态发生的page fault有什么不同么？

在用户态时，发生缺页时，CPU硬件会切换到到特权模式(内核态)，进行异常处理；在内核态时发生缺页，由于当前本来就出于内核态，所以处理方式肯定会有不同。这也许只是一方面，应该还有其它不同～

# 如何处理的？

在page fault的流程中针对Vmalloc区发生的缺页有专门的处理，可以参见vmalloc_fault函数。

那对于copy_from_user的情况呢？

肯定的。其实就是do_page_fault流程中exception table相关的处理，之前对这里的代码没有仔细理解清楚，原来就是用来处理这种情况的。

# 为什么要用copy_from_user?

还是回到老问题，为什么要用copy_from_user，按理解，拷贝数据，不就是从一个地址搬移数据到另一个地址，用memcpy不就好了？

就是因为可能发生page fault的情况，memcpy对这种情况没有任何的处理措施，可能引发不可知的后果。

而copy_from_user对这种情况进行了处理，处理方式就是在代码中增加了一个.fixup的段，该段在do_page_fault中会被读取，结合exception table进行相应的修正，具体修正方式，没时间深入研究了，感兴趣可以继续看看。

copy_from_user代码如下：

    static unsigned long
    __copy_user_intel(void __user *to, const void *from, unsigned long size)
    {
    	int d0, d1;
    	__asm__ __volatile__(
    		       "       .align 2,0x90\n"
    		       "1:     movl 32(%4), %%eax\n"
    		       "       cmpl $67, %0\n"
    		       "       jbe 3f\n"
    		       "2:     movl 64(%4), %%eax\n"
    		       "       .align 2,0x90\n"
    		       "3:     movl 0(%4), %%eax\n"
    		       "4:     movl 4(%4), %%edx\n"
    		       "5:     movl %%eax, 0(%3)\n"
    		       "6:     movl %%edx, 4(%3)\n"
    		       "7:     movl 8(%4), %%eax\n"
    		       "8:     movl 12(%4),%%edx\n"
    		       "9:     movl %%eax, 8(%3)\n"
    		       "10:    movl %%edx, 12(%3)\n"
    		       "11:    movl 16(%4), %%eax\n"
    		       "12:    movl 20(%4), %%edx\n"
    		       "13:    movl %%eax, 16(%3)\n"
    		       "14:    movl %%edx, 20(%3)\n"
    		       "15:    movl 24(%4), %%eax\n"
    		       "16:    movl 28(%4), %%edx\n"
    		       "17:    movl %%eax, 24(%3)\n"
    		       "18:    movl %%edx, 28(%3)\n"
    		       "19:    movl 32(%4), %%eax\n"
    		       "20:    movl 36(%4), %%edx\n"
    		       "21:    movl %%eax, 32(%3)\n"
    		       "22:    movl %%edx, 36(%3)\n"
    		       "23:    movl 40(%4), %%eax\n"
    		       "24:    movl 44(%4), %%edx\n"
    		       "25:    movl %%eax, 40(%3)\n"
    		       "26:    movl %%edx, 44(%3)\n"
    		       "27:    movl 48(%4), %%eax\n"
    		       "28:    movl 52(%4), %%edx\n"
    		       "29:    movl %%eax, 48(%3)\n"
    		       "30:    movl %%edx, 52(%3)\n"
    		       "31:    movl 56(%4), %%eax\n"
    		       "32:    movl 60(%4), %%edx\n"
    		       "33:    movl %%eax, 56(%3)\n"
    		       "34:    movl %%edx, 60(%3)\n"
    		       "       addl $-64, %0\n"
    		       "       addl $64, %4\n"
    		       "       addl $64, %3\n"
    		       "       cmpl $63, %0\n"
    		       "       ja  1b\n"
    		       "35:    movl  %0, %%eax\n"
    		       "       shrl  $2, %0\n"
    		       "       andl  $3, %%eax\n"
    		       "       cld\n"
    		       "99:    rep; movsl\n"
    		       "36:    movl %%eax, %0\n"
    		       "37:    rep; movsb\n"
    		       "100:\n"
    		       ".section .fixup,\"ax\"\n"
    		       "101:   lea 0(%%eax,%0,4),%0\n"
    		       "       jmp 100b\n"
    		       ".previous\n"
    		       _ASM_EXTABLE(1b,100b)
    		       _ASM_EXTABLE(2b,100b)
    		       _ASM_EXTABLE(3b,100b)
    		       _ASM_EXTABLE(4b,100b)
    		       _ASM_EXTABLE(5b,100b)
    		       _ASM_EXTABLE(6b,100b)
    		       _ASM_EXTABLE(7b,100b)
    		       _ASM_EXTABLE(8b,100b)
    		       _ASM_EXTABLE(9b,100b)
    		       _ASM_EXTABLE(10b,100b)
    		       _ASM_EXTABLE(11b,100b)
    		       _ASM_EXTABLE(12b,100b)
    		       _ASM_EXTABLE(13b,100b)
    		       _ASM_EXTABLE(14b,100b)
    		       _ASM_EXTABLE(15b,100b)
    		       _ASM_EXTABLE(16b,100b)
    		       _ASM_EXTABLE(17b,100b)
    		       _ASM_EXTABLE(18b,100b)
    		       _ASM_EXTABLE(19b,100b)
    		       _ASM_EXTABLE(20b,100b)
    		       _ASM_EXTABLE(21b,100b)
    		       _ASM_EXTABLE(22b,100b)
    		       _ASM_EXTABLE(23b,100b)
    		       _ASM_EXTABLE(24b,100b)
    		       _ASM_EXTABLE(25b,100b)
    		       _ASM_EXTABLE(26b,100b)
    		       _ASM_EXTABLE(27b,100b)
    		       _ASM_EXTABLE(28b,100b)
    		       _ASM_EXTABLE(29b,100b)
    		       _ASM_EXTABLE(30b,100b)
    		       _ASM_EXTABLE(31b,100b)
    		       _ASM_EXTABLE(32b,100b)
    		       _ASM_EXTABLE(33b,100b)
    		       _ASM_EXTABLE(34b,100b)
    		       _ASM_EXTABLE(35b,100b)
    		       _ASM_EXTABLE(36b,100b)
    		       _ASM_EXTABLE(37b,100b)
    		       _ASM_EXTABLE(99b,101b)
    		       : "=&c"(size), "=&D" (d0), "=&S" (d1)
    		       :  "1"(to), "2"(from), "0"(size)
    		       : "eax", "edx", "memory");
    	return size;
    }

代码整体上跟memcpy实现差不多，主要差别就是fixup段了，感兴趣的TX可以对比看看。
