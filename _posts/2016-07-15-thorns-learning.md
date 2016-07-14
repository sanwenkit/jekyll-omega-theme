---
layout: post
title: thorns异步任务队列的初步使用体会
description: "记录thorns框架的学习过程"
category: scanner
tags: [扫描器, thorns]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---
### 背景
在对大批量的网络主机进行扫描测试的过程中，使用异步任务队列对扫描任务进行管理具备分布式水平扩展、高容错性等优势。通过引入thorns框架，可以有效提升扫描程序的性能及稳定性。

### thorns初步安装及原理

thorns框架的初始化安装可以参见项目说明文档
> 项目主页：https://github.com/ring04h/thorns<br />

安装并配置完毕后，开启步骤如下：

* 启动redis服务
`redis-server /etc/redis/redis.conf`

* 利用supervisord开启程序
`supervisord -c /etc/supervisord.conf`

* 利用run.py压入扫描任务
`python run.py 10.10.10.1-10.10.10.255`

通过对框架的初步学习使用，将框架的工作原理总结如下：

* 框架使用redis作为维护任务队列的数据池，所有的worker通过连接redis获取当前任务队列信息、接受任务、并反馈任务结果。

* 框架使用celery对任务进行抽象封装，形成一个具有任务类型、任务参数的一个单独task。

* 框架还搭建了一个用来进行任务执行状态操作和监控的web组件，可以对任务执行情况进行查看，并通过HTTP API新增或删除任务队列中的任务。

我们在这个框架的基础上，只要将自己的扫描脚本抽象为一个扫描任务，就能实现分布式、高可靠性的持续扫描程序。

### 以端口扫描任务为例理解task模型

通对对框架代码的阅读发现，端口扫描的task相关定义代码在task.py中

具体内容如下：

~~~python

@app.task
def nmap_dispath(targets, taskid=None):
    # nmap环境参数配置
    run_script_path = '/home/thorns'
    if taskid == None:
        cmdline = 'python wyportmap.py %s' % targets
    else:
        cmdline = 'python wyportmap.py %s %s' % (targets, taskid)
    nmap_proc = subprocess.Popen(cmdline,shell=True,stdout=subprocess.PIPE,stderr=subprocess.PIPE)
    process_output = nmap_proc.stdout.readlines()
    return process_output

~~~

这里定义了一个celery的task，通过子进程的方式运行wyportmap.py脚本，向脚本传递对应的targets参数，并获得任务运行结果。

celery通过将任务队列信息和结果在redis和mysql中进行维护，其全局配置代码如下：

~~~python

app.conf.update(
    CELERY_IMPORTS = ("tasks", ),
    BROKER_URL = 'redis://127.0.0.1:6379/0',
    CELERY_RESULT_BACKEND = 'db+mysql://usenmae:password@127.0.0.1:3306/dbname',
    CELERY_TASK_SERIALIZER='json',
    CELERY_RESULT_SERIALIZER='json',
    CELERY_TIMEZONE='Asia/Shanghai',
    CELERY_ENABLE_UTC=True,
    CELERY_REDIS_MAX_CONNECTIONS=5000, # Redis 最大连接数
    BROKER_TRANSPORT_OPTIONS = {'visibility_timeout': 3600},
    # 如果任务没有在 可见性超时 内确认接收，任务会被重新委派给另一个Worker并执行  默认1 hour.
    # BROKER_TRANSPORT_OPTIONS = {'fanout_prefix': True},
    # 设置一个传输选项来给消息加上前缀
)

~~~

这样，通过web管理界面及对应的API，我们就实现查看并控制所有队列中的任务状态:

![worker当前工作状态](http://7xwdx7.com1.z0.glb.clouddn.com/thorns_work_status.png)

![task运行状态列表](http://7xwdx7.com1.z0.glb.clouddn.com/thorns_task_list.png)

### 创建自己的扫描task

在理解了thorns框架中task的定义及其工作原理后，我们就可以创建自定义的task并加入队列中进行运行。

例如创建一个对端口未授权访问进行扫描的task，任务定义如下：

~~~python

@app.task
def port_dispath(address, port, service, taskid = None):
        # 命令执行环境参数配置
        run_script_path = '/home/thorns/'
        #run_env = '{"LD_LIBRARY_PATH": "/home/liet/code/git/doom"}'
        if taskid == None:
                cmdline = 'python portScan.py %s %s %s' % (address, port, service)
        else:
                cmdline = 'python portScan.py %s %s %s %s' % (address, port, service, taskid)
        cmd_proc = subprocess.Popen(cmdline,shell=True,stdout=subprocess.PIPE,stderr=subprocess.PIPE)

        process_output = cmd_proc.stdout.readlines()
        return process_output

~~~

同时，portScan.py中可以通过定义不同端口的扫描poc文件实现自定义的扫描规则：

~~~python

scan_rule = {
    "openssl" : {
        "script" : "python exploit/openssl.py -p %(port)s %(address)s",
        "port" : [443,587,465,995,8443],
        "service" : ["https","smtp","pop","imap","https-alt"],
        "banner": "None"
    },
    "snmp" : {
        "script" : "python exploit/snmp_anonymous_access.py %(address)s %(port)s",
        "port": [161],
        "service":["snmp"],
        "banner": "None"
    },
    "nfs":{
        "script" : "python exploit/nfs_anonymous_access.py %(address)s %(port)s",
        "port" : [2049],
        "service":["nfs"],
        "banner" : "None"
    },
    "redis":{
        "script" : "python exploit/redis-check.py %(address)s %(port)s",
        "port" : [6379],
        "service": ["redis"],
        "banner": "None"
    },
    "mongodb":{
        "script" : "python exploit/mongodb-check.py %(address)s %(port)s",
        "port" : [27017],
        "service":["mongodb"],
        "banner" : "None"
    },
    "memcache":{
        "script" : "python exploit/memcached_anonymous_access.py %(address)s %(port)s",
        "port" : [11211],
        "service": ["memcache"],
        "banner": "None"
    },
    "zookeeper":{
        "script" : "python exploit/zk-smoketest/zk-latencies.py --servers \"%(address)s:%(port)s\" --znode_count=1 --znode_size=100 --synchronous --force",
        "port": [2181],
        "service":["zookeeper"],
        "banner" : "None"
    },
    "fastcgi":{
        "script": "python exploit/fastcgi.py %(address)s %(port)s",
        "port" : [9000],
        "service": ["None"],
        "banner" : "None"
    },
    "elastic":{
        "script": "python exploit/elastic.py %(address)s %(port)s",
        "port" : [9200],
        "service": ["None"],
        "banner" : "None"
    },
    "rsync":{
        "script": "python exploit/rsync_anonymous_access.py %(address)s %(port)s",
        "port": [873],
        "service":["rsync"],
        "banner":"None"
    }
}

~~~

这样，通过HTTP API我们就可以压入各种自定义的扫描task，把我们手头的各种POC加入到thorns队列中进行自动化扫描啦。
