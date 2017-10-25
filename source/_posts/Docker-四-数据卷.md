---
title: Docker(四)数据卷
date: 2017-10-23 13:02:37
categories: Docker
tags:
---

当删除容器时, 会把在容器中保存的数据也删除掉, 数据卷的目的是数据的持久化和共享数据, 它完全独立容器的生命周期.

### 容器和宿主机共享
启动一个名为`ubuntu`的容器, 镜像是`ubuntu:16.04`
- 宿主机目录是` /Users/qinjianxun/Desktop/docker_shared`
- 容器目录是`/workspace`, 容器目录可以自动创建
```
docker run -it --name ubuntu -v /Users/qinjianxun/Desktop/docker_shared:/workspace ubuntu:16.04 /bin/bash
```
- `--name`: 容器名称
- `-v`: 挂载目录, 宿主机目录:容器目录

**场景:** 
- 使用mysql容器的时候保存数据到宿主机
- 运行web项目可以保存日志文件


### 容器和容器之间的数据共享
启动一个`ubuntu_test`容器, 挂载上例ubuntu容器中的`/workspace`目录
```
docker run -it --name ubunu_test --volumes-from ubuntu ubuntu:16.04 /bin/bash
```
- `--volumes-from`: 挂载哪个容器

**场景:** 
- 一个python容器运行web, 另一个运行nginx, 使用nginx挂载web容器来实现静态文件的共享


