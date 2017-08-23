---
title: 解决Django与MySQL交互丢失连接问题
date: 2017-06-22 20:34:39
categories: Web后端
tags:
  - Django
---

在使用`Django`和`Scrapy`的过程中碰到的一个bug,  我在Scrapy中使用了Django的ORM来做存储, 前端通过Django的API来调用爬虫, 爬虫把结果入库`MySQL`之后返回给前端, 测试是一直正常的, 但是每隔一段时间就会出现不能入库的情况, 抛出__Exception Value: (0, '')__错误, 我把异常处理去掉后会报__mysql gone away__错误

确认问题发生在`Pipeline`中的入库阶段, 因为日志里面已经显示出爬虫结果了, 之后经过一段时间的测试, 应为相同的代码我跑了两个, 代码一模一样, 扩充数据源的那份一直没有出过问题, 但是接口这份一天不调用第二天就出问题, 所以感觉跟`Django`与`MySQL`的交互有问题

#### 原因

用户通过Django来使用MySQL数据库的时候, Django会与MySQL建立连接, 然后把数据返回, 但是这个连接只能默认保持8小时, 过后MySQL会把这个连接丢掉, 但是Django并没有, 认为该连接还能使用, 所以下次和数据库交互还会使用该连接, 然后报错

http://www.jianshu.com/p/69dcae4454b3

#### 解决问题

##### 一. 每次都关闭连接

```python
from django.db import connection

connection.close()
```

该方法我没有进行测试, 因为如果某一时间段调用量非常大, 那么每次都进行关闭很耗时间

##### 二. 调整MySQL连接的过期时间

因为数据库其它程序也还在用, 所以粗暴修改可能解决该问题, 但是可能会给其它程序带来问题

##### 三. 在异常处关闭不可用连接

我最后采用的这种方案, 也确实解决了问题, 以下代码是Scrapy中的Pipeline内容

```python

from django.db import connection
from django.db.utils import OperationalError


class CrawlerPipeline(object):
    def process_item(self, item, spider):
        try:
            # 使用Django的ORM进行入库
            item.save()
        except OperationalError:
            # 如果发生异常, 关闭该连接, 这样下次djagno就不会选择该失效连接
            connection.close()
			# 重新保存, Django会自动建立新的连接
            item.save()
        return item
```





#### 资料

- http://www.cnblogs.com/bugmaker/articles/2444905.html
- https://zhaojames0707.github.io/post/django_mysql_gone_away/
- http://blog.csdn.net/orangleliu/article/details/41480417
- https://stackoverflow.com/questions/30898068/django-pymysql-throws-mysql-server-has-gone-away-after-a-couple-of-hours





<!--more-->


















