---
layout: post
title: 系统自带端口转发命令小结
description: "记录一下便于虚拟机配置"
category: config
tags: [端口转发]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

###  Windows平台

开启端口转发功能:

`netsh interface ipv6 install`

查看端口转发配置：

`netsh interface portproxy show v4tov4`

创建一条转发规则：

{% highlight shell %}

netsh interface portproxy add v4tov4 listenaddress=192.168.1.1 listenport=2333 connectaddress=192.168.1.100 connnectport=22

{% endhighlight %}

删除一条转发规则：

{% highlight shell %}

netsh interface portproxy delete v4tov4 listenaddress=192.168.1.1 listenport=2333

{% endhighlight %}

### Linux平台

将内网ssh端口映射到公网IP中的配置：

对于内网服务器A创建到公网机器B的反向代理：

{% highlight shell %}

ssh -fCNR <port_b1>:localhost:22 usr_b@233.233.233.233

{% endhighlight %}

在公网机器B上创建本地端口转发：

{% highlight shell %}

ssh -fCNL "*:<port_b2>:localhost:<port_b1>' localhost

{% endhighlight %}

利用任意机器登录内网主机A的SSH端口：

{% highlight shell %}

ssh -p <portb2> usra@233.233.233.233

{% endhighlight %}

