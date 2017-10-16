---
title: 使用Scrapyd部署爬虫
date: 2017-05-23 04:45:29
categories: 爬虫
tags:
  - scrapy
---


Scrapy官方文档中推荐使用Scrapyd来部署爬虫, 它可以通过JSON API来控制爬虫

资料
- [官方文档](http://scrapyd.readthedocs.io/en/stable/overview.html)
- [scrapyd和scrapyd-client使用教程](http://blog.wiseturtles.com/posts/scrapyd.html)

__缺点:__ 并不适合分布式爬虫, 比较适合单机部署的爬虫和调度使用

-------

安装使用模块

```shell
pip install scrapyd
pip install scrapyd-client
```

### 目录结构

```
(xh_venv) ql@ubuntu:~/Test/scrapyd_test$ tree
.
├── scrapy.cfg
└── scrapyd_test
    ├── __init__.py
    ├── __init__.pyc
    ├── items.py
    ├── middlewares.py
    ├── pipelines.py
    ├── settings.py
    ├── settings.pyc
    └── spiders
        ├── baidu.py
        ├── __init__.py
        └── __init__.pyc

2 directories, 11 files
(xh_venv) ql@ubuntu:~/Test/scrapyd_test$ 
```

#### 一. spider内容

先修改`settings.py`中robots协议为False

spider内容如下

```python
# -*- coding: utf-8 -*-
import scrapy


class BaiduSpider(scrapy.Spider):
    name = "baidu"
    allowed_domains = ["baidu.com"]
    start_urls = ['http://www.baidu.com/']

    def parse(self, response):
        for link in response.xpath('//a/@href').extract():
        	print link
```

#### 二. 修改scrapy.cfg

源文件

```
[settings]
default = scrapyd_test.settings

[deploy]
#url = http://localhost:6800/
project = scrapyd_test

```

修改后

```
[settings]
default = scrapyd_test.settings

[deploy:baidu_d]    # 发布名称(随意定)
url = http://localhost:6800/    # 取消注释
project = baidu_p    # 项目名称(随意定)

# 需要用户名和密码的情况
# username = test
# password = test
```

<!--more-->

#### 三. 发布项目

启动`scrapyd`(启动scrapyd的目录会保存整个scrapyd运行期间生成的log, item文件), 此处我在项目根目录下启动

```
(xh_venv) ql@ubuntu:~/Test/scrapyd_test$ scrapyd
2017-05-22T20:03:46-0700 [-] Loading /home/ql/.virtualenvs/xh_venv/local/lib/python2.7/site-packages/scrapyd/txapp.py...
2017-05-22T20:03:46-0700 [-] Scrapyd web console available at http://127.0.0.1:6800/
2017-05-22T20:03:46-0700 [-] Loaded.
2017-05-22T20:03:46-0700 [twisted.scripts._twistd_unix.UnixAppLogger#info] twistd 17.1.0 (/home/ql/.virtualenvs/xh_venv/bin/python 2.7.12) starting up.
2017-05-22T20:03:46-0700 [twisted.scripts._twistd_unix.UnixAppLogger#info] reactor class: twisted.internet.epollreactor.EPollReactor.
2017-05-22T20:03:46-0700 [-] Site starting on 6800
2017-05-22T20:03:46-0700 [twisted.web.server.Site#info] Starting factory <twisted.web.server.Site instance at 0x7face696c0e0>
2017-05-22T20:03:46-0700 [Launcher] Scrapyd 1.2.0 started: max_proc=4, runner=u'scrapyd.runner'
```

启动后会生成`dbs`文件夹和`twistd.pid`文件

```
(xh_venv) ql@ubuntu:~/Test/scrapyd_test$ ls
dbs  scrapy.cfg  scrapyd_test  twistd.pid
```



接着在另一个终端使用打包项目, 还是进入项目根目录:`scrapyd-deploy <deploy-name> -p <project-name>`

```shell
(xh_venv) ql@ubuntu:~/Test/scrapyd_test$ scrapyd-deploy baidu_d -p baidu_p
Packing version 1495511455
Deploying to project "baidu_p" in http://localhost:6800/addversion.json
Server response (200):
{"status": "ok", "project": "baidu_p", "version": "1495511455", "spiders": 1, "node_name": "ubuntu"}
```

此时就已经打包成功, 再次查看目录

```
(xh_venv) ql@ubuntu:~/Test/scrapyd_test$ ls
build  dbs  eggs  project.egg-info  scrapy.cfg  scrapyd_test  setup.py  twistd.pid
```



#### 四. 调度爬虫

命令`curl http://localhost:6800/schedule.json -d project=default -d spider=somespider`

- `default`是在`scrapy.cfg`中project的名字
- `somespider`是spider的名字

```shell
(xh_venv) ql@ubuntu:~/Test/scrapyd_test$ curl http://localhost:6800/schedule.json -d project=baidu_p -d spider=
baidu
{"status": "ok", "jobid": "07ac12263f7311e78202000c29ea42d2", "node_name": "ubuntu"}
```

执行后就可以在浏览器中打开`127.0.0.0:6800`查看执行情况, 日志等信息.



#### 五. 常用命令

- 调度爬虫

  ```
  curl http://localhost:6800/schedule.json -d project=default -d spider=somespider
  ```

- 取消爬虫

  ```
  curl http://localhost:6800/cancel.json -d project=myproject -d job=jobid
  ```

- 列出项目

  ```
  curl http://localhost:6800/listprojects.json
  ```

- 删除项目

  ```
  curl http://localhost:6800/delproject.json -d project=myproject
  ```

- 列出job

  ```
  curl http://localhost:6800/listjobs.json?project=myproject
  ```

- 列出爬虫

  ```
  curl http://localhost:6800/listspiders.json?project=myproject
  ```


