---
layout: post
title:  "Mips架构学习笔记"
date:   2016-11-11 05:58:33
author: HappySeeker
categories: Kernel
---

龙芯龙芯，需要学习Mips，至少了解下。

# Mips架构组件
- ISA(Instruction Set Architecture)
- Privileged Resource Architecture (PRA)
The MIPS32 and MIPS64 Privileged Resource Architecture defines a set of environments and capabilities on which the
ISA operates.
- MIPS Application Specific Extensions (ASEs)
- MIPS User Defined Instructions (UDIs)

# 架构和实现
架构和实现不同，需要区分

When describing the characteristics of MIPS processors, architecture must be distinguished from the hardware
implementation of that architecture.

• Architecture refers to the instruction set, registers and other state, the exception model, memory management,
virtual and physical address layout, and other features that all hardware executes.

• Implementation refers to the way in which specific processors apply the architecture.Here are two examples:
1. A floating point unit (FPU) is an optional part of the MIPS64 Architecture. A compatible implementation of the
FPU may have different pipeline lengths, different hardware algorithms for performing multiplication or division,
etc.
2. Most MIPS processors have caches; however, these caches are not implemented in the same manner in all MIPS
processors. Some processors implement physically-indexed, physically tagged caches. Other implement
virtually-indexed, physically-tagged caches. Still other processor implement more than one level of cache.
The MIPS64 architecture is decoupled from specific hardware implementations, leaving microprocessor designers free
to create their own hardware designs within the framework of the architectural definition.

# MIPS32和MIPS64的关系
由于需要保证兼容性。Mips32是严格的mips64架构的子集。

# Mips32指令集（ISA）
Table 2-1 MIPS32 Instructions
ABS.PS ABS.D ADD ADD.D ADD.PS ADD.S ADDI ADDIU ADDU ALNV.PS AND ANDI BC1F BC1FL BC1T BC1TL BC2F BC2FL BC2T BC2TL BEQ BEQL BGEZ BGEZAL BGEZALL BGEZL BGTZ BGTZL BLEZ BLEZL BLTZ BLTZAL BLTZALL BLTZL BNE BNEL BREAK C.cond.D C.cond.PS CACHE CEIL.L.D CEIL.L.S CEIL.W.D CEIL.W.S CFC1 CFC2 C.cond.S CLO CVT.L.D ABS.S CLZ  COP2
CVT.L.S CVT.PS.S CTC1 CTC2 CVT.D.L CVT.D.S CVT.D.W
CVT.S.D CVT.S.L ...

# Mips64指令集
DADD DADDI DADDIU DADDU DCLO DDIV DDIVU DEXT DEXTM DEXTU DINS DINSM DINSU DLCZ DMFC0 DMFC1 DMFC2 DMTC0 DMTC1 DROTRV DSBH  DSRAV LDR DMULTU DROTR DROTR32 DSLL32 DSLLV DSRA DSRA32 DSUB DSUBU LD LDL SD SDL SDR DMTC2 DMULT DSHD DSLL DSRL DSRL32 DSRLV LLD LWU SCD

# 流水线架构
## 流水线stage
- 指令预取
- Arithmetic operation-算术操作
- 内存访问
- write back -回写
每个stage消耗一个时钟周期，所以如果串行执行的化，一个指令需要4个时钟周期。

## 并发流水线(Parallel Pipeline)
流水线并发后，理论上一个指令只需要一个时钟周期（接近）

## Superpipeline
将每个时钟周期切分为更细的片段，那么每个时钟周期可以执行多个stage，因此流水线并发后，每个时钟周期就可以执行>1条指令了

## Superscalar Pipeline
4 way 5 stage
也可以实现每个时钟周期执行>1条指令。
IF = instruction fetch
ID = instruction decode and dependency
IS = instruction issue
EX = execution
WB = write back

## Load/Store Architecture
由于寄存器和内存访问的性能差异，MIPS使用load/Store设计：
- 所有指令都只操作寄存器中的数据。
- 内存只能通过Load/Store指令访问。

好处：
- 减少内存访问，减少内存带宽需求
- 简化指令集
- 更便于编译器优化寄存器分配

# 编程模型
## CPU数据格式
- Bit (b)
- Byte (8 bits, B)
- Halfword (16 bits, H)
- Word (32 bits, W)
- Doubleword (64 bits, D)

## FPU数据格式
- 32-bit single-precision floating point (.fmt type S)
- 32-bit single-precision floating point paired-single (.fmt type PS) 1
- 64-bit double-precision floating point (.fmt type D)
- 32-bit Word fixed point (.fmt type W)
- 64-bit Long fixed point (.fmt type L)

## 协处理器(CP0-CP3)
MIPS架构定义了4个协处理器：

- CP0, 系统控制协处理器，协助完成虚拟内存、异常处理
CP0 translates virtual addresses into physical addresses, manages exceptions, and handles switches between kernel,
supervisor, and user states. CP0 also controls the cache subsystem, as well as providing diagnostic control and error
recovery facilities. The architectural features of CP0 are defined in Volume III.
- CP1, FPU
- CP2, 取决具体实现
- CP3, FPU also

## CPU寄存器
MIPS64 Architecture定义的寄存器：

- 32个64位的通用寄存器(GPRs)
- a pair of special-purpose registers to hold the results of integer multiply, divide, and multiply-accumulate operations
(HI and LO)
- PC(程序计数器)

64位处理器总是产生64位的结果，即使时针对32位操作的指令，会对齐结果自动扩展到64位。

### 通用寄存器
r0和r31有特定的用途，其它的寄存器都是通用的。
- r0,s hard-wired to a value of zero, and can be used as the target register for any instruction whose result is to be
discarded. r0 can also be used as a source when a zero value is needed.
- r3l, is the destination register used by JAL, BLTZAL, BLTZALL, BGEZAL, and BGEZALL without being explicitly
specified in the instruction word. Otherwise r31 is used as a normal register.

### 特殊用途寄存器

如前面所述，只有3个：
HI、LO和PC

## 内存访问类型

所有的实现都提供如下两种访问类型：
- uncached，In an uncached access, physical memory resolves the access. Each reference causes a read or write to physical memory.Caches are neither examined nor modified.
- cached，In a cached access, physical memory and all caches in the system containing a copy of the physical location are used to
resolve the access. A copy of a location is coherent if the copy was placed in the cache by a cached coherent access; a
copy of a location is noncoherent if the copy was placed in the cache by a cached noncoherent access. (Coherency is
dictated by the system architecture, not the processor implementation.)
Caches containing a coherent copy of the location are examined and/or modified to keep the contents of the location
coherent. It is not possible to predict whether caches holding a noncoherent copy of the location will be examined and/or
modified during a cached coherent access.

具体架构实现时，可能会实现其它的访问类型。

# ASE（Application Specific Extensions）

按需使用，主要包括：
- MIPS16e，is composed of 16-bit compressed code instructions, designed for the embedded processor market
and situations with tight memory constraints.
- The MIPS Digital Media Extension (MDMX) provides video, audio, and graphics pixel processing through vectors of
small integers. Although not a part of the MIPS ISA, this extension is included for informational purposes.
- The MIPS-3D ASE provides enhanced performance of geometry processing calculations by building on the paired single
floating point data type, and adding specific instructions to accelerate computations on these data types.
- The SmartMIPS ASE extends the MIPS32 Architecture with a set of new and modified instruction designed to
improve the performance and reduce the memory consumption of MIPS-based smart card or smart object systems.
- The MIPS DSP ASE provides enhanced performance of signal-processing applications by providing computational
support for fractional data types, SIMD, saturation, and other elements that are commonly used in such applications.
- The MIPS MT ASE provides the architecture to support multi-threaded implementations of the Architecture.

# CPU指令集overview

## 按功能分类：
- Load and store
- Computational
- Jump and branch
- Miscellaneous
- Coprocessor
