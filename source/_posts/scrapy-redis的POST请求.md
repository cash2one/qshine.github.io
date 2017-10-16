---
title: scrapy-redis的POST请求
date: 2017-07-31 20:35:50
categories: 爬虫
tags:
  - scrapy
---

一直使用的`scrapy-redis`做的分布式爬虫, 今天要爬的一个站要进行搜索, 入口函数是`POST`请求的, 但是`scrapy-redis`针对`start_urls`默认是`GET`的, 于是怎样传参数成了一个问题.



我这的情况是得到一个值num, 针对num去目标网站进行爬取结果返回, 搜索采用的是`POST`方式, 下面记录一下的我的解决方案:

- 把num值放到url参数中, 如`http://123456789`
- 通过下载器中间件的`process_request`方法去得到这个值
- 得到后修改请求为`POST`形式, 把参数封装到请求里, 返回新的请求

如上, scrapy在运行的时候会经过下载器中间件, 由于在`process_request`方法中返回了新的`request`, 所以旧的请求不会去执行

<!--more-->

```python
def process_request(self, request, spider):
    """搜索页更换为POST方法"""
    if spider.name == 'test':
        if re.match(r'http://\d+', request.url):
            url_parse = urlparse.urlparse(request.url)
            reg_no = url_parse.netloc
            return scrapy.http.FormRequest(
                post_url,
                formdata= {
                    'keyword': reg_no,
                }
            )
        return None
```




