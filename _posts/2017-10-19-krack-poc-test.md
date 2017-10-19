---
layout: post
title: KRACK WPA2协议漏洞测试记录
description: "学习wifi相关协议底层及linux命令使用"
category: scanner
tags: [poc, wifi, wpa2]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# KRACK WPA2协议漏洞测试记录

### KRACK漏洞背景

有研究人员发现WPA2协议存在逻辑层面的漏洞，可以通过构造恶意的数据包来控制通信的加密密钥，从而实现流量的解密与中间人劫持。

漏洞具体信息可以参考：https://www.krackattacks.com/

漏洞披露后，作者的github repo中也提供了一个FT Handshake模式下的POC验证脚本，用来测试一个AP是否存在key reinstall漏洞。POC脚本下载地址为：https://github.com/vanhoefm/krackattacks-test-ap-ft

### WIFI命令行基础
由于前期对wifi安全测试接触不多，因此通过记录这个脚本的测试过程，将linux中wifi相关的命令行工具使用进行记录。

通过命令行工具配置wifi无线网卡连接热点的步骤可以总结如下：

* 关闭系统的wpa_supplicant进程

{% highlight shell %}

ps -aux | grep wpa_supplicant

sudo  kill -9 wpa_supplicant_pid

{% endhighlight %}

* 查找AP热点的SSID

{% highlight shell %}

iw wlo1 scan | grep SSID

{% endhighlight %}

查看列出的SSID列表中有没有需要测试AP

* 配置连接conf文件

{% highlight shell %}

wpa_passphrase <SSID> <passwd> > network.conf

{% endhighlight %}

如果后续需要通过wpa_cli命令进行进一步操作，可以继续编辑network.conf。在第一行加入：

{% highlight shell %}

ctrl_interface=/var/run/wpa_supplicant

{% endhighlight %}

* 尝试连接

{% highlight shell %}

wpa_supplicant -Dnl80211 -iwlo1 -c network.conf

{% endhighlight %}

如果配置都正确的话，就会成功与AP建立连接。可以在AP的管理界面中看到连接设备。

* wpa_cli进一步操作

{% highlight shell %}

sudo wpa_cli -i wlo1

{% endhighlight %}

成功连接后，我们可以通过status命令查看当前wifi连接状态；通过scan_results命令查看当前wifi连接列表；通过roam MAC_addr命令重新连接到其他AP。

### POC脚本测试

在完成wifi连接AP的基础配置后，我们可以调用POC脚本建立一个wifi连接。此时，POC脚本将会为当前连接创建一个MITM proxy。在测试机中，通过ifconfig可以看到增加了一个wlo1mon00-00的网卡。

![mitm_interface](http://7xwdx7.com1.z0.glb.clouddn.com/mitm-proxy.png)

随后，我们通过arping命令让wifi连接保持数据交互，便于POC脚本工作。我们可以按照如下命令中显示的操作说明完成相关验证测试工作。
{% highlight shell %}

python krack-ft-test.py --help

{% endhighlight %}

由于我实际的测试设备并不支持FT handshake模式，因此如果将key_mgmt=FT-PSK配置在network.conf中，是无法成功连接AP的。如果删除该配置，那么通过wpa_cli命令查看wifi连接状态，实际的key_mgmt为WPA2-PSK模式。

在这个模式下，POC脚本发送的key reinstall数据包并不会被识别，因此也无法完成对AP的漏洞利用验证。运行POC脚本的劫持如下所示：

![POC_run](http://7xwdx7.com1.z0.glb.clouddn.com/wifi-krack.png)
