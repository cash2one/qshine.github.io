---
title: 'Docker(一)安装,概念和学习资料'
date: 2017-08-05 14:08:43
categories: Docker
tags:
---

### 介绍
`Docker`依赖于宿主机环境, 不需要像传统的VMWare一样安装一个完整的系统, 所以体积更小, 启动更快
`Docker`采用`C/S架构`, 由3部分组成: 客户端, 服务端, 远程仓库


### 环境: ubuntu16.04

1. 安装
   ```
   sudo apt-get install docker.io
   ```

2. 配置docker加速器, 此处使用[DaoCloud](https://www.daocloud.io/mirror#accelerator-doc)
   ```
   curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://bf898785.m.daocloud.io
   ```

### 概念解释

- `镜像(images)`: build component, 简单理解就是容器的只读版本(也能理解为系统盘), 如使用`docker pull命令来`
- `容器(container)`: 使用容器来运行应用
- `仓库(Registry)`: 用来集中的存储镜像

Docker使用的Linux核心技术AFUS: 用来建立不同的操作系统和隔离运行时的硬盘空间

> AUFS -> Another Union File System，AUFS的技术可以让多个文件目录union成一个新的目录，并且可以对这个新的目录进行读写操作. docker的Image，其实也就是一个事先制作好的只读的文件目录，当我们要使用这个系统功能的时候，docker为我们开辟了一个新的文件夹和这个image做了union，提供给docker container做为系统运行的存储，这个image里面已经包含了系统程序、工具软件、以及程序，当系统启动后产生的运行时文件（如logs、临时目录等）或新安装的软件都在这个新的文件夹内。这样我们在启动一个container的时候，其实并没有加载镜像的过程，也不会像vm一样需要安装一个系统这么负责，只是做了一次unoin，一切就和安装过系统的虚拟机同样使用了。docker还提供了一个`docker commit`命令，这个命令可以随时将你现在的运行中的cantainer构建成一个新的image。
>
> https://yq.aliyun.com/articles/63517?spm=5176.100239.blogcont63035.18.8Tdj7g


### 学习资料
[官方文档 ](http://link.zhihu.com/?target=https%3A//docs.docker.com/)
[知乎链接](https://zhuanlan.zhihu.com/p/22005181)
[Docker简明教程](http://open.daocloud.io/learning-docker/)
[Docker入门教程](http://dockone.io/article/111)
[Docker资源](http://www.docker.org.cn/page/resources.html)
[阿里云-大白话Docker入门（一）](https://yq.aliyun.com/articles/63035?spm=5176.100239.blogcont63517.18.eZV2Mv)
[阿里云-大白话Docker入门（一）](https://yq.aliyun.com/articles/63517?spm=5176.100239.blogcont63035.18.KUHo9D)
[腾讯云-Docker快速入门与进阶](https://www.qcloud.com/community/article/299094)
[Docker博客教程](http://hello1024world.com/page/3/)

<!--more-->
