---
title: supervisor使用笔记
date: 2017-07-07 17:22:18
categories: 工具
tags:
  - supervisor
---

> [Supervisor](http://supervisord.org/)是一个 Python 开发的 client/server 系统，可以管理和监控类 UNIX 操作系统上面的进程。它可以同时启动，关闭多个进程

```
pip install supervisor
```

以下为Django项目使用的配置过程

```
(xh_venv) ql@ubuntu:~/mysite$ tree
.
├── manage.py
└── mysite
    ├── __init__.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py

1 directory, 5 files
```

<!--more-->

1. 首先进入项目根目录`mysite`, 使用如下命令生成配置文件

  ```
  echo_supervisord_conf > supervisord.conf
  ```

2. 修改配置文件

  此处只进行简单配置, 找到`;[program:theprogramname]`内容,  根据需求自定义即可, 但是注意以下两个配置

  ```
  [program:mysite_web]
  command=python manage.py runserver 0.0.0.0:8000              ; the program (relative uses PATH, can take args)
  process_name=%(program_name)s ; process_name expr (default %(program_name)s)
  ;numprocs=1                    ; number of processes copies to start (def 1)
  ```

  - command: 项目启动命令

  - process_name: 启动进程名称(默认值是`%(program_name)s`)

  - process: 启动进程数(默认为1), 当需要修改这个值的时候需要改变process_name的值, 加上`%(process_num)s`来表示不同的进程, 如

    ```
    process_name=%(program_name)s_%(process_num)s 
    numprocs=8
    ```

3. 启动`supervisor`服务端

  ```
  supervisord
  ```

  此时启动的话默认会自动启动我们的项目, 因为参数`autostart`默认为`true`. 该命令会优先在本目录寻找`supervisord.conf`这个配置文件, 也可以指定配置文件

  ```
  supervisord -c supervisord.conf
  ```

4. 启动客户端查看情况

  - status: 状态
  - start: 启动
  - stop: 停止
  - restart: 重启

  ```
  (xh_venv) ql@ubuntu:~/mysite$ supervisorctl 
  mysite:mysite_0                  RUNNING   pid 25793, uptime 0:02:07
  mysite:mysite_1                  STARTING  
  mysite:mysite_2                  RUNNING   pid 26239, uptime 0:00:03
  supervisor> 
  supervisor> 
  supervisor> status
  mysite:mysite_0                  RUNNING   pid 25793, uptime 0:02:09
  mysite:mysite_1                  RUNNING   pid 26251, uptime 0:00:02
  mysite:mysite_2                  STARTING  
  supervisor> 
  supervisor> 
  supervisor> stop mysite:*
  mysite:mysite_2: stopped
  mysite:mysite_0: stopped
  mysite:mysite_1: stopped
  supervisor> 
  supervisor> 
  supervisor> status
  mysite:mysite_0                  STOPPED   Jul 07 05:17 PM
  mysite:mysite_1                  STOPPED   Jul 07 05:17 PM
  mysite:mysite_2                  STOPPED   Jul 07 05:17 PM
  supervisor> 
  supervisor> start mysite
  mysite: ERROR (no such process)
  supervisor> 
  supervisor> start mysite:*
  mysite:mysite_2: started
  mysite:mysite_0: started
  mysite:mysite_1: started
  supervisor> 
  supervisor> status
  mysite:mysite_0                  RUNNING   pid 26290, uptime 0:00:04
  mysite:mysite_1                  RUNNING   pid 26291, uptime 0:00:04
  mysite:mysite_2                  RUNNING   pid 26289, uptime 0:00:04
  supervisor> 

  ```

   ​


### 资料

- [官方文档](http://supervisord.org/index.html)
- http://liuzxc.github.io/blog/supervisor/
- http://liyangliang.me/posts/2015/06/using-supervisor/
- http://www.restran.net/2015/10/04/supervisord-tutorial/



