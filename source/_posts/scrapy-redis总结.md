---
title: scrapy-redis总结
date: 2017-11-08 14:41:34
categories: 爬虫
tags:
  - scrapy
  - scrapy-redis
---

## queue.py
scrapy-redis中有三个队列, 三个队列都继承自`Base`类
#### 1. 先进先出: FIFO
```python
class FifoQueue(Base):
    """Per-spider FIFO queue"""

    def __len__(self):
        """Return the length of the queue"""
        return self.server.llen(self.key)

    def push(self, request):
        """Push a request"""
        self.server.lpush(self.key, self._encode_request(request))

    def pop(self, timeout=0):
        """Pop a request"""
        if timeout > 0:
            data = self.server.brpop(self.key, timeout)
            if isinstance(data, tuple):
                data = data[1]
        else:
            data = self.server.rpop(self.key)
        if data:
            return self._decode_request(data)

```
使用了redis的list结构
#### 2. 优先级队列: PriorityQueue
```python
class PriorityQueue(Base):
    """Per-spider priority queue abstraction using redis' sorted set"""

    def __len__(self):
        """Return the length of the queue"""
        return self.server.zcard(self.key)

    def push(self, request):
        """Push a request"""
        data = self._encode_request(request)
        score = -request.priority
        # We don't use zadd method as the order of arguments change depending on
        # whether the class is Redis or StrictRedis, and the option of using
        # kwargs only accepts strings, not bytes.
        self.server.execute_command('ZADD', self.key, score, data)

    def pop(self, timeout=0):
        """
        Pop a request
        timeout not support in this queue class
        """
        # use atomic range/remove using multi/exec
        pipe = self.server.pipeline()
        pipe.multi()
        pipe.zrange(self.key, 0, 0).zremrangebyrank(self.key, 0, 0)
        results, count = pipe.execute()
        if results:
            return self._decode_request(results[0])
```
使用redis的`sorted set`实现, 如果在spider脚本中需要指定`priority`的话, 一定要在`settings`中来声明使用的是`PriorityQueue`,
取相反数后高优先级的就会排在最前面
#### 3. 后进先出(栈)
```python

class LifoQueue(Base):
    """Per-spider LIFO queue."""

    def __len__(self):
        """Return the length of the stack"""
        return self.server.llen(self.key)

    def push(self, request):
        """Push a request"""
        self.server.lpush(self.key, self._encode_request(request))

    def pop(self, timeout=0):
        """Pop a request"""
        if timeout > 0:
            data = self.server.blpop(self.key, timeout)
            if isinstance(data, tuple):
                data = data[1]
        else:
            data = self.server.lpop(self.key)

        if data:
            return self._decode_request(data)
```
和先进先出队列基本一样, 实现了栈结构

## 2. dupefilter.py
scrapy原生使用了一个python的`set`结构来进行请求去重, 在scrapy-redis中使用redis的`集合(set)`进行了替换, 指纹的计算方法还是用的原生的, 及如果之前请求过返回0, 否则返回1

```python
def request_seen(self, request):
    """Returns True if request was already seen.

    Parameters
    ----------
    request : scrapy.http.Request

    Returns
    -------
    bool

    """
    fp = self.request_fingerprint(request)    # 得到请求的指纹
    # This returns the number of values added, zero if already exists.
    added = self.server.sadd(self.key, fp)    # 把指纹添加到redis的集合中
    return added == 0

def request_fingerprint(self, request):
    """Returns a fingerprint for a given request.

    Parameters
    ----------
    request : scrapy.http.Request

    Returns
    -------
    str

    """
    return request_fingerprint(request)    # 得到请求指纹
```
请求指纹计算使用的是**sha1**算法, 计算值包括请求方法, url, body等信息

## scheduler.py
调度器肯定要和`请求队列`和`去重队列`进行交互, 所以初始化要获取使用的`queue`和`dupfilter`的类, 并在`open`方法中完成实例化

#### open方法
```python

    def open(self, spider):
        self.spider = spider

        try:
            # 得到队列queue的实例化对象
            self.queue = load_object(self.queue_cls)(
                server=self.server,
                spider=spider,
                key=self.queue_key % {'spider': spider.name},
                serializer=self.serializer,
            )
        except TypeError as e:
            raise ValueError("Failed to instantiate queue class '%s': %s",
                             self.queue_cls, e)

        try:
            # 得到去重的实例化对象
            self.df = load_object(self.dupefilter_cls)(
                server=self.server,
                key=self.dupefilter_key % {'spider': spider.name},
                debug=spider.settings.getbool('DUPEFILTER_DEBUG'),
            )
        except TypeError as e:
            raise ValueError("Failed to instantiate dupefilter class '%s': %s",
                             self.dupefilter_cls, e)

        if self.flush_on_start:    # 如果为True, 要在爬虫开启前删除对应爬虫request队列和dupfilter队列
            self.flush()
        # notice if there are requests already in the queue to resume the crawl
        if len(self.queue):
            spider.log("Resuming crawl (%d requests scheduled)" % len(self.queue))
```

#### close方法
如果在`settings`中设置`SCHEDULER_PERSIST = True`, 则不会执行删除(flush)操作, 否则在爬虫关闭时会删除爬虫的`request`和`dupfilter`队列

#### 其它方法
- enqueue_request: 入队操作, 先判断是否要过滤及是否抓取过
- next_request: 弹出任务去执行

#### 注意
在settings中有个配置`SCHEDULER_PERSIST`, 默认是`False`的, 就是每次关闭爬虫的时候都会删除`去重集合`和`请求队列`, 所以如果要求爬虫能够暂停和恢复的话要设置为True

### spider.py
scrapy有`Spider`和`CrawlerSpider`, 这里只看scrapy-redis中的`RedisSpider`
```python

from scrapy import signals
from scrapy.exceptions import DontCloseSpider
from scrapy.spiders import Spider, CrawlSpider

from . import connection, defaults
from .utils import bytes_to_str


class RedisMixin(object):
    """Mixin class to implement reading urls from a redis queue."""
    redis_key = None
    redis_batch_size = None
    redis_encoding = None

    # Redis client placeholder.
    server = None

    def start_requests(self):
        """Returns a batch of start requests from redis."""
        return self.next_requests()

    def setup_redis(self, crawler=None):
        """Setup redis connection and idle signal.

        This should be called after the spider has set its crawler object.
        """
        if self.server is not None:
            return

        if crawler is None:
            # We allow optional crawler argument to keep backwards
            # compatibility.
            # XXX: Raise a deprecation warning.
            crawler = getattr(self, 'crawler', None)

        if crawler is None:
            raise ValueError("crawler is required")

        settings = crawler.settings

        if self.redis_key is None:
            self.redis_key = settings.get(
                'REDIS_START_URLS_KEY', defaults.START_URLS_KEY,
            )

        # 得到redis的start_urls key
        self.redis_key = self.redis_key % {'name': self.name}

        if not self.redis_key.strip():
            raise ValueError("redis_key must not be empty")

        # 获取并发数量
        if self.redis_batch_size is None:
            # TODO: Deprecate this setting (REDIS_START_URLS_BATCH_SIZE).
            self.redis_batch_size = settings.getint(
                'REDIS_START_URLS_BATCH_SIZE',
                settings.getint('CONCURRENT_REQUESTS'),
            )

        try:
            self.redis_batch_size = int(self.redis_batch_size)
        except (TypeError, ValueError):
            raise ValueError("redis_batch_size must be an integer")

        if self.redis_encoding is None:
            self.redis_encoding = settings.get('REDIS_ENCODING', defaults.REDIS_ENCODING)

        self.logger.info("Reading start URLs from redis key '%(redis_key)s' "
                         "(batch size: %(redis_batch_size)s, encoding: %(redis_encoding)s",
                         self.__dict__)

        # 返回redis连接的实例
        self.server = connection.from_settings(crawler.settings)
        # The idle signal is called when the spider has no requests left,
        # that's when we will schedule new requests from redis queue
        crawler.signals.connect(self.spider_idle, signal=signals.spider_idle)

    def next_requests(self):
        """Returns a request to be scheduled or none."""
        # 从 start_urls 中获取任务进行爬取
        use_set = self.settings.getbool('REDIS_START_URLS_AS_SET', defaults.START_URLS_AS_SET)
        fetch_one = self.server.spop if use_set else self.server.lpop
        # XXX: Do we need to use a timeout here?
        found = 0
        # TODO: Use redis pipeline execution.
        while found < self.redis_batch_size:
            data = fetch_one(self.redis_key)
            if not data:
                # Queue empty.
                break
            req = self.make_request_from_data(data)    # 构造请求
            if req:
                yield req
                found += 1
            else:
                self.logger.debug("Request not made from data: %r", data)

        if found:
            self.logger.debug("Read %s requests from '%s'", found, self.redis_key)

    def make_request_from_data(self, data):
        """Returns a Request instance from data coming from Redis.

        By default, ``data`` is an encoded URL. You can override this method to
        provide your own message decoding.

        Parameters
        ----------
        data : bytes
            Message from redis.

        """
        url = bytes_to_str(data, self.redis_encoding)
        return self.make_requests_from_url(url)

    def schedule_next_requests(self):
        """Schedules a request if available"""
        # TODO: While there is capacity, schedule a batch of redis requests.
        for req in self.next_requests():
            self.crawler.engine.crawl(req, spider=self)

    def spider_idle(self):
        """Schedules a request if available, otherwise waits."""
        # XXX: Handle a sentinel to close the spider.
        self.schedule_next_requests()
        raise DontCloseSpider


class RedisSpider(RedisMixin, Spider):
    """Spider that reads urls from redis queue when idle.

    Attributes
    ----------
    redis_key : str (default: REDIS_START_URLS_KEY)
        Redis key where to fetch start URLs from..
    redis_batch_size : int (default: CONCURRENT_REQUESTS)
        Number of messages to fetch from redis on each attempt.
    redis_encoding : str (default: REDIS_ENCODING)
        Encoding to use when decoding messages from redis queue.

    Settings
    --------
    REDIS_START_URLS_KEY : str (default: "<spider.name>:start_urls")
        Default Redis key where to fetch start URLs from..
    REDIS_START_URLS_BATCH_SIZE : int (deprecated by CONCURRENT_REQUESTS)
        Default number of messages to fetch from redis on each attempt.
    REDIS_START_URLS_AS_SET : bool (default: False)
        Use SET operations to retrieve messages from the redis queue. If False,
        the messages are retrieve using the LPOP command.
    REDIS_ENCODING : str (default: "utf-8")
        Default encoding to use when decoding messages from redis queue.

    """

    @classmethod
    def from_crawler(self, crawler, *args, **kwargs):
        obj = super(RedisSpider, self).from_crawler(crawler, *args, **kwargs)    # 调用Spider类中的方法, 配置close signal
        obj.setup_redis(crawler)    # RedisSpider中初始化redis的连接
        return obj
```

## 总结
工作中经常用到`scrapy-redis`来进行抓取, 也会有很多情况处理起来比较复杂的情况, 比如之前碰到过一个网站一开始就是post请求的方式, 当时的解决方案是把post的参数放到url中, 然后把整个url扔到start_urls队列, 在下载器中间件拦截住连接, 然后再改为post形式, 最后正常工作了. 

