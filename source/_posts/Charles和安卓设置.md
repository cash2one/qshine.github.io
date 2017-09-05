---
title: Charles和安卓设置
date: 2017-09-05 15:40:19
categories: 工具
tags:
  - 爬虫
---

工作需要抓的一个站只有app, 所以只能采用抓包的形式来抓取信息, 我使用的是mac连接有线网络, 共享热点给安卓手机. 记录下使用charles的配置流程:
1. 设置charles的HTTP代理: `Proxy>Proxy settings>Enable transparent HTTP proxying`, 端口使用默认`8888`
2. 点击charles的`Help>Local IP Address`可以看到两个地址, 下面还有一个`en4`, `en4`的使用情景是电脑和手机同处于一个wifi
![enter image description here](http://oh7hdmoe1.bkt.clouddn.com/17-9-5/30814215.jpg)
3. 设置手机
![enter image description here](http://oh7hdmoe1.bkt.clouddn.com/17-9-5/39508948.jpg)

以上设置可以通过查看charles捕获的http网站的内容.

因为需要的网站是https的, 所以还需设置以下
1. charles安装证书
`Help > SSL Proxying > Install Charles Root Certificate`
这是安装电脑端的证书, 注意需要设置为信任该证书
![enter image description here](http://oh7hdmoe1.bkt.clouddn.com/17-9-5/10267594.jpg)
2. `Help > SSL Proxying > Install Charles Root Certificate On a Mobild ......`
根据提示安装手机端的证书, 华为手机是在`设置 > 安全 > 从SD卡安装`
3. 打开charles的`Proxy > SSL Proxy Settgings`, 添加要抓取网站的`域名和端口(设置为443)`
![enter image description here](http://oh7hdmoe1.bkt.clouddn.com/17-9-5/47956428.jpg)

