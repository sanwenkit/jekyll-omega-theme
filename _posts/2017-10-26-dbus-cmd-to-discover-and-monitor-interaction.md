---
layout: post
title: DBUS接口信息查看与调用
description: "系统middleware层dbus接口测试基础知识"
category: develop
tags: [dbus, middleware, system]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# DBUS接口信息查看与调用

### 查看系统内的dbus接口与接口描述

在对系统dbus调用的研究测试中，可以借助工具dbus-send、dbus-monitor来进行信息获取、消息监听、接口调用等工作。

首先在系统中一般存在session、system两个bus,当然也可以通过-bus选项增加一个自定义的总线地址，这个总线地址可以是domainsocket文件，也可以是一个TCP端口。

利用标准接口“org.freedesktop.DBus”的ListNames、ListActivatableNames方法，可以对系统中的所有dbus服务接口进行枚举。

* 列出所有服务接口

{% highlight shell %}

dbus-send --session --print-reply --dest=org.freedesktop.DBus /org/freedesktop
/DBus org.freedesktop.DBus.ListNames

{% endhighlight %}

* 列出活跃的服务接口

{% highlight shell %}

dbus-send --session --print-reply --dest=org.freedesktop.DBus /org/freedesktop
/DBus org.freedesktop.DBus.ListActivatableNames

{% endhighlight %}

标准接口“org.freedesktop.DBus”共有16个method和3个signal，具体内容可以查阅官方文档。

在获取所有接口列表后，需要查看每个接口中对应的method（方法）和signal（信号）。这个操作可以通过标准接口“org.freedesktop.DBus.Introspectable”来完成。

* 列出interface下的method和signal描述

{% highlight shell %}

dbus-send --session --type=method_call --print-reply --dest=com.yunos.xxx /
org.freedesktop.DBus.Introspectable.Introspect

{% endhighlight %}

通过以上命令，我们可以完整获取系统中的dbus interface、method、signal。

### dbus接口调用消息的监听

直接通过命令dbus-monitor可以对系统中的dbus调用消息进行监听，此外可以加入一些约束条件在监听特定接口的交互。示例命令如下：

{% highlight shell %}

dbus-monitor "type='signal',interface='com.yunos.xxx'"

{% endhighlight %}

此外，可以通过命令kdbus-monitor对与内核交互的dbus调用进行监听。

### dbus接口的主动调用

当需要调用一个dbus接口的特定方法时，可以采用如下命令：

{% highlight shell %}

dbus-send --session --dest=com.yunos.xxx  --type=method_call  --print-reply  --reply-timeout=20000  /com/yunos/xxx  com.yunos.xxx.SetActive  boolean:true

{% endhighlight %}

以上命令的意义为向com.yunos.xxx接口下的SetActive方法传入一个true的布尔值参数。

### 将本地dbus接口映射到TCP端口

在一些场景中，为了便于测试，我们可以将本机系统的dbus接口映射为一个TCP端口对外暴露。从而可以通过远程网络对系统的功能接口进行调用，便于各种自动化工具的介入使用。

端口映射命令记录如下：

{% highlight shell %}

socat TCP-LISTEN:1234,bind=0.0.0.0,reuseaddr,fork, UNIX-CLIENT:/var/run/dbus/system_bus_socket

adb forward tcp:1234 tcp:1234

{% endhighlight %}

当然这里使用了支持arm移动端的socat，如果系统没有自带的话可以adb push到文件系统中。
