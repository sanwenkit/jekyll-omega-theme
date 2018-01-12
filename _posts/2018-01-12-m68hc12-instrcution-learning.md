---
layout: post
title: M68HC12 CPU指令集学习
description: "飞思卡尔硬件基础知识"
category: develop
tags: [M68HC12, soc]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---


### M68HC12 CPU指令集学习

# 基础架构

![cpu-regs](http://7xwdx7.com1.z0.glb.clouddn.com/hc12-cpu-1.png)

![program-model](http://7xwdx7.com1.z0.glb.clouddn.com/hc12-programming-model.png)

* 缓存器

通用的８bit缓存器A和B,用来存储操作数和操作接口，AB缓存器可以组合成一个16bit的double类型缓存器D

* 索引寄存器

两个用于索引的寄存器X和Y，在索引寻址模式下，索引寄存器中的当前值会被加上一个5bit,9bit,16bit的常量来形成一个指令操作的有效地址。第二个索引寄存器通常是在一次计算分布在两个不同的table的情况下使用

* 栈指针

SP保存了当前栈顶的16bit地址，通常SP被初始化为应用程序的第一个指令的地址。栈是由高地址空间向低地址空间进行生长的。

子程序调用时，会自动将返回后的下一条指令地址入栈。通常来说，一个RTS(Return from subroutine)或RTC(Return from call)指令将会在子程序最后被执行。这个指令会从栈顶将这个返回地址加载进PC。

产生中断时，下一条指令的地址会被压栈，所有CPU寄存器的值也会被压栈，PC会转向中断向量的处理程序。

* PC指针

16bit的指针，永远执行即将执行的下一条指令，在一条指令被fetch后，它会自增。

* 条件码寄存器

CCR包括5bit状态位，2bit中断mask位, 1bitSTOP控制位。五个状态位分别是Half carry(H), Negative(N), Zero(Z), Overflow(V), Carry/borrow(C)

* 数据类型

bit，5bit有符号整型，8bit有符号及无符号整型，8bit有2bit二进制编码的十进制数(?)，9bit有符号整型，16bit有符号及无符号整型，16bit有效地址,32bit有符号及无符号整数

5bit有符号整型，9bit有符号整型![cpu-regs](http://7xwdx7.com1.z0.glb.clouddn.com/hc12-cpu-1.png)是在索引寻址模式中

# 指令地址模式

M68HC12 CPU架构下的寻址模式可以总结为如下表格

![addressing-mode1](http://7xwdx7.com1.z0.glb.clouddn.com/hc12-addressing-mode-1.png)

![addressing-mode2](http://7xwdx7.com1.z0.glb.clouddn.com/hc12-addressing-mode-2.png)

* 有效地址

所有的寻址模式，除inherent寻址模式以外，都会形成一个16bit的有效地址。有效地址的计算不需要额外的计算运行过程。

* inehrent寻址模式

这个寻址模式下，指令没有操作数或者所有的操作数都是在CPU寄存器中。这种寻址模式下，CPU不需要访问内存地址来完成指令

* immediate寻址模式

immediate寻址模式中，操作数被包含在指令数据流中，并且被正常的fetch流程加载为一个16bit的地址。使用#来表示immediate寻址模式。

* direct寻址模式

在$0000至$00FF之间的地址空间被成为zero-page，通常可以用来存储一些操作数。由于这个地址段永远以0x00开始，因此只要8bit的低位地址就能表示。因此系统将最常被访问的数据存储来这个地址段来进行性能优化。

* extended寻址模式

这个寻址模式下，指令中提供16bit的内存地址，因此可以用来寻址64k内存映射中的任意地址。

* relative寻址模式

这个寻址模式只在分支类指令中使用。短分支指令包含一个8bit操作码和一个8bit有符号偏移值。长分支指令包含一个8bit的prebyte,一个8bit操作码和一个16bit有符号偏移值。

每个条件分支指令都会对条件码寄存器的特定bit进行检查。如果这些bits满足特定状态,偏移量就会被加上下一个有效内存地址形成一个有效地址，同时控制流跳转到这个地址继续执行，反之则继续执行下一个有效内存地址中的指令。

条件分支指令可以测试内存中的某个byte是否满足特殊状态。可以使用多个不同的寻址模式来访问内存地址。一个8bit的掩码操作数(Mask opcode)被用来测试这些bits，如果内存中的bits满足条件，一个8bit的偏移量就会被加上下一个有效内存地址形成一个有效地址，同时控制流跳转到这个地址继续执行，反之则继续执行下一个有效内存地址中的指令。

有符号的偏移量可以是8bit,9bit,16bit的，8bit的偏移量取值范围为$80(-128)~$7F(127),9bit的偏移量取值范围为$100(-256)~$0FF(255),16bit的偏移量zhi取值范围为$8000(-32768)~$7FFF(32767)。

由于偏移量在分支指令的末尾，使用一个负数偏移量可能导致PC指针继续指向指令操作数，并引发死循环。例如，一个BRA指令长度为2byte,当偏移值为$FE时会导致死循环；一个LBRA指令长度为4byte,当偏移值为$FFFC时会导致死循环。

* indexed寻址模式

indexed寻址模式使用一个postbyte加上0-2个扩展byte，postbyte和扩展byte会做如下任务：
１、确定使用那个index寄存器
２、确定在缓存器中的值是否用作偏移量
３、开启自动的pre或post自增、pre或post自减
４、确定自增或自减的量
５、确定使用的是5bit、９bit、16bit有符号偏移量

下表中是indexed寻址模式的能力以及postbyte格式描述。postbyte在指令描述中被标记为xb。
![indexed-addressing](http://7xwdx7.com1.z0.glb.clouddn.com/hc12-indexed-addressing.png)

所有的indexed寻址模式使用一个16bit CPU寄存器和一些附加信息来构造合法地址。这个地址可以是数据存储的内存地址，也可以是一个指向数据存储内存地址的指针地址。

indexed寻址模式使用postbyte来确定index寄存器（X或Y）,SP或PC作为基础的索引寄存来进一步构造有效地址。一组特殊的指令可以将构造的有效地址加载进索引寄存器来进行进一步运算:
  LEAS(Load stack pointer with effective address)
  LEAX(Load X with effective address)
  LEAY(Load Y with effective address)

5bit常量偏移的indexed寻址模式：这个寻址模式中使用指令postbyte中定义的5bit有符号偏移。这个短偏移量被加上基础索引寄存器(X, Y, SP, or PC)的值来形成有效地址。可以用来寻址基础索引寄存器的-16至15偏移范围。尽管其他indexed寻址模式支持9bit或16bit偏移量，这些模式也需要在指令中加入更多的扩展byte来存储信息。

9bit常量偏移的indexed寻址模式: 这个寻址模式中使用指令postbyte中定义的9bit有符号偏移。这个偏移量被加上基础索引寄存器(X, Y, SP, or PC)的值来形成有效地址。可以用来寻址基础索引寄存器的-256至255偏移范围。

16bit常量偏移的indexed寻址模式: 这个寻址模式中使用指令postbyte中定义的16bit偏移。这个偏移量被加上基础索引寄存器(X, Y, SP, or PC)的值来形成有效地址。这个模式允许访问64k内存地址空间的任意地址。由于系统总线为16bit，因此不需要区分signed或unsigned。

16bit常量间接indexed寻址模式: 这个寻址模式使用16bit偏移加上基础索引寄存器的值形成有效地址，这个地址内包含有指向实际计算地址的指针。使用中括号来表示这种间接寻址。

自动Pre/Post递增或递减寻址模式: 这个寻址模式提供4种方式来自动变化基础索引寄存器中的值。索引寄存器的值会被在一次索引过程之前或之后进行自动的递增或递减。同样，基础索引寄存器可以是(X, Y, SP, or PC)。HC12 CPU允许索引寄存器在-8~-1，1~8之间自增或自减。

{% highlight c %}

　 STAA 1,-SP = PSHA
   STX  2,-SP = PSHX
   LDX  2,SP+ = PULX
   LDAA 1,SP+ = PULA

{% end highlight %}

缓存器偏移的indexed寻址模式: 在这个寻址模式下，有效地址是基础索引寄存器中的值加上缓存器中的值。基础索引寄存器(X, Y, SP, or PC)的值不会改变，缓存寄存器可以是8bit也可以是16bit。

缓存器间接indexed寻址模式: 在这个寻址模式下，有效地址是基础索引寄存器中的值加上缓存器D中的值。这个有效地址指向一个指针，指针的值是实际的内存操作地址。例子：

{% highlight c %}

JMP [D, PC]

{% end highlight %}

* 使用多个寻址模式的指令

一些指令在指向过程中，可能使用不止一种寻址模式。

MOVE指令：　MOVE指令的源地址和目的地址可以使用两种不同的寻址模式。MOVE指令不支持立即数寻址和间接寻址模式。

Bit操作指令: bit操作指令可能使用两个或三个不同的寻址模式。清除bit指令(BCLR)和设置bit指令(BSET)使用一个8bit掩码来决定需要被更改的bit位。这个掩码是通过立即数进行设置的。而操作的内存地址空间可以使用多个不同的寻址模式。同样的指令还有BRCLR(branch if bits cleared)和BRSET(branch if bits set)

* 超出64K的寻址

一些M68HC12设备支持超过64K空间的寻址，扩展的内存系统使用快速的片上逻辑来实现透明切换。扩展内存架构包含了一个8bit的程序页寄存器(PPAGE),从而允许256个16k的memory page来在程序窗口中进行切换，这样提供了多至4M的分页程序内存。

H12 CPU包含一组子程序来处理扩展内存调用以及调用返回，从而简化扩展内存空间的使用。这些指令可以在不支持扩展内存寻址能力的设备中正确运行，从而提供了多平台支持的能力。
