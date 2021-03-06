---
layout: post
title: 满足ASIL与ISO26262要求的系统架构
description: "QNX系统安全相关文档学习"
category: develop
tags: [ISO26262, ASIL]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# 满足ASIL与ISO26262要求的系统架构

### 背景
与其将汽车认为是一个带有部分电子功能模块的机械机器，不如将其描述为一个为了个人交通出行服务的分布式计算系统。目前工程师们正尝试将车内所有ECU的信息整合到一个中心：
一部分是高安全性的车内系统；一部分是娱乐应用系统，难以被证明是安全的。这些完全不同的系统需要在同一个CPU上运行，带来来安全风险。同时，智能网联汽车使得任何车内系统理论上都可能被外部所接触，从而导致了大量风险。例如OTA软件、固件更新等，引入了新的安全漏洞。


### 交互影响

如果我们考虑到当今车内系统的数量，以及他们所执行的不同任务，那么可以判断并不是所有系统都需要满足相同的ASIL标准。比如，一个多媒体播放器的稳定性不一定需要和车道偏移告警系统保持一致。唯一的要求在于，多媒体播放器的交互接口不要干扰到这些高安全要求的相关系统。

如果一个高安全需要的系统与不可靠的系统进行交互，那么就会破坏其ASIL标准规范。那么很重要的问题在于，有时它们共享了同一个CPU和内存硬件。

例如一个ASIL B级别的通讯模块和一个ASIL D级别的自动巡航模块进行交互，通讯模块为自动巡航模块提供数据。可能出现的情况是，通讯模块快速地广播大量消息（babbling），导致自动巡航模块所有资源耗费在接收消息上，无法正常执行功能。而这个通讯模块可能还会和一个更低等级的多媒体播放器交互，被这个不安全的程序不断占用共享内存用来缓存音乐。

一些不良的交互影响如下：

* 资源耗尽：对系统资源不正确的使用，例如文件描述符fd、mutex死锁、内存等，一个进程可能将其他进程的资源耗尽。

* 运行时间耗尽：  一个进程可以通过不断占用CPU来影响其他进程的响应实时性。

* 非法的内存访问：  一个进程可能会对另一个进程的私有内存地址进行访问。如果是读操作，可能导致敏感信息泄露，如果是写操作，将直接造成严重后果

* 数据伪造：  一个进程可以向其他进程提供伪造的数据，来触发其他进程出现异常行为

* 拒绝服务攻击：  一个进程可能会以极高的频率向其他组件发送消息，从而导致拒绝服务攻击

* 死锁：  进程A等待使用资源X，并占用锁定了资源Y；进程B等待使用资源Y，并占用锁定了资源X；那么它们将无法继续执行，导致循环死锁。

### 构建一个弹性系统

没有一个设计是不会犯错的，所有的设计都会有其特有的限制和薄弱环节。幸运的是，设计只是防御阵地的其中一道防线。一些其他的技术，例如应用验证、预先设计、静态分析等，被用来帮助解决问题。通过构建一个弹性系统可以满足ISO26262的要求：对组件进行隔离，对不同ASIL等级组件的交互影响进行隔离。

### 缺陷、错误和失败

对于一个安全操作系统的设计，识别和接收系统缺陷是非常基础的。对于一个现代化的车内系统来说，大约包含1亿行软件代码。而波音787飞机中的软件代码量只包含650万行。ISO26262的作者对此表述更加直接精确：
由于系统复杂性的增加，出现系统性风险和随机硬件异常的情况会越来越多。ISO26262包含了一些明确的需求和流程来避免这些风险。ISO 26262强调预先识别并隔离缺陷。如果一个包含不同ASIL等级组件的系统，能够把每个组件隔离，并在组件内处理异常缺陷，那么就不会影响到其他组件或整个系统。然而，很多错误并不会导致当前组件出现异常（例如对其他进程内存空间的数据写入），反而会对其他组件造成影响从而使得其他组件或整个系统失败。

因此一个弹性系统应该包含以下特性：
* 由多个满足其自身安全要求（ASIL等级）的稳定组件组成
* 组件之间被良好地隔离，防止一个组件对其他组件造成影响

当前确保隔离的策略包括虚拟化和微内核架构。

### 虚拟化

一般而言，车内系统的理想虚拟化方案是指Type1 hypervisor，即不同的操作系统运行在虚拟化层。而不是Type 2 hypervisor，比如一个guest OS嵌套在另一个guest OS内。hypervisor可以满足ISO 26262的组件隔离要求，两个客户OS运行在虚拟化层，并相互隔离。一个OS运行安全相关的组件，另外一个OS运行infotainment系统。

虚拟化的优势：
1、通过虚拟化，将dirty world与secure world相隔离
2、对于非技术型的管理审计人员来说，虚拟化方案更容易解释

虚拟化的一些问题：
有很多原因解释虚拟化为什么通常不是最佳解决方案，这些原因包括技术上和经济上的。

可见性：很多虚拟化层的功能依赖于硬件，这些硬件虚拟化支持往往是很复杂的，例如内存管理单元MMU。同时，MMU技术经历了很多年来不断改进，片上虚拟化技术支持相对来说是很新的。如果一个影响片上软件可靠性或隔离性的BUG出现，要么需要替换硬件，要么需要对hypervisor层实现甚至secure world OS实现进行修改，所有这些措施都是非常耗费资源的。

性能：虚拟化技术在芯片上增加了软件运行的层级，新的硬件技术已经消除了虚拟化层带来的性能延迟，但是虚拟化层本身会影响关键组件的性能表现。特别是要求高硬件交互带宽的设备 ，例如GPU等。尽管在一个芯片上的多个功能不需要共享同一个显示器，但是一个低安全要求的head unit可能会与高安全要求的碰撞告警系统共享同一个高带宽GPU资源。

可靠性：虚拟化解决的隔离问题，确保不同ASIL等级的组件不会对对方造成影响。但是，它并不能解决稳定性（包括可用性、可靠性），这些责任仍然需要GuestOS承担。

复杂性：hypervisor需要兼容不同的Guest OS，因此相对于运行单一OS的系统，带有虚拟化层的系统是一个更加复杂的方案。
双系统方案明确要求Dirty world OS不能对secure OS（ASIL高等级）造成影响，同时需要支持所有要求的特性（例如图形等）。这样对所有OS能力的支撑会减弱虚拟化隔离的有效性。

粒度：虚拟化的隔离是在OS层面的，它不能隔离每个Guest OS内部运行的组件。Secure world的OS虽然不用担心来自Dirty world组件的影响，但仍然需要对每个组件相互隔离。

成本：双系统方案不仅仅包括一个虚拟化层和两个操作系统，它还会带来一大堆基础设施：开发环境、工具链、开发人力资源等。两支不同的开发团队需要完成不同ASIL等级的功能。而如果使用同一个操作系统，基础设施团队的专业性会更强，减少工程消耗。

开销：一个虚拟化层的软件license和一个infotainment操作系统的license费用是相当的，因此采用虚拟化方案设计虽然缩减了硬件开销，但是它并不能降低软件整体成本。更重要的是，一个车内系统的安全责任并不是在车辆下线时就完成了，而是会贯穿整个车辆生命周期。当出现问题时，OEM生产商需要承担责任，因此OEM需要确保其供应商随时准备维护更新系统。而在双系统架构中，需要确保在汽车生命周期内，维护两个OS系统。

### 隔离和稳定性

无论是否使用虚拟化来区分Secure World和Dirty World，一个ISO 26262系统必须按照以下原则设计：
高安全需求的组件必须满足它们自己的稳定性需求
高安全需求的组件必须防止被其他组件影响，不论是Dirty World还是Secure World的

OS架构
在ISO26262系统中，OS架构是决定性的。因为它是整个系统稳定性的基础，它决定了隔离和保护不同ASIL等级组件的难度和开销，以及达到ASIL等级需求的难度。下表中列出了在嵌入式系统中常见的OS架构，以及这些架构对系统内组件隔离能力的影响

设计                                    优点                                缺点

实时运行  所有组件在同一个内存地址空间运行      高效        组件内的指针错误可能导致其他组件或内核奔溃，导致系统级失败
巨内核  应用运行在独立的进程空间，内核组件和文件系统、协议栈、驱动共享内存      内核与用户空间隔离      内核地址空间内服务的异常会导致系统级失败
微内核     应用、设备驱动、文件系统、网络协议栈都分步在独立内存地址空间，与内核及其他组件隔离       异常不会导致整个系统失败，系统能够重启失败的组件，不同ASIL等级的组件可以在同一个系统中结合      组件间通讯的开销会加大

通过单个微内核系统，可以达成ISO26262的相关稳定性、隔离保护的要求。相关架构如下图：
![microkernel](http://7xwdx7.com1.z0.glb.clouddn.com/microkernel.png)

### 微内核架构对于交互影响的保护

在整个软件生命周期，可以通过很多方法来减少组件之间的交互影响。在OS设计层面，一些可用的特性如下：

资源耗尽： 通过rlimit参数，系统设计者可以保证单个进程或应用不耗尽系统资源。另一种防御方式是，系统可以引入一个检测程序，这个程序可以记录系统正常运行的资源消耗基线。当有进程出现异常资源消耗时，对这些进程进行处理。BMP（Bound multiprocessing）是一个SMP（symmetrical multiprocessing）的高级形式，让设计者将特定的线程树指定在某个特点的核上运行。在一个ISO 26262系统中，一般有双核处理器。核A运行高安全组件的所有线程，核B运行非安全组件的所有线程。

时间耗尽： 调度策略和工具可以被应用到系统中来确保进程满足它们的实时性要求，例如deadline-monotonic scheduling 、rate-monotonic scheduling等。
CPU cycles分片是一个保证所有进程满足它们时间限制的有效方法，它把CPU时间进行分片，确保每个进程或进程组能够获取固定的时间分片，因此不会有进程被其他进程阻断。同时，分片还能保证系统资源不会被浪费，它确保了进程或进程组运行的最小时间避免无谓的调度开销，同时又预设了系统高负载时进程运行的最大分片限制。当然，在其他进程空闲时，高负载进程可以暂时占用分配给它的时间分片。

![cycles-partition](http://7xwdx7.com1.z0.glb.clouddn.com/cycles-partition.png)

非法内存访问： 一个MMU硬件可以防止操作系统中的非法内存访问。如果OS提供这样的特性，MMU会保证进程的内存空间不被其他进程访问。作为一个安全相关系统，这应该是软件架构的必须特性。

数据篡改：对于数据篡改的防御包括校验和、简单备份、数据变换、完整性检查等。校验和或者CRC校验是一种开销最小的方式，但不能修复被篡改的数据。简单备份把数据放在多个地址存储，有时包括远程存储。数据变换是指数据存储在多个地址，并通过不同的形式进行保护。完整性检查确保数据输入在可接受的范围内，对于异常输入会被丢弃。

这些措施都会增加内存、计算能力的使用，但他们对于系统的稳定性和安全性都极大的帮助，因此这些额外的开销是可接受的。

DDOS攻击：  拒绝服务攻击可以通过一个自动检测程序来识别，它监控系统资源和进程活动。在出现异常时纠正这些异常的进程。另外一种对这类问题有帮助的测试叫做“fuzzing”，可以将恶意输入或异常方式传递给功能模块，尝试发现一些没有被覆盖的系统异常。

死锁： 当相互协作的进程彼此互相等待对方完成操作并释放资源时，它们会发生互相死锁。优先级继承一个防止死锁的有效方式，它通过优先级倒置的方式解决问题。当低优先级进程阻止了高优先级进程工作时，低优先级进程会被阻断来优先完成高优先级任务。
硬件看门狗对于死锁也非常有用，但它们并不能保证100%有效，因为对于那些不希望被踢的进程来说，他们是透明的。然而，软件看门狗或者高可用控制器可以。系统可以允许单独的组件停止或重启来确保不产生系统级的Crash，软件看门狗会对这些恶意的进程进行停止和重启。

