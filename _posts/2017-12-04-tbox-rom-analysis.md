---
layout: post
title: Tbox　ROM固件分析
description: "TBOX ROM逆向基础学习"
category: develop
tags: [kernel, debug, qemu]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# Tbox　ROM固件分析

### 硬件基础信息

TBOX硬件型号为MC9S12XEQ512，其硬件Flash空间为512K，RAM空间32K，EEPROM空间36K。数据总线宽度16bit,因此可以在0x0000~0xFFFF这64K的寻址空间进行寻址。

![tbox-ram](http://7xwdx7.com1.z0.glb.clouddn.com/freescale-tbox-ram.jpg)

在单片普通模式中，所有内存资源的映射结构如上图中所示。

* 系统寄存器

从0x0000-0x07FF的2K范围内为寄存器映射区，如I/O端口寄存器、CAN寄存器等。具体寄存器分布后续会进行详细分析。

* EEPROM PAGE映射

从0x0800-0x0BFF的1K范围为EEPROM分页映射区，通过设置寄存器EPAGE选择不同的EEPROM页进行访问

* EEPROM NONE PAGE映射

从0x0C00-0x0FFF的1K范围本来是保留区域，这里被声明为non-paged EEPROM映射区。

* RAM PAGE映射

从0x1000-0x3FFF的12K空间为RAM区，分为三页。这三页中的后两页为固定页（0x2000-0x3FFF），前一页（0x1000-0x1FFF）为窗口区，通过设置寄存器RPAGE来映射到其他分页的RAM。因此，0x1000-0x1FFF这4K大小的空间为RAM PAGE映射区。

* RAM NONE PAGE映射

0x2000-0x3FFF的8K空间为non-paged RAM区域，其中0x2000-0x37CF为RAM区，0x37D0-0x388F为OS_RAM区，0x3890-0x3FFF为CAN_LIB_RAM区。

* FLASH映射区简要说明

从0x4000-0xFFFF的总共48K空间为Flash区，同样分为三页。其中第一页（0x4000-0x7FFF）和第三页（0xC000-0xFFFF）为固定FLASH页，第二页（0x8000-0xBFFF）为窗口区，通过设置PPAGE寄存器，可以映射到各个分页FLASH。

* 第一页FLASH NONE PAGE映射

0x4000-0x7FFF映射定义如下： 0x4000-0x410F定义为STARTUP_CODE分区，0x4110-0x41FF定义为VECTOR_TABLE分区，对应的定义为fmc_bsp_vectors.c: const tIsrFunc VectorTable[] @0x4110， 0x4200-0x5FFF定义为OSVECTORS，保存OSEK Interrupt Vectors， 0x6000-0x7FFF定义为ROM_VAR区，保存ROM中的各种变量值。

第三页固定页并没有被定义使用。

* PAGED EEPROM\RAM\FLASH

项目配置文件prm还定义了各个PAGE的EEPROM、RAM、FLASH对应的数据段和代码段，这样我们对于每个段的数据意义可以通过prm配置文件完整的逆向出来。
