---
layout: post
title:  "Mips TLB miss实现in Linux"
date:   2016-12-28 06:35:18
author: HappySeeker
categories: Kernel
---

TLB miss是Mips中内存管理的核心流程。上一篇写了关于Mips中，TLB miss的相关原理，本文关注在Linux kernel中的代码实现。

# TLB Refill初始化

内核启动过程中，会对TLB Refill异常进行初始化，设置相应的处理接口。主要流程如下(以R3k为例)：

    start_secondary
      per_cpu_trap_init
        tlb_init
          build_tlb_refill_handler
            build_r3000_tlb_refill_handler

  具体设置处理函数的操作在build_r3000_tlb_refill_handler进行，代码如下：

      /*
     * The R3000 TLB handler is simple.
     */
    /*
     * 标准的TLB Refill代码，相对比较简单，只用一级页表，跟Mips手册中的基本一致。
     */
    static void build_r3000_tlb_refill_handler(void)
    {
      /* 页目录 */
    	long pgdc = (long)pgd_current;
    	u32 *p;
      /* 初始化tlb处理接口 */
    	memset(tlb_handler, 0, sizeof(tlb_handler));
    	p = tlb_handler;
      /* 读取产生异常的虚拟地址 */
    	uasm_i_mfc0(&p, K0, C0_BADVADDR);
    	uasm_i_lui(&p, K1, uasm_rel_hi(pgdc)); /* cp0 delay */
    	uasm_i_lw(&p, K1, uasm_rel_lo(pgdc), K1);
    	uasm_i_srl(&p, K0, K0, 22); /* load delay */
    	uasm_i_sll(&p, K0, K0, 2);
    	uasm_i_addu(&p, K1, K1, K0);
      /* 从CONTEXT寄存器中读取信息 */
    	uasm_i_mfc0(&p, K0, C0_CONTEXT);
    	uasm_i_lw(&p, K1, 0, K1); /* cp0 delay */
    	uasm_i_andi(&p, K0, K0, 0xffc); /* load delay */
    	uasm_i_addu(&p, K1, K1, K0);
    	uasm_i_lw(&p, K0, 0, K1);
    	uasm_i_nop(&p); /* load delay */
      /* 写ENTRYLO0寄存器，供后面的tlbwr指令使用，将其写入TLB */
    	uasm_i_mtc0(&p, K0, C0_ENTRYLO0);
      /* 读取EPC寄存器，其中保存了异常返回地址 */
    	uasm_i_mfc0(&p, K1, C0_EPC); /* cp0 delay */
      /* tlbwr指令，根据ENTRYLO0寄存器内容(来源于页表，通过Context寄存器索引)，写TLB */
    	uasm_i_tlbwr(&p); /* cp0 delay */
      /* 跳转到异常返回地址，退出异常 */
    	uasm_i_jr(&p, K1);
    	uasm_i_rfe(&p); /* branch delay */
    	...
      /*
       * 关键点：将tlb_handler(也就是前面设置的TLB refill异常处理函数)内容拷贝到ebase
       * (Mips为异常分配的物理地址空间(大小为0x80)，用ebase寄存器指向，简单情况下，就是
       * 物理地址0x0)，当TLB Refill异常发生时，硬件会自动跳转到ebase对应的地址执行。
       */
      memcpy((void *)ebase, tlb_handler, 0x80);
      ...
    }

  可以看出，TLB refill异常的处理函数代码，是内核动态设置的，而不是汇编静态代码，由于Mips硬件型号比较多，如此操作更具灵活性。

  同时，请注意TLB refill异常处理函数入口位于固定物理地址：0x0。

# General Exception初始化

内核启动过程中，会对General Exception相应的异常处理接口进行初始化，主要流程如下：

    start_kernel
      trap_init

trap_init完成异常相关初始化操作，主要代码如下：

    void __init trap_init(void)
    {
      // General Exception处理接口
    	extern char except_vec3_generic;
      ...
    	/*
    	 * Copy the generic exception handlers to their final destination.
    	 * This will be overriden later as suitable for a particular
    	 * configuration.
    	 */
      /*
       * 将General Exception的处理函数(指令)拷贝到0x180(物理地址)中，0x180是r4000以上
       * 的Mips CPU中General Exception默认入口地址，该段物理地址空间大小为0x80,专门
       * 预留出来用于General Exception的处理。
       */
    	set_handler(0x180, &except_vec3_generic, 0x80);

    	/*
    	 * Setup default vectors
    	 */
      /×
       * 为0-31号异常设置默认的处理接口，General Exception对应编号为3,TLB refill异常
       * 对应编号为1。
       */
    	for (i = 0; i <= 31; i++)
    		set_except_vector(i, handle_reserved);

      ...
      /*
       * General Exception是所有普通优先级异常的总入口，其中会根据异常码调用不同的处理
       * 接口，这里就是设置不同的异常吗对应的处理接口。
       * 注意：2号异常码对应为**page fault**异常，即TLB Load异常，处理接口为handle_tlbl
       */
    	set_except_vector(0, using_rollback_handler() ? rollback_handle_int
    						      : handle_int);
    	set_except_vector(1, handle_tlbm);
    	set_except_vector(2, handle_tlbl);
    	set_except_vector(3, handle_tlbs);

    	set_except_vector(4, handle_adel);
    	set_except_vector(5, handle_ades);

    	set_except_vector(6, handle_ibe);
    	set_except_vector(7, handle_dbe);
      /* 设置系统调用处理接口 */
    	set_except_vector(8, handle_sys);

      /*
       * 根据不同的CPU类型，设置相应的General Exception的处理接口地址，对应R4,地址为
       * 0x180，其它(更老的)为0x80.
       */
    	if (cpu_has_vce)
    		/* Special exception: R4[04]00 uses also the divec space. */
    		set_handler(0x180, &except_vec3_r4000, 0x100);
    	else if (cpu_has_4kex)
    		set_handler(0x180, &except_vec3_generic, 0x80);
    	else
    		set_handler(0x080, &except_vec3_generic, 0x80);
      ...
    }

# General Exception处理

General Exception处理在except_vec3_generic函数中，以汇编实现，其中主要工作是：读取异常类型码ExcCode，然后调用相应的处理接口。主要代码如下(genex.S文件中)：

    /*
     * General exception vector for all other CPUs.
     *
     * Be careful when changing this, it has to be at most 128 bytes
     * to fit into space reserved for the exception handler.
     */
    /*
     * 汇编代码，最终被拷贝到0x180的物理地址中，注意这段物理地址空间是有大小限制的，最大为128
     * 字节(0x80)，所以处理不能太复杂。
     */
    NESTED(except_vec3_generic, 0, sp)
    	.set	push
    	.set	noat
    #if R5432_CP0_INTERRUPT_WAR
    	mfc0	k0, CP0_INDEX
    #endif
    	mfc0	k1, CP0_CAUSE  //从CP0_CAUSE寄存器中读取异常类型码ExcCode
    	andi	k1, k1, 0x7c
    #ifdef CONFIG_64BIT
    	dsll	k1, k1, 1
    #endif
      /*
       * 根据类型码，条用exception_handlers中对应的接口,exception_handlers中的处理接口
       * 是在trap_init初始化时设置的。
       */
    	PTR_L	k0, exception_handlers(k1)
    	jr	k0
    	.set	pop
    	END(except_vec3_generic)

# TLB Load异常处理初始化

前面的General Exception初始化流程中，仅设置了General Exception的处理入口和不同的异常码对应的处理接口，但是并设置具体的异常处理函数内容，即异常发生后具体执行的代码，其初始化是在另外的流程中：

    start_secondary
      per_cpu_trap_init
        tlb_init
          build_tlb_refill_handler
            build_r3000_tlb_load_handler

具体实现在build_r3000_tlb_load_handler函数中，关键代码如下：

    static void build_r3000_tlb_load_handler(void)
    {
    	u32 *p = handle_tlbl;
    	const int handle_tlbl_size = handle_tlbl_end - handle_tlbl;
      ...
      /* 初始化 */
    	memset(handle_tlbl, 0, handle_tlbl_size * sizeof(handle_tlbl[0]));
      ...
      /*
       * 写汇编代码，相当于j tlb_do_page_fault_0，即直接跳转到tlb_do_page_fault_0处
       * 执行，即调用tlb_do_page_fault_0函数
       */
    	uasm_i_j(&p, (unsigned long)tlb_do_page_fault_0 & 0x0fffffff);
    	uasm_i_nop(&p);
      ...
    }


# TLB Load异常处理(Page Fault)

如前面的代码分析，tlb load异常的处理接口中，其实就是调用tlb_do_page_fault_0函数，再看看这个函数实现(汇编代码，在tlbex-fault.S文件中)：

    /* tlb_do_page_fault_0和tlb_do_page_fault_1一起实现，通过宏控制 */
    .macro tlb_do_page_fault, write
    NESTED(tlb_do_page_fault_\write, PT_SIZE, sp)
    /* 保存上下文 */
    SAVE_ALL
    /* 读取发送异常的虚拟地址 */
    MFC0	a2, CP0_BADVADDR
    KMODE
    move	a0, sp
    REG_S	a2, PT_BVADDR(sp)
    li	a1, \write
    PTR_LA	ra, ret_from_exception
    /* 关键点：调用do_page_fault函数 */
    j	do_page_fault
    END(tlb_do_page_fault_\write)
    .endm

代码很简单，除保存上下文之类的操作外，就是调用do_page_fault函数了，这个就是我们很熟悉的page fault的标准接口了，其中会分配内存，并新增页表项，建立虚拟地址和物理地址的映射关系。这里就不说的，以前写过分析文档。
