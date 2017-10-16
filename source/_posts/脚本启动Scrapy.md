---
title: 脚本启动Scrapy
date: 2017-06-16 04:12:02
categories: 爬虫
tags:
  - scrapy
---

[官方文档_脚本启动爬虫](https://doc.scrapy.org/en/latest/topics/practices.html)

在项目根目录下新建`run.py`, 如目录结构

```
.
├── mysite
│   ├── __init__.py
│   ├── __init__.pyc
│   ├── items.py
│   ├── middlewares.py
│   ├── pipelines.py
│   ├── settings.py
│   ├── settings.pyc
│   └── spiders
│       ├── baidu.py
│       ├── __init__.py
│       └── __init__.pyc
├── run.py
└── scrapy.cfg
```



### 一. execute方法

```python
from scrapy.cmdline import execute
execute(['scrapy', 'crawl', spider_name])
```

### 二. CrawlerProcess方法

`Scrapy`是构建于`Twisted`异步网络库上的, 所以必须在`Twisted reactor`里面启动`Scrapy`, `CrawlerProcess`类会为我们启动`Twisted reactor`

```python
from scrapy.crawler import CrawlerProcess
# 导入项目的配置文件
from scrapy.utils.project import get_project_settings
process = CrawlerProcess(get_project_settings())

# 导入爬虫
from <project_name>.spiders.<spider_name> import BaiduSpider

process.crawl(BaiduSpider)
# 爬虫启动后会阻塞在这里, 直到爬虫关闭
process.start()
```

<!--more-->

### 三. CrawlerRunner(推荐方法)

这个类可以帮助运行多个爬虫, 但是不能自动**启动**和**关闭**`Twisted reactor`

如果想在同一个`reactor`中运行爬虫, 那么建议使用`CrawlerRunner`

**注意:** 一定要在爬虫完毕时关闭`reactor`, 可以在`CrawlerRunner.crawl`中加入回调函数来进行关闭

```python
from twisted.internet import reactor
from scrapy.crawler import CrawlerRunner
from scrapy.utils.log import configure_logging
from scrapy.utils.project import get_project_settings

# 加入项目配置文件
runner = CrawlerRunner(get_project_settings())

# 导入爬虫
from <project_name>.spiders.<spider_name> import BaiduSpider

configure_logging({'LOG_FORMAT': '%(levelname)s: %(message)s'})

d = runner.crawl(BaiduSpider)

# 关闭reactor
d.addBoth(lambda _: reactor.stop())

reactor.run()
```



### 四. 在同一进程中运行多个爬虫

默认情况下, 当你运行`scrapy crawl ...`的时候每个进程运行一个脚本, 但也能在同一进程运行多个脚本

- 使用`CrawlerProcess`

  ```python
  import scrapy
  from scrapy.crawler import CrawlerProcess

  class MySpider1(scrapy.Spider):
      # Your first spider definition
      ...

  class MySpider2(scrapy.Spider):
      # Your second spider definition
      ...

  process = CrawlerProcess()
  process.crawl(MySpider1)
  process.crawl(MySpider2)
  process.start() # the script will block here until all crawling jobs are finished
  ```

- 使用`CrawlerRunner`

  ```python
  import scrapy
  from twisted.internet import reactor
  from scrapy.crawler import CrawlerRunner
  from scrapy.utils.log import configure_logging

  class MySpider1(scrapy.Spider):
      # Your first spider definition
      ...

  class MySpider2(scrapy.Spider):
      # Your second spider definition
      ...

  configure_logging()
  runner = CrawlerRunner()
  runner.crawl(MySpider1)
  runner.crawl(MySpider2)
  d = runner.join()
  d.addBoth(lambda _: reactor.stop())

  reactor.run() # the script will block here until all crawling jobs are finished
  ```

- 使用`reactor`按照顺序调用spider

  ```python
  from twisted.internet import reactor, defer
  from scrapy.crawler import CrawlerRunner
  from scrapy.utils.log import configure_logging

  class MySpider1(scrapy.Spider):
      # Your first spider definition
      ...

  class MySpider2(scrapy.Spider):
      # Your second spider definition
      ...

  configure_logging()
  runner = CrawlerRunner()

  @defer.inlineCallbacks
  def crawl():
      yield runner.crawl(MySpider1)
      yield runner.crawl(MySpider2)
      reactor.stop()

  crawl()
  reactor.run()
  ```

  ​
