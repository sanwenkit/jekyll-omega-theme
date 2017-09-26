---
layout: post
title: OSSEC部署记录
description: "记录OSSEC部署的一些配置"
category: develop
tags: [develop]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

# OSSEC Agent自动化部署步骤
### 创建安装包
创建OSSEC Agent TAR格式安装包

{% highlight shell %}

cd /; tar –exclude client.keys -cf /tmp/ossec.tar var/ossec \ etc/init.d/ossec ‘find etc -name “S[0-9][0-9]ossec”’ \ ‘find etc -name “K[0-9][0-9]ossec”’ 

{% endhighlight %}

### 远程安装
在目标主机远程安装OSSEC Agent

{% highlight shell %}

cat /tmp/ossec.tar | ssh root@192.168.65.30 \ “groupadd ossec; useradd -g ossec -d /var/ossec ossec; cd / ; tar -xf - ” 

{% endhighlight %}

### 服务端添加Agent
在OSSEC Server添加agent IP：

{% highlight shell %}

/var/ossec/bin/manage_agents
A  (增加一个agent)
agentname   （输入agent名称）
127.0.0.1    (输入agentIP地址)
00X    （输入agent ID）
y      （确认添加agent）

{% endhighlight %}

### 获取Agent密钥
在OSSEC Server中获取agent的key

{% highlight shell %}

grep 127.0.0.1 /var/ossec/etc/client.keys > /tmp/agent.key

{% endhighlight %}

### Agent安装密钥
将key安装到agent端

{% highlight shell %}

scp /tmp/agent.key root@192.168.65.30:/var/ossec/etc/client.keys 

{% endhighlight %}

### 启动Agent
在agent端启动ossec

{% highlight shell %}

service ossec start
chkconfig ossec on

{% endhighlight %}

