---
layout: post
title: AUTOSAR架构学习
description: "AUTOSAR安全相关实现"
category: develop
tags: [AUTOSAR, SecOS, HSM]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---




# AUTOSAR架构学习

### 架构
AUTOSAR架构使用三层软件平台：
* 应用层 应用层包含了在ECU中运行的实际应用，例如发动机控制或定速巡航软件等。应用层是唯一的其功能性不被AUTOSAR框架定义的软件层。
* 运行时环境层  第二层为运行时环境（RTE Run Time Environment）层， 它是一个在AUTOSAR操作系统中介于基础软件层（Basic Software Layer）和应用层之间的层级。
* 基础软件层（BSW）  基础软件层包含AUTOSAR操作系统，对应于诊断、内存管理、通讯、系统服务等功能。

这三层软件在MCU上运行，基础软件层可以进一步分解为4个层级，具体结构如下图所示：
![autosar-arch](http://7xwdx7.com1.z0.glb.clouddn.com/AUTOSAR-layer.jpg)

### 应用层

应用层包含AUTOSAR软件中的功能组件，这些功能应用是硬件独立的，并能够在不同ECU中进行移植。

### 运行时环境层

运行时环境层负责提供功能应用与基础软件模块进行交互的接口，运行时环境层是基于特定ECU的，因为它需要基于ECU规格来访问正确的通讯通道。

### 基础软件层

基础软件层向应用层提供系统服务以保证应用能正确实现功能。基础软件自身不会进行任何功能任务。在上图中，基础软件层又分为如下4层

* MCU抽象层（MCAL） 基于特定MCU提供BSW其他层访问MCU的标准化接口
* ECU抽象层 ECU抽象层提供了一个MCU和硬件独立的设备、外设接口
* 服务层 服务层是BSW中最高层，为应用、运行时环境、BSW其他模块提供基础服务。这些服务是MCU和硬件独立的，包括：
    操作系统功能
    诊断协议和NVRAM管理
    加解密服务
    车载网络通讯（CAN、LIN、Flexray）

* 复杂驱动层 复杂驱动层负责连接硬件和运行时环境。它提供了整合特定功能设备驱动的能力，例如高实时要求的设备或没有采用AUTOSAR架构的设备。

### SecOS（Secure Onboard Communication）

在AUTOSAR 4.2版本中，引入了一个名为SecOS的模块。这个模块通过对关键数据交互增加安全认证机制来提升安全性。这个模块能够和AUTOSAR当前通讯系统进行快速整合并高效利用资源。

在下图中，SecOS使用CSM（Crypto Service Manager - 软件加解密）或CAL（Crypto Abstraction Layer - 硬件加密）来提供加解密功能。SecOS模块使用MAC机制或数字签名机制来保证关键数据传输时对ECU身份和数据真实性的验证。
此外，SecOS模块来与AUTOSAR通讯系统连接，SecOS模块在软件层，它直接与PDU（Protol Data Unit）路由器连接来处理PDU的安全信息。当PDU路由器接口到一个配置了SecOS的消息，它首先将该消息路由到SecOS模块。SecOS完成安全检查后再返回PDU路由器进行下一步处理。

### CAL

CAL包含加解密库模块CPL（Cryptographic Primi- tive Library），并将其封装为一个标准化接口提供给其他AUTOSAR软件使用。这个层级架构允许对底层CPL实现进行更换而不影响上层功能代码。由于不同厂商都有自己的加解密库，因此这个设计允许在不同的加解密库中进行切换

### HMAC

HMAC是一种使用密码学hash函数及一个密钥确保消息完整性、可认证性的MAC校验机制，使用的HASH函数可以是MD5、SHA-1、SHA-256等。目前而言，至少需要SHA-256、SHA-512散列函数才能满足安全要求等级。

### SecOS中的消息校验

SecOS模块的一个重要功能是在PDU级别提供关键数据交互的实际认证机制。根据SecOS规格说明，对称或非对称的认证与完整性保护都是支持的。
SecOS使用CSM或CAL来提供加解密功能。确保数据完整性和可认证的方式可以是MAC或数字签名形式。为来验证来自合法ECU的消息，SecOS使用了一个刷新值，可以是一个计数器counter或者一个timestamp。当ECU完成一次消息发送或接口后，这个刷新值会被更新。
SecOS定义了两种类型的数据包，称为认证PDU（Authentic PDU）、安全PDU（Secure PDU）。认证PDU是指需要SecOS进行保护的消息原文，安全PDU中包括消息原文、刷新值、MAC值三部分。其结构如下图所示：
![Secure-PDU](http://7xwdx7.com1.z0.glb.clouddn.com/Secure-PDU.jpg)

当SecOS接收到一个认证PDU时，它负责加入刷新值并将PDU传入CAL模块计算MAC值。全部完成后，MAC值也被附加在消息末尾，完成安全PDU的组装。随后，该消息就能够在CAN网络中进行传输。当接收ECU收到安全PDU时，继续将其拆分为PDU原文、刷新值、MAC值三部分。随后ECU再次利用加密功能和密钥计算MAC值进行比较，一旦两次MAC值一致且刷新值保持正确，这个PDU就通过了认证并可以被应用进行处理。相关发送与认证逻辑如下图所示：
![Secure-PDU](http://7xwdx7.com1.z0.glb.clouddn.com/Secure-PDU-Verify.jpg)
