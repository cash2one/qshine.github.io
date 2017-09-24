---
title: 'pyspider[一]CentOS7.3安装pyspider'
date: 2017-03-01 19:32:37
categories: 爬虫
tags:
  - pyspider
  - python
---

感谢作者能开发出来这么好用的框架, 以下是作者博客和官方文档:

- [作者博客](https://binux.blog/archives/)
- [pyspider中文社区](http://www.pyspider.cn/)


# 安装

__环境:__ centos7.3

1. 首先参考中文社区的资料安装, [资料链接](http://www.pyspider.cn/book/pyspider/centos-install-pyspider-7.html), 之后运行可能会碰到以下错误, 依次解决就行
2. `curl-config`错误
`__main__.ConfigurationError: Could not run curl-config: [Errno 2] No such file or directory`
解决
```shell
sudo yum install libcurl-devel -y
```
3. `gcc`错误
 `unable to execute gcc: No such file or directory error: command 'gcc' failed with exit status 1`
安装
```shell
yum install gcc
```
4. `pycurl`错误
`failed with error code 1 in /tmp/pip-build-_DZWAj/pycurl/`
安装 
```shell
yum install python-devel
```
5. 出现`mysql-connector`错误
```
pip install mysql-connector
```
如果提示错误的话执行以下语句
```
pip install mysql-connector-python-rf
```
```python
In [1]: import mysql.connector

In [2]:
```
不报错即可


# 运行起来后的错误

1. `pycurl`错误
`ImportError: pycurl: libcurl link-time ssl backend (nss) is different from compile-time ssl backend (none/other)`
解决: 使用pip安装pycurl运行提示失败, easy_install安装运行是成功的
```
pip uninstall pycurl
export PYCURL_SSL_LIBRARY=nss
easy_install pycurl
```

<!--more-->

