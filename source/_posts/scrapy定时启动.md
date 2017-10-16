---
title: scrapy定时启动
date: 2017-09-09 19:01:54
categories: 爬虫
tags:
  - scrapy
---

使用linux下的**cron**设置可以使scrapy定时启动, 方便做定时抓取,

#### scrapy目录结构

```
root@712d2efbe3e0:/workspace/baidu# pwd
/workspace/baidu
root@712d2efbe3e0:/workspace/baidu# tree
.
|-- baidu
|   |-- __init__.py
|   |-- items.py
|   |-- middlewares.py
|   |-- pipelines.py
|   |-- settings.py
|   `-- spiders
|       |-- __init__.py
|       `-- baidu_spider.py
|-- scrapy.cfg
|-- start.sh
root@712d2efbe3e0:/workspace/baidu#
```



#### 启动脚本start.sh

```shell
#!/bin/bash

export PATH=$PATH:/usr/local/bin    

cd /workspace/baidu/
nohup scrapy crawl baidu_spider >> spider.log 2>&1 &
```



#### 定时任务设置

输入`crontab -e`, 加入以下内容

```shell
*/1 *  *  *  * sh /workspace/baidu/start.sh    # 每一分钟启动一次
```

或者设置以下

```shell
30  7  *  *  * sh /workspace/baidu/start.sh    # 每天7:30启动
```

具体设置参考[Ubuntu 14.04 使用 Cron 实现计划任务](http://outprog.github.io/blog/2015/10/15/ubuntu-14-dot-04-shi-yong-cron-shi-xian-ji-hua-ren-wu/)




