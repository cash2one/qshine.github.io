---
title: 'pyspider[三]监控组件webui'
date: 2017-03-15 06:14:22
categories: 爬虫
tags:
  - pyspider
  - python
---

主要说明在前端点击`run`按钮后爬虫动作, 因为一直对前端通过点击来控制后端爬虫很疑惑, 其它主要是Falsk的内容, 基本通过Chrome的tools配合源代码都好理解.

首先要了解`XML-RPC`, 摘自维基百科:
> XML-RPC是一个远程过程调用（远端程序呼叫）（remote procedure call，RPC)的分布式计算协议，通过XML将调用函数封装，并使用HTTP协议作为传送机制, 用法: XML-RPC透过向装置了这个协定的服务器发出HTTP请求。发出请求的用户端一般都是需要向远端系统要求呼叫的软件

在python中使用的样例(改自: http://www.cnblogs.com/coderzh/archive/2008/12/03/1346994.html):
```python
# server端
#!/usr/bin/env python

import SimpleXMLRPCServer

class Test(object):
  	def hello(self):
  		return 'hello world'

obj = Test()

# 绑定23335端口
server = SimpleXMLRPCServer.SimpleXMLRPCServer(('127.0.0.1', 23335))
# 注册
server.register_instance(obj)

print 'listening on 23335'
#　开启服务
server.serve_forever()
```

```python
# client端
#!/usr/bin/env python

import xmlrpclib

server = xmlrpclib.ServerProxy("http://127.0.0.1:23335")
# 调用hello方法
words = server.hello()
print words
```

开启后的client端输出hello world, 然后client关闭, server还在运行


## `webui`和`scheduler`之间的通信

在run.py中的`scheduler`函数有以下代码
```python
if xmlrpc:
    utils.run_in_thread(scheduler.xmlrpc_run, port=xmlrpc_port, bind=xmlrpc_host)
```
这段代码会调用`Scheduler`类的`xmlrpc_run`方法, 把xmprpc的server端服务开起来

在`webui`函数中有
```python
if isinstance(scheduler_rpc, six.string_types):
    scheduler_rpc = connect_rpc(ctx, None, scheduler_rpc)
if scheduler_rpc is None and os.environ.get('SCHEDULER_NAME'):
    app.config['scheduler_rpc'] = connect_rpc(ctx, None, 'http://%s/' % (
        os.environ['SCHEDULER_PORT_23333_TCP'][len('tcp://'):]))
elif scheduler_rpc is None:
    app.config['scheduler_rpc'] = connect_rpc(ctx, None, 'http://127.0.0.1:23333/')
else:
    app.config['scheduler_rpc'] = scheduler_rpc
```
这段代码正是在开启xmlrpc的`client`端, 连接server. 然后赋值`app.config['scheduler_rpc'] = scheduler_rpc`, 此时scheduler_rpc就可以调用server端的所有方法了

<!--more-->

## 创建项目save和更改项目状态后的触发动作

前端创建项目后首先会把脚本信息保存到数据库, 然后调用绑定到app上的`rpc`, 调用后端`xmlrpc_run`函数里面的`update`函数, 执行`self._force_update_project = True`, 这样在`self._update_projects()`中就会更新, 然后输出日志, 同样, 在首页更改`group`, `status`, `rate`都会触发rpc的update

当有很多项目的时候只更改一个项目, 会输出该项目update的信息, 是检查数据库的更新时间来做
```python
for project in self.projectdb.check_update(self._last_update_project):
    ......
```

## 点击run运行爬虫的过程

点击`run`后, 首先会从projectdb中读出该项目的信息, 然后构造一条新任务如下:
```python
newtask = {
        "project": project,
        "taskid": "on_start",
        "url": "data:,on_start",
        "process": {
            "callback": "on_start",
        },
        "schedule": {
            "age": 0,
            "priority": 9,  # default priority
            "force_update": True,
        },
    }
try:
    # 这里传给scheduler
    ret = rpc.newtask(newtask)
......
```
把这个任务通过绑定的`rpc`服务发送到`newtask_queue`队列, 然后`scheduler`会之后后续操作


这一块的逻辑不是很复杂, 需要知道的是`webui`通过`scheduler_rpc`来跟爬虫进行通信
