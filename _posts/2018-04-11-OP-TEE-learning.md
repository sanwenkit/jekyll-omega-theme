---
layout: post
title: OP-TEE架构设计
description: "OP-TEE学习"
category: develop
tags: [Trustzone, OPTEE]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---


# OP-TEE架构设计

### 介绍

OP-TEE包含三个组件：
* OP-TEE Client:  Normal World用户空间运行的客户端API封装
* OP-TEE Linux Kernel Driver: 内核驱动来处理Normal World与Trust World之间的信息通讯
* OP-TEE Trust OS: Trust World运行的OS系统，包含系统内核以及一些TA运行需要使用的类库。TA可以通过调用这些类库来完成对Trust OS内核的系统调用。

整体系统架构如下图所示：
![OP-TEE-Arch](http://7xwdx7.com1.z0.glb.clouddn.com/OP-TEE-Arch.jpg)


### SMC
OP-TEE中的SMC接口被定义为两个等级，一种为optee_smc.h中定义的接口，规定了每个SMC调用中寄存器的用途定义，另一种为optee_msg.h中定义的接口，定义了OP-TEE Message协议的格式。

SMC通讯的主要结构被定义在struct optee_msg_arg中。对于OP-TEE Linux Kernel Driver来说，它可能从OP-TEE Client获取输入参数，也可能直接从Linux内核的其他服务获取输入参数。TEE驱动会填充该结构体，并加入一些额外的登记信息。SMC调用时，a0寄存器存储SMC id，a1-a7寄存器用来传递相关参数。

### MMU
OP-TEE使用多个L1转换表，一个为4G大小，另外存在至少2个32M大小的转换表。大的转换表负责内核模式地址Mapping以及掐所有小转换表无法覆盖的地址转换mapping。小转换表被分配到每个线程，并负责为每个TA运行的上下文空间映射虚拟内存。

在大表、小表之间的内存地址转换由TTBRC配置。TTBR1永远指向大的转换表，TTBR0指向active状态的小转换表以及非active状态的大转换表。

![MMU](http://7xwdx7.com1.z0.glb.clouddn.com/MMU-Translation-table.jpg)

地址转换存在一些对准约束，转换表必须和物理地址进行相同大小的对齐。转换表被静态分配以防止出现内存碎片。

每个线程都有自己的小转换表，每个TA上下文都有一个转换表快照。当TA上下文处于active运行状态时，转换表快照会被用于初始化转换表。

用户空间地址转换： 通过配置 CFG_WITH_LPAE=n CFG_CORE_UNMAP_CORE_AT_EL0=y 开启。 当转换到用户空间时，只有最小化的内核地址空间映射会被保存。这个机制时通过在转换到用户空间时，在TTBR1指向一张空表来实现的。当返回内核空间时，TTBR1会被重新指向原来的L1地址转换表。

Normal World地址转换： 当通过外部中断或RPC调用转换进入Normal World时，在其他CPU中可以继续进行Trust World的程序运行。这时，TA运行上下文被设置在该CPU保持不变。

### Stack

在不同阶段，会使用多个不同的stack：
* Secure Monitor Stack: 128Bytes,和CPU绑定。在OP-TEE以Secure Monitor模式编译时存在（适用于ARMv7，不适用于ARMv8）
* Temp Stack：空间较小，约1KB，和CPU绑定。当CPU从一个运行级转换到另一个运行级时使用。当使用该STACK时，中断不会被响应，一旦退出会造成系统级影响。
* Abort Stack：空间中等，约2KB，和CPU绑定。当出现DATA或PRE-FETCH自陷时使用，用户空间内的退出仅会造成该TA的关闭，内核空间中分页器使用abort来获取分页数据，当分页器功能关闭时abort会造成系统级影响
* Thread Stack： 大空间，约8KB，不与CPU绑定，只和当前任务/线程绑定。当使用该stack时，支持中断。

对于ARMv7/AArch32架构，堆栈地址被分配到如下寄存器：

| Stack  | Comment |
|--------|---------|
| Temp   | Assigned to `SP_SVC` during entry/exit, always assigned to `SP_IRQ` and `SP_FIQ` |
| Abort  | Always assigned to `SP_ABT` |
| Thread | Assigned to `SP_SVC` while a thread is active |

启动： 在启动阶段，CPU一致使用Temp Stack，直到OP-TEE第一次进入Normal World。

从Normal World正常进入： 每次OP-TEE从Normal World进入Trust World时，使用temp stack作为初始化堆栈。对于fast call，这是唯一使用的堆栈；对于normal call将开辟一个线程空间并转移到该空间的堆栈地址。

正常退出到Normal World： 正常退出到Normal World发生在一个线程工作完成并被Free。线程主要工作逻辑tee_entry_std()完成并返回，所有中断被暂停，CPU被切换到temp stack。随后，这个线程将会被Free，OP-TEE返回到Normal World。

RPC退出： 当OP-TEE需要Normal World部分服务时，会产生RPC退出。RPC退出只能在一个线程运行时状态下发起，通过调用thread_rpc()来触发，线程运行上下文会被保持，随后如同线程正常运行完成一样，CPU被切换到temp stack，OP-TEE返回到Normal World。

外部中断退出： 当OP-TEE接收到外部中断时，会返回到Normal World。这是因为ARM GICv2中外部中断由IRQ发送并永远在Normal World进行处理。外部中断退出和RPC退出非常相似，不同之处在于由thread_irq_handler() （ ARMv7-A/Aarch32 ）或者elx_irq()（ARMv7-A/Aarch64）来完成线程运行上下文的保存。对于ARM GICv3，外部中断由FIQ发送并可以在Normal World或Trust World处理，OP-TEE目前并不支持该模式。

 ARMv7/AArch32: SP_IRQ被初始化为temp stack而非一个线程stack，在返回normal world前，CPU状态被切换到SVC，temp stack被选中。

 AArch64: 在进行IRQ处理时，SP_EL0被分配为temp stack。原始的SP_EL0寄存器值被保存在线程运行上下文中，并会在恢复线程运行时被重新写入。

 恢复进入： 在恢复进入过程中，OP-TEE使用temp stack作为初始化堆栈，与正常进入模式一致。随后将查询需要恢复的线程，并恢复其运行上下文。RPC退出和外部中断退出的恢复流程是一致的。

 系统调用： 系统调用在线程stack中运行。ARMv7/AArch32中，SP_SVC在线程stack中已经被设置，ARMv7/AArch64中，在处理异常中断时，原始的SP_EL0被存储在struct thread_svc_regs中，同时当前线程stack被写入SP_EL0作为当前堆栈。当系统调用退出时，从struct thread_svc_regs中还原SP_EL0，这允许tee_svc_sys_return_helper()直接返回到thread_unwind_user_mode()。


 ### 共享内存

 共享内存是一块同时被Normal World和Trust World共享的内存区域，它被用来在两种模式间传递数据。

 共享内存分配： 共享内存通过Linux驱动从一个池（struct shm_pool）中分配,这个内存池包括：

    内存池的起始物理地址
    内存池大小
    内存是否被缓存
    已分配的内存块列表

备注：

    共享内存池是物理连续的
    共享内存池区域并不安全，因为它同时被Normal World和Trust World使用

共享内存配置： OP-TEE中，linux内核驱动负责初始化共享内存池，相关信息由OP-TEE core提供。Linux内核驱动通过一个SMC调用OPTEE_SMC_GET_SHM_CONFIG来获取以下信息：

    内存池的起始物理地址
    内存池大小
    内存是否被缓存

共享内存配置是平台相关的，内存映射，包括MEM_AREA_NSEC_SHM区域，通过调用一个平台相关的函数bootcfg_get_memory()来获取。

共享内存块分配： 在OP-TEE中Linux内核驱动负责分配共享内存块。该内核驱动依赖于内核通用的分配支持（CONFIG_GENERIC_ALLOCATION)）来完成共享内存块的分配/释放工作。该内核驱动依赖于内核的dma-buf支持（CONFIG_DMA_SHARED_BUFFER）来维护共享内存buffer引用。

共享内存使用：

* 用户空间应用程序使用  应用程序可以通过GlobalPlatform Client API功能TEEC_AllocateSharedMemory()来申请分配共享内存。应用程序也可以通过TEEC_RegisterSharedMemory()功能直接提供共享内存。如果直接提供共享内存，那么必须保证该内存段是物理地址连续的，因为OP-TEE core没有能力处理分散内存。当一个共享内存块被注册时，其引用计数会被加1，当共享内存块被分配创建时其引用计数为1。

* 内核驱动使用  在某些场景下内核驱动使用共享内存来与Trust World进行通讯，例如使用TEEC_TempMemoryReference类型时。

* OP-TEE core使用  当OP-TEE core需要从TEE Supplicant获取信息时（例如动态TA加载，REE时间请求等），必须分配共享内存。OP-TEE core可以进行创建如下类型共享内存：

    optee_msg_arg结构，用来向Normal World传递参数，通过发送OPTEE_SMC_RPC_FUNC_ALLOC消息来进行创建分配

    在一些情况下，需要创建一个共享内存块并提供TEE Supplicant写入数据。例如进行TA的动态加载，该类型的创建分配通过消息OPTEE_MSG_RPC_CMD_SHM_ALLOC(OPTEE_MSG_RPC_SHM_TYPE_APPL,...)进行，并返回如下结果：共享内存的物理地址，一个用于Free该内存块的内存操作句柄

* TEE Supplicant使用  TEE Supplicant也需要共享内存来进行数据交互。例如在一次动态加载TA的过程中，TEE Supplicant从OP-TEE core获取来一个内存地址，那么它首先应该注册该共享内存，正如同客户端应用、内核驱动一样。

### Trusted Application

Globalplatform Core Internal API描述了提供TA使用的所有服务，libutee是一个该API实现的类库。libutee是一个静态链接库，与TA一起在用户空间运行。其中一些功能完全在libutee中实现，一部分在OP-TEE core中实现，由libutee通过系统调用进行访问。

对于TA来说，存在两种实现方式，一种为User mode TA，拥有GlobalPlatform TEE规范中的所有功能特性；另一种为伪TA（Pseudo Trusted Applications）。

* 伪TA（Pseudo Trusted Applications）
直接在OP-TEE core内实现，路径core/arch/arm/pta。伪TA可以理解为GlobalPlatform TA Client API封装的内核service。伪TA的作用是实现某些特殊安全服务。由于伪TA不会去静态链接libutee，因此不会按照GlobalPlatform Core Internal API与内核进行交互，而是完全按照OP-TEE core的接口定义进行交互。
由于伪TA与OP-TEE core拥有相同权限，因此不适用于功能复杂的TA。在大多数情况下，一个User mode TA都是首选。

* User mode TA 当REE中的客户端程序通过特定UUID请求TA服务时，用户空间TA会被加载。它运行在OP-TEE core更低一级的特权级别，即在Trust world用户空间运行。它完全遵循Globalplatform Core Internal API。它又能够细分为三种子类型：

    1、安全存储TA： 这些TA存储在安全存储中。所有安装TA的描述信息被放入一个数据库，并在Normal World REE文件系统中作为一个单独的加密文件存储。在TA被加载前，它首先需要被安装，安装过程可以在初始化部署时完成，也可以后期加载。测试程序xtest可以将一个安全存储TA进行安装`xtest --install-ta`

    2、REE文件系统TA： 这些TA是明文的签名ELF文件，以TA的UUID命令，并以ta为文件名后缀。它们与OP-TEE core隔离，并通过OP-TEE core内的密钥进行签名。由于这些TA被签名了，因此可以保存在REE文件系统内，并由tee-supplicant 复杂将它们加载进入Trust World进行验证和加载。

    3、预装TA： 虽然这些TA在Normal World文件系统内，但不由tee-supplicant加载，在Normal World文件系统初始化之前，直接由TEE core读取并加载。

* TA文件格式：

```
<Signed header>
<ELF>
```

Where `<ELF>` is the content of a standard ELF file and `<Signed header>`
consists of:

| Type | Name | Comment |
|------|------|---------|
| `uint32_t` | magic | Holds the magic number `0x4f545348` |
| `uint32_t` | img_type | image type, values defined by enum shdr_img_type |
| `uint32_t` | img_size | image size in bytes |
| `uint32_t` | algo | algorithm, defined by public key algorithms `TEE_ALG_*` from TEE Internal API specification |
| `uint16_t` | hash_size | size of the signed hash |
| `uint16_t` | sig_size | size of the signature |
| `uint8_t[hash_size]` | hash | Hash of the fields above and the `<ELF>` above |
