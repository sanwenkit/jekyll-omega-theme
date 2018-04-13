---
layout: post
title: OP-TEE中的RPMB使用
description: "OP-TEE学习材料"
category: develop
tags: [Trustzone, OPTEE, RPMB]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# RPMB安全存储

### 介绍
OP-TEE默认实现了RPMB安全存储功能，可以通过CFG_RPMB_FS=y来开启。在该功能开启时，TA通过标示TEE_STORAGE_PRIVATE_RPMB进行安全存储使用，该功能关闭时，标示变更为TEE_STORAGE_PRIVATE

其总体架构如下图所示：
![rpmb-arch](http://7xwdx7.com1.z0.glb.clouddn.com/RPBM-OPTEE-ARCH.jpg)

### 安全存储API

安全存储API与普通REE文件系统类似，在系统调用与RPMB文件系统间的接口是tee_file_operations结构： tee_file_ops。

RPMB文件系统分区分为三个部分：
* 开头128bytes预留存储分区信息（struct rpmb_fs_partition）
* 从偏移512bytes来时，为FAT（File Allocation Table）。它是一个struct rpmb_fat_entry元素组成的数组，每个文件对应一个元素。随着加入文件系统中的文件增加，FAT列表也会动态增加。每个entry元素都保存来文件数据起始地址、文件大小、文件名等信息。
* 文件数据从RPMB分区末尾开始向前填充。

分区内的空间通过通用分配函数 tee_mm_alloc() 和 tee_mm_alloc2() 进行分配。
所有文件操作都是原子性的。该特性通过以下特性保证：
* 对RPMB分区块数据的写操作由EMMC规范保证是原子性的
* 在数据写入确认完成后，才开始进行FAT entry更新
* 对于一个文件内容的更新，只有在修改数据量不大于“reliable write block count”的前提下才会进行直接更新，否则一个新的文件会被创建。同时，如果文件大小需要扩大，也会重新创建新文件。

### 设备访问

在OP-TEE中，没有EMMC控制器驱动。所有的设备操作都需要通过Normal World。这些操作请求由TEE-supplicant进程来处理，并通过Linux内核IOCTL接口来访问设备。TEE-supplicant也拥有一个模拟模式来实现一个虚拟的RPMB分区以满足测试目的。

RPMB操作包括如下几个：

* 获取设备信息（分区大小，reliable write block count）
* 设置security key。这个key用于认证，它与安全存储key（SSK Secure Storage Key）不同，后者用于加密数据。和SSK类似，security key也通过一个设备唯一指纹演化而来。目前，通过tee_otp_get_hw_unique_key()来生成RPMB security key。
* 读取write counter值，write counter用于读写请求时HMAC的计算。这个值在初始化时被读取，并存储在tee_rpmb_ctx结构： rpmb_ctx->wr_cnt
* 读写块数据

RPMB操作由文件系统层面的请求发起，用于请求和返回数据的内存区域分配在共享内存中，使用thread_optee_rpc_alloc_payload()创建。该内存区域被放入一个TEE_RPC_RPMB_CMD消息传递到Normal World，外部为thread_rpc_cmd()机制。大多数的RPMB请求与返回数据使用JEDEC eMMC规范定义的数据帧格式。

### 加密

RPMB通过块加密保护文件系统数据。使用算法为AES-128 CBC with ESSIV。

* 在OP-TEE初始化时，一个128bit AES SSK会通过演化算法从HUK（Hardware Unique Key）生成。它被存储在Trust World内存中并永远不会落盘。通过SSK和TA的UUID，一个TSK（Trusted Application Storage Key）会被通过演化算法生成。

* 对于每个文件，一个128bit的FEK（File Encryption Key）会在文件创建时被随机生成，并通过TSK进行加密存储在FAT中文件对应的entry内。

* 文件数据的每256byte会被分块并通过CBC模式加密。初始向量IV通过ESSIV算法获取，实际为FEK的hash值。这允许对文件中的任意块进行直接访问。

当无RPMB分区，采用REE文件系统进行安全存储时，SSK、TSK、FEK的逻辑均一致。AES CBC块加密仅用于RPMB，REE文件系统使用AES-GCM加密。FAT列表没有加密。
