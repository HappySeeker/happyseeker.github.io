---
layout: post
title:  "Mips KVM Trap&Emulate implemented in Linux"
date:   2017-01-11 06:10:54
author: HappySeeker
categories: Kernel
---

# 基本原理

Trap&Emulate，即陷入&模拟的方式，是纯软件实现的全虚拟化方案，基本不借助硬件虚拟化功能。本文主要关注内存虚拟化实现中的核心，TLB miss相关实现。基本原理是：

所有的TLB miss都将导致Guest退出到VMM处理，然后在VMM中进行相应模拟。

具体实现原理描述如下：

- 当前内核版本(4.9)的实现方式定义了2中TLB：

  - Guest TLB。由Guest OS维护，用于映射GVA->GPA，本质为一段内存，用于模拟TLB，虚拟TLB，并不真实存在。该TLB不会被Guest OS直接使用，只用于生成Shadow TLB。Guest OS 在维护Guest TLB时,对 TLB 的读写指令会被虚拟机管理器(virtual machine monitor, VMM)捕捉并模拟,使 Guest OS 认为成功读写了硬件 TLB。
  - Shadow Host TLB，也是Host使用的物理TLB。由VMM(即Host维护)。对于Guest来说，用于映射GVA->HPA，对于Host来说，用于映射HVA->HPA。

- Guest最终通过Shadow Host TLB中的条目实现GVA到HPA的转换，GPA->HPA的映射通过线性映射表实现。

- Guest中，任何TLB相关的操作都将触发异常。

Guest OS做访存操作时有如下几中情况发生：

1. Shadow Host TLB(即硬件TLB)中刚好有该GVA到HPA的映射，此时不会发生异常，硬件TLB直接翻译地址，正常访问。
2. 如果硬件TLB中不存在相应映射条目，触发TLB Miss异常。此时退出到VMM中进行处理。
   - VMM先遍历Guest TLB，如果找到相应条目，则将Guest TLB中翻译后的GPA转换为HPA，然后填入Shadow Host TLB(硬件TLB)，并返回Guest OS继续运行。
   - 如果Guest TLB中没有找到，则向Guest OS中注入TLB Load/Store异常(由Guest OS负责重填Guest TLB)，然后填入Shadow Host TLB(硬件TLB)，并返回Guest OS继续运行。
    - 当Guest得到调度运行时，TLB Load/Store异常(General异常)入口，最终会进入page fault相关流程，完成Guest TLB重填。
    - 重填TLB时，会调用类似TLBWR之类的之令，此时会再次触发VM-Exit，VMM通过捕获该异常，其中填入虚拟的Guest TLB、flust相应的Shadow Host TLB条目，然后返回Guest OS继续执行。

# 代码实现

## kvm_mips_handle_tlbmiss

tlbmiss异常通用入口，在如下两种情况下发生：

1. TLB Entry在Guest TLB和shadow host TLB中都不存在。这种情况下，将相应异常注入Guest中，由Guest自己处理。
2. TLB Entry在Guest TLB中存在，但在shadow host TLB中不存在。这种情况下，由VMM根据Guest TLB中的相应Entry填充shadow host TLB。

主要处理逻辑：

- 查找Guest TLB中是否存在相应entry;
- 如果存在，则调用kvm_mips_handle_mapped_seg_tlb_fault填充shadow Host TLB(即物理TLB)；
- 如果不存在，则根据exccode向Guest中注入相应的异常，比如TLB miss load异常，接口如kvm_mips_emulate_tlbmiss_ld。

具体代码实现如下：

    enum emulation_result kvm_mips_handle_tlbmiss(u32 cause,
                                                  u32 *opc,
                                                  struct kvm_run *run,
                                                  struct kvm_vcpu *vcpu)
    {
            enum emulation_result er = EMULATE_DONE;
            /* 读取异常类型码 */
            u32 exccode = (cause >> CAUSEB_EXCCODE) & 0x1f;
            /* 获取发送tlbmiss的地址，为GVA地址 */
            unsigned long va = vcpu->arch.host_cp0_badvaddr;
            int index;

            kvm_debug("kvm_mips_handle_tlbmiss: badvaddr: %#lx\n",
                      vcpu->arch.host_cp0_badvaddr);

            /*
             * KVM would not have got the exception if this entry was valid in the
             * shadow host TLB. Check the Guest TLB, if the entry is not there then
             * send the guest an exception. The guest exc handler should then inject
             * an entry into the guest TLB.
             */
            /* 查找Guest TLB(本质为一段内存，虚拟TLB，并不真实存在)，确认是否存在 */
            index = kvm_mips_guest_tlb_lookup(vcpu,
                          (va & VPN2_MASK) |
                          (kvm_read_c0_guest_entryhi(vcpu->arch.cop0) &
                           KVM_ENTRYHI_ASID));
            /* 不存在 */
            if (index < 0) {
                    /* 根据异常类型码向Guest中注入不同的异常 */
                    if (exccode == EXCCODE_TLBL) {
                            /* TLB load异常 */
                            er = kvm_mips_emulate_tlbmiss_ld(cause, opc, run, vcpu);
                    } else if (exccode == EXCCODE_TLBS) {
                            /* TLB set异常 */
                            er = kvm_mips_emulate_tlbmiss_st(cause, opc, run, vcpu);
                    } else {
                            kvm_err("%s: invalid exc code: %d\n", __func__,
                                    exccode);
                            er = EMULATE_FAIL;
                    }
            } else {/* 如果在guest TLB中找到相应条目 */
                    /* 获取相应条目 */
                    struct kvm_mips_tlb *tlb = &vcpu->arch.guest_tlb[index];

                    /*
                     * Check if the entry is valid, if not then setup a TLB invalid
                     * exception to the guest
                     */
                    /* 检查条目是否可用，不可用，则再注入TLB相应异常 */
                    if (!TLB_IS_VALID(*tlb, va)) {
                            if (exccode == EXCCODE_TLBL) {
                                    er = kvm_mips_emulate_tlbinv_ld(cause, opc, run,
                                                                    vcpu);
                            } else if (exccode == EXCCODE_TLBS) {
                                    er = kvm_mips_emulate_tlbinv_st(cause, opc, run,
                                                                    vcpu);
                            } else {
                                    kvm_err("%s: invalid exc code: %d\n", __func__,
                                    exccode);
                            er = EMULATE_FAIL;
                          }
                    } else { /* 条目可用，则填充Host TLB */
                    kvm_debug("Injecting hi: %#lx, lo0: %#lx, lo1: %#lx into shadow host TLB\n",
                              tlb->tlb_hi, tlb->tlb_lo[0], tlb->tlb_lo[1]);
                    /*
                     * OK we have a Guest TLB entry, now inject it into the
                     * shadow host TLB
                     */
                    /* Host TLB的实际填充操作在kvm_mips_handle_mapped_seg_tlb_fault函数中完成*/
                    if (kvm_mips_handle_mapped_seg_tlb_fault(vcpu, tlb)) {
                            kvm_err("%s: handling mapped seg tlb fault for %lx, index: %u, vcpu: %p, ASID: %#lx\n",
                                    __func__, va, index, vcpu,
                                    read_c0_entryhi());
                            er = EMULATE_FAIL;
                    }
            }
        }

        return er;
      }



## kvm_mips_emulate_tlbmiss_ld

实现向Guest中注入TLB load异常，本质为：

设置CP0 Status中的ST0_EXL，然后将PC指向General异常的处理入口，Guest得到调度即可直接跳转到异常入口执行，实现异常注入。

    enum emulation_result kvm_mips_emulate_tlbinv_ld(u32 cause,
                                                     u32 *opc,
                                                     struct kvm_run *run,
                                                     struct kvm_vcpu *vcpu)
    {
            struct mips_coproc *cop0 = vcpu->arch.cop0;
            struct kvm_vcpu_arch *arch = &vcpu->arch;
            unsigned long entryhi =
                    (vcpu->arch.host_cp0_badvaddr & VPN2_MASK) |
                    (kvm_read_c0_guest_entryhi(cop0) & KVM_ENTRYHI_ASID);
            /*
             * 读取Guest C0中Status寄存器，判断ST0_EXL(标识当前是否已经处于异常状态，
             * 如果已经是，且再次触发tlbmiss异常，此时不会重入，而会触发General异常，
             * 即TLB load异常，也可以理解成X86中的缺页异常)，如果没有设置，则需要进行
             * 相应设置，以便能向Guest中注入TLB load异常 */
            if ((kvm_read_c0_guest_status(cop0) & ST0_EXL) == 0) {
                    /* save old pc */
                    kvm_write_c0_guest_epc(cop0, arch->pc);
                    /* 设置ST0_EXL标记 */
                    kvm_set_c0_guest_status(cop0, ST0_EXL);

                    if (cause & CAUSEF_BD)
                            kvm_set_c0_guest_cause(cop0, CAUSEF_BD);
                    else
                            kvm_clear_c0_guest_cause(cop0, CAUSEF_BD);

                    kvm_debug("[EXL == 0] delivering TLB INV @ pc %#lx\n",
                              arch->pc);

                    /* set pc to the exception entry point */
                    /*
                     * 设置当前的PC指针指向General异常的处理入口，这样，当Guest得到调度
                     * 时，就能直接进入General异常处理流程中，其中会最终走到page fault
                     * 相关流程，之前的文章中有相应的原理描述
                     */
                    arch->pc = KVM_GUEST_KSEG0 + 0x180;

            } else {/* 如果已经设置了ST0_EXL标记，则直接修改PC指针即可 */
                    kvm_debug("[EXL == 1] delivering TLB MISS @ pc %#lx\n",
                              arch->pc);
                    arch->pc = KVM_GUEST_KSEG0 + 0x180;
            }
            /* 写入EXCCODE为TLBL */
            kvm_change_c0_guest_cause(cop0, (0xff),
                                      (EXCCODE_TLBL << CAUSEB_EXCCODE));

            /* setup badvaddr, context and entryhi registers for the guest */
            kvm_write_c0_guest_badvaddr(cop0, vcpu->arch.host_cp0_badvaddr);
            /* XXXKYMA: is the context register used by linux??? */
            kvm_write_c0_guest_entryhi(cop0, entryhi);
            /* Blow away the shadow host TLBs */
            kvm_mips_flush_host_tlb(1);

            return EMULATE_DONE;
    }

## kvm_mips_handle_mapped_seg_tlb_fault

主要完成任务：根据Guest TLB中的entry，填充Host TLB中相应条目。

    int kvm_mips_handle_mapped_seg_tlb_fault(struct kvm_vcpu *vcpu,
                                             struct kvm_mips_tlb *tlb)
    {
            unsigned long entryhi = 0, entrylo0 = 0, entrylo1 = 0;
            struct kvm *kvm = vcpu->kvm;
            kvm_pfn_t pfn0, pfn1;
            gfn_t gfn0, gfn1;
            long tlb_lo[2];
            int ret;

            tlb_lo[0] = tlb->tlb_lo[0];
            tlb_lo[1] = tlb->tlb_lo[1];

            /*
             * The commpage address must not be mapped to anything else if the guest
             * TLB contains entries nearby, or commpage accesses will break.
             */
            if (!((tlb->tlb_hi ^ KVM_GUEST_COMMPAGE_ADDR) &
                            VPN2_MASK & (PAGE_MASK << 1)))
                    tlb_lo[(KVM_GUEST_COMMPAGE_ADDR >> PAGE_SHIFT) & 1] = 0;
            /* 获取gfn */
            gfn0 = mips3_tlbpfn_to_paddr(tlb_lo[0]) >> PAGE_SHIFT;
            gfn1 = mips3_tlbpfn_to_paddr(tlb_lo[1]) >> PAGE_SHIFT;
            if (gfn0 >= kvm->arch.guest_pmap_npages ||
                gfn1 >= kvm->arch.guest_pmap_npages) {
                    kvm_err("%s: Invalid gfn: [%#llx, %#llx], EHi: %#lx\n",
                    __func__, gfn0, gfn1, tlb->tlb_hi);
            kvm_mips_dump_guest_tlbs(vcpu);
            return -1;
      }
          /*
           * 建立gfn和pfn的映射关系，本质是通memslot中获取对应关系，然后设置guest_pmap线性
           * 映射数组，后面使用
           */
          if (kvm_mips_map_page(kvm, gfn0) < 0)
                  return -1;

          if (kvm_mips_map_page(kvm, gfn1) < 0)
                  return -1;
          /* 获取pfn */
          pfn0 = kvm->arch.guest_pmap[gfn0];
          pfn1 = kvm->arch.guest_pmap[gfn1];

          /* Get attributes from the Guest TLB */
          /* 从Guest TLB中获取entrylo0、entrylo1、entryhi等关键信息，以便于后续将其写入Host TLB */
          entrylo0 = mips3_paddr_to_tlbpfn(pfn0 << PAGE_SHIFT) |
                  ((_page_cachable_default >> _CACHE_SHIFT) << ENTRYLO_C_SHIFT) |
                  (tlb_lo[0] & ENTRYLO_D) |
                  (tlb_lo[0] & ENTRYLO_V);
          entrylo1 = mips3_paddr_to_tlbpfn(pfn1 << PAGE_SHIFT) |
                  ((_page_cachable_default >> _CACHE_SHIFT) << ENTRYLO_C_SHIFT) |
                  (tlb_lo[1] & ENTRYLO_D) |
                  (tlb_lo[1] & ENTRYLO_V);

          kvm_debug("@ %#lx tlb_lo0: 0x%08lx tlb_lo1: 0x%08lx\n", vcpu->arch.pc,
                    tlb->tlb_lo[0], tlb->tlb_lo[1]);

          preempt_disable();
          entryhi = (tlb->tlb_hi & VPN2_MASK) | (KVM_GUEST_KERNEL_MODE(vcpu) ?
                                                 kvm_mips_get_kernel_asid(vcpu) :
                                                 kvm_mips_get_user_asid(vcpu));
          /* 将TLB Entry信息写入Host TLB，即物理TLB */
          ret = kvm_mips_host_tlb_write(vcpu, entryhi, entrylo0, entrylo1,
                                    tlb->tlb_mask);
          preempt_enable();

          return ret;
    }

## kvm_mips_map_page

建立gfn和pfn的映射关系，本质是通memslot中获取对应关系，然后设置guest_pmap线性映射数组

    static int kvm_mips_map_page(struct kvm *kvm, gfn_t gfn)
    {
            int srcu_idx, err = 0;
            kvm_pfn_t pfn;
            /* 已经存在映射 */
            if (kvm->arch.guest_pmap[gfn] != KVM_INVALID_PAGE)
                    return 0;

            srcu_idx = srcu_read_lock(&kvm->srcu);
            /* 将gfn转换为pfn，通过memslot实现，kvm标准接口 */
            pfn = gfn_to_pfn(kvm, gfn);

            if (is_error_noslot_pfn(pfn)) {
                    kvm_err("Couldn't get pfn for gfn %#llx!\n", gfn);
                    err = -EFAULT;
                    goto out;
            }
            /* 填入映射关系 */
            kvm->arch.guest_pmap[gfn] = pfn;
    out:
            srcu_read_unlock(&kvm->srcu, srcu_idx);
            return err;
    }
