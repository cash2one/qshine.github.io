---
title: 'pyspider[十二]使用常见问题总结'
date: 2017-05-16 23:35:11
categories: 爬虫
tags: 
  - pyspider
---

目前我们使用的整个pyspider系统比较稳定, 目前上面的脚本数量有80多个, 当然也碰到过坑, 打算把解决的方法和当时参考过的文章全部在这记录下来供以后参考.

### `HTTP 599: Could not resolve host: www.xxx.com; Unknown error`
出现该错误的任务状态一直是`ACTIVE`, 我自己使用的`ubuntu`开发, 本地是任何问题没有的, 但是把pyspider在生产上跑起来后出现的该问题, 单步调试也是没有问题的, 一旦开起来, 会有大量错误, 总之该问题困扰了我们很长时间
#### 1. IPV6和DNS的错误
- http://stackoverflow.com/questions/24967855/curl-6-could-not-resolve-host-google-com-name-or-service-not-known)
- http://www.ttlsa.com/linux/curl-6-couldnt-resolve-host/

#### 2. tornado依赖的问题
搜上面的两个文章包括其它文章, 经常会出现`pycurl`和`libcurl`这两个东西, 应该和它们有关.
- [pycurl百度百科](http://baike.baidu.com/link?url=kopSFlyMVlx8uwDBsWamwU20I1kDLsvt1EZ4Kg-1nsbyaQzNJnFSeOCBS_1EBVHRDqXoQ-y3ye6Lcp7Cw6tpba)
- [libcurl百度百科](http://baike.baidu.com/item/libcurl)

这个两个东西主要是和`网络`有关的, 那么说明出错的原因只能和负责网络请求的组件有关, 即抓取器Fetcher, pyspider的Fetcher依赖的是[tornado的异步HTTP客户端](http://tornado-zh.readthedocs.io/zh/latest/httpclient.html), 也就是说爬虫的网络请求都是异步的,
在tornado的章节中出现如下一段
> 注意, 如果你正在使用 curl_httpclient, 强力建议你使用最新版本的 libcurl 和 pycurl. 当前 libcurl 能被支持的最小版本是 7.21.1, pycurl 能被支持的最小版本是 7.18.2. 强烈建议你所安装的 libcurl 是和异步 DNS 解析器 (threaded 或 c-ares) 一起构建的, 否则你可能会遇到各种请求超时的问题

找到这里后我马上去查看了我的ubuntu和centos这两个东西的版本, 发现确实ubuntu上的版本更高一点
```python
In [4]: import pycurl

In [5]: pycurl.version
Out[5]: 'PycURL/7.43.0 libcurl/7.47.0 OpenSSL/1.0.2g zlib/1.2.8 libidn/1.32 librtmp/2.3'

In [6]:
```
以上就把`pycurl`和`libcurl`的版本全部列出来了, 该版本跑pyspider一切都是正常的, 可以进行对比

关于升级:
环境centos7, 使用yum升级并不能保证升级到最新, 我使用以下命令
```shell
# 安装最新libcurl源
rpm -ivh http://mirror.city-fan.org/ftp/contrib/yum-repo/city-fan.org-release-1-13.rhel6.noarch.rpm
# 升级libcurl
yum upgrade libcurl
# 升级完后卸载此源
rpm -e city-fan.org-release
```
__注意:__ 如果升级时提示Requires: libnghttp2.so.14()(64bit)错误时，可以使用安装epel源来解决，安装方法如下：
```shell
yum install epel-release -y
```

<!--more-->
