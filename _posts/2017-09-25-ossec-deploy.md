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
``
cd /; tar –exclude client.keys -cf /tmp/ossec.tar var/ossec \ etc/init.d/ossec ‘find etc -name “S[0-9][0-9]ossec”’ \ ‘find etc -name “K[0-9][0-9]ossec”’ 
``

### 远程安装
在目标主机远程安装OSSEC Agent
``
cat /tmp/ossec.tar | ssh root@192.168.65.30 \ “groupadd ossec; useradd -g ossec -d /var/ossec ossec; cd / ; tar -xf - ” 
``

### 服务端添加Agent
在OSSEC Server添加agent IP：
``
/var/ossec/bin/manage_agents
A  (增加一个agent)
agentname   （输入agent名称）
127.0.0.1    (输入agentIP地址)
00X    （输入agent ID）
y      （确认添加agent）
``

### 获取Agent密钥
在OSSEC Server中获取agent的key
``
grep 127.0.0.1 /var/ossec/etc/client.keys > /tmp/agent.key
``

### Agent安装密钥
将key安装到agent端
``
scp /tmp/agent.key root@192.168.65.30:/var/ossec/etc/client.keys 
``

### 启动Agent
在agent端启动ossec
``
service ossec start
chkconfig ossec on
```
