---
layout: post
title: 删除扫描器超时进程的脚本
description: "定期清理进程列表"
category: scanner
tags: [scanner]
imagefeature: http://7xwdx7.com1.z0.glb.clouddn.com/5551e1cabd3bb.jpg
comments: true
share: true
---

### 进程清理脚本

在调用扫描器进行批量扫描时，需要对扫描进程的生命周期进行控制。对于长时间运行没有返回的进程，需要定时进行清理避免消耗大量系统资源。

~~~python

import win32com.client
import logging
from colorlog import ColoredFormatter
import os

global logger
formatter = ColoredFormatter(
        "%(log_color)s%(levelname)-8s%(reset)s %(white)s%(message)s",
        datefmt=None,
        reset=True,
        log_colors={
            'DEBUG': 'cyan',
            'INFO': 'green',
            'WARNING': 'yellow',
            'ERROR': 'red',
            'CRITICAL': 'red',
        }
)
logger = logging.getLogger('mylogger')
logger.setLevel(logging.INFO)
ch = logging.StreamHandler()
ch.setLevel(logging.INFO)
ch.setFormatter(formatter)
logger.addHandler(ch)
fh = logging.FileHandler('temp.log')
fh.setFormatter(formatter)
logger.addHandler(fh)
fh.setLevel(logging.INFO)

# get process list and runtime
wmi = win32com.client.GetObject('winmgmts:')
for p in wmi.InstancesOf('win32_process'):
    #print(p.Name+'\t'+str(p.Properties_('ProcessId'))+'\t'+str(int(p.Properties_('UserModeTime').Value)+int(p.Properties_('KernelModeTime').Value)))
    if(((3600*18*10000000) < (int(p.Properties_('UserModeTime').Value)+int(p.Properties_('KernelModeTime').Value))) and (p.Name == 'wvs.exe')):
        # kill outoftime wvs process
        logger.info('killing process: '+p.Name+'\t'+str(p.Properties_('ProcessId')))
        cmd = 'TaskKill /T /F /PID %s' % p.Properties_('ProcessId')
        os.popen(cmd)

~~~

配合AT命令等创建计划任务，我们就能做到定期执行脚本清理超时进程。

`AT 00:00 /every:M,T,W,Th,F,S,Su "python C:\Users\admin\Desktop\tool\processKiller.py"`
