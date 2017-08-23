---
title: 'pyspider[二]启动函数run'
date: 2017-03-05 18:44:41
categories: 爬虫
tags:
  - pyspider
  - python
---

`run.py`是整个pyspider的入口文件, 通过作者画的架构图可以看出, 整个框架主要有三部分:

- 调度器(scheduler)
- 抓取器(fetcher)
- 解析器(processor)

三个部分独立性很好, 互相之间通过消息队列连接, 比较灵活

`pyspider`如果不指定数据库文件和消息队列, 默认使用`sqlite`数据库, 使用python内置的`Queue`做队列

# 常用命令

- 启动所有组件
```
pyspider all
```

- 使用`config.json`启动所有组件
```
pyspider -c config.json all
```

- 启动单个文件
```
pyspider -c config.json scheduler/fetcher/processor/webui
```
整个文件都使用了`click`这个模块, 可以方便的捕获在终端输入的参数

<!--more-->

# connect_rpc函数

使用rpc协议来传输数据
关于该协议的参考资料

- https://docs.python.org/2/library/xmlrpclib.html
- http://www.cnblogs.com/coderzh/archive/2008/12/03/1346994.html

简单理解: 把数据定义为xml格式, 通过http协议进行远程传输
```python
# 客户端连接
def connect_rpc(ctx, param, value):
    """使用rpc协议来传输数据"""
    if not value:
        return
    try:
        from six.moves import xmlrpc_client
    except ImportError:
        import xmlrpclib as xmlrpc_client
    return xmlrpc_client.ServerProxy(value, allow_none=True)
```

# cli函数

cli函数主要完成的功能有

- 创建并连接`projectdb`, `taskdb`, `resultdb`三个数据库
```python
for db in ('taskdb', 'projectdb', 'resultdb'):
    """主要创建了三个数据库"""
    if kwargs[db] is not None:
        continue
    if os.environ.get('MYSQL_NAME'):
        kwargs[db] = utils.Get(lambda db=db: connect_database(
            'sqlalchemy+mysql+%s://%s:%s/%s' % (
                db, os.environ['MYSQL_PORT_3306_TCP_ADDR'],
                os.environ['MYSQL_PORT_3306_TCP_PORT'], db)))
    ......
```

- 连接队列
创建了5个队列:
  - newtask_queue
  - status_queue
  - scheduler2fetcher
  - fetcher2processor
  - processor2result

```python
for name in ('newtask_queue', 'status_queue', 'scheduler2fetcher',
             'fetcher2processor', 'processor2result'):
    if kwargs.get('message_queue'):
        kwargs[name] = utils.Get(lambda name=name: connect_message_queue(
            name, kwargs.get('message_queue'), kwargs['queue_maxsize']))
    else:
        kwargs[name] = connect_message_queue(name, kwargs.get('message_queue'),
                                             kwargs['queue_maxsize'])
    ......
```
不管是运行所有组件还是运行单个组件, 依靠`click`模块, `cli`函数都会在运行组件之前运行,
可以说cli这个函数做的都是pyspider运行之前的准备工作, 准备好各种需要的数据库, 队列等等.


# all函数

比较常用的命令`pyspider all`开启后, 依赖`click`模块的`click.group`装饰器, 整个执行流程是这样的:

- 启动`main`函数
- `main`里面启动`cli`函数, 数据库等一系列基本信息连接好
- 开始运行`all`函数中的内容

在`all`函数中, 依次调用以下函数`phantomjs`, `result_worker`, `processor`, `fetcher`, `scheduler`, 最后调用`webui`, 也就是去启动了这几个组件
```python
......
# phantomjs, 默认没有代理为None
if not g.get('phantomjs_proxy'):
    phantomjs_config = g.config.get('phantomjs', {})
    phantomjs_config.setdefault('auto_restart', True)

    # 在这里把 phantomjs 函数启动起来
    threads.append(run_in(ctx.invoke, phantomjs, **phantomjs_config))
    time.sleep(2)
    if threads[-1].is_alive() and not g.get('phantomjs_proxy'):
        g['phantomjs_proxy'] = '127.0.0.1:%s' % phantomjs_config.get('port', 25555)

# result worker
result_worker_config = g.config.get('result_worker', {})
for i in range(result_worker_num):
    # 把 result_worker 加入进来启动
    threads.append(run_in(ctx.invoke, result_worker, **result_worker_config))

# processor
processor_config = g.config.get('processor', {})
for i in range(processor_num):
    # 启动 processor
    threads.append(run_in(ctx.invoke, processor, **processor_config))

# fetcher
fetcher_config = g.config.get('fetcher', {})
fetcher_config.setdefault('xmlrpc_host', '127.0.0.1')
for i in range(fetcher_num):
    threads.append(run_in(ctx.invoke, fetcher, **fetcher_config))

# scheduler
scheduler_config = g.config.get('scheduler', {})
# 这里使用了 xmlrpc
scheduler_config.setdefault('xmlrpc_host', '127.0.0.1')
# 启动scheduler
threads.append(run_in(ctx.invoke, scheduler, **scheduler_config))

# running webui in main thread to make it exitable
webui_config = g.config.get('webui', {})
webui_config.setdefault('scheduler_rpc', 'http://127.0.0.1:%s/'
                        % g.config.get('scheduler', {}).get('xmlrpc_port', 23333))
ctx.invoke(webui, **webui_config)
```

注意在这几个组件中, `fetcher`, `scheduler`和`webui`涉及到了`xmlrpc`, __但是只有`scheduler`使用了该服务__
最后启动`webui`, 配置文件使用了调度器的`xmlrpc`的配置


# webui

webui使用了Flask, 首先设置peojectdb, taskdb, resultdb的操作
```python
# fetcher rpc
if isinstance(fetcher_rpc, six.string_types):
    import umsgpack
    fetcher_rpc = connect_rpc(ctx, None, fetcher_rpc)
    app.config['fetch'] = lambda x: umsgpack.unpackb(fetcher_rpc.fetch(x).data)
else:
    # get fetcher instance for webui
    fetcher_config = g.config.get('fetcher', {})
    webui_fetcher = ctx.invoke(fetcher, async=False, get_object=True, no_input=True, **fetcher_config)

    app.config['fetch'] = lambda x: webui_fetcher.fetch(x)
```
以上代码是`webui`使用`fetcher`

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
连接`调度器的xmlrpc`, 最后启动`flask`

所以整个`app.config`包括:

- taskdb
- projectdb
- resultdb
- cdn
- max_rate
- max_burst
- webui_username
- webui_password
- need_auth
- process_time_limit
- queues, 连接创建的5个队列
    - newtask_queue
    - status_queue
    - scheduler2fetcher
    - fetcher2processor
    - processor2result
- fetch, 是`Fetcher`类里面的`fetch`函数
- scheduler_rpc, 值为: xmlrpc_client.ServerProxy(value, allow_none=True)





# scheduler函数

该函数主要和调度器有关, 该函数的参数可以定义:

- inqueue-limit: 每个项目任务队列中的最大任务数量
- delete-time: 项目的删除时间, 默认24小时, 可以改这里缩短
- active-tasks: 激活的任务书, 默认100
- loop-limit: ?????
- fail-pause-num: 任务暂停数量, 默认为10, 就是当启动的爬虫遇到错误超过10次时, 默认显示状态会改为`PAUSE`
- scheduler-cls: 调度器类, 默认为`pyspider.scheduler.ThreadBaseScheduler`
- threads: 和调度器类相关, 默认4

该函数会实例化`调度器类(pyspider.scheduler.ThreadBaseScheduler)`, 调度器类实例化需要的参数有:
```python
kwargs = dict(
    taskdb=g.taskdb,
    projectdb=g.projectdb,
    resultdb=g.resultdb,
    newtask_queue=g.newtask_queue,	 # 新任务队列
    status_queue=g.status_queue,	   #  状态队列
    out_queue=g.scheduler2fetcher,	 # 输出队列: scheduler -> fetcher, 给抓取器发任务
    data_path=g.get('data_path', 'data')
)
......
scheduler = Scheduler(**kwargs)
```

```python
......
if xmlrpc:
    utils.run_in_thread(scheduler.xmlrpc_run, port=xmlrpc_port, bind=xmlrpc_host)
......
```
这里运行了`xmlrpc`的`server端`, 最后`scheduler`开始运行

