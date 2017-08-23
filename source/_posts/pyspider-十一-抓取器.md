---
title: 'pyspider[十一]抓取器'
date: 2017-05-12 22:31:58
categories: 爬虫
tags:
  - pyspider
  - python
  - tornado
---

### 入口函数run
run函数中的`queue_loop`函数是由`PeriodicCallback`来启动的, 先来看tornado中的这个函数, 以下内容翻译自[tornado文档](http://tornado-zh.readthedocs.io/zh/latest/ioloop.html)
```python
class tornado.ioloop.PeriodicCallback(callback, callback_time, io_loop=None):
    pass
```
> callback函数每隔callback_time会被调用一次, callback_time的单位是`毫秒`, 注意tornado中其它的单位大部分是`秒`.  
如果callback运行时间大于`callback_time`, 随后的调用将被跳过以按时返回
`start`方法必须在`PeriodicCallback`创建后运行

这里有点懵, 不知道为啥要用这个东西??? 还望帮忙解答

在`queue_loop`中的逻辑和调度器差不多, 一个死循环不断从`inqueue`即`scheduler2fetcher`中非阻塞拿出任务(封装成一定格式的task), 交给`fetch`函数去处理
```python
    def run(self):
        '''
        抓取器的入口程序
        Run loop
        '''
        logger.info("fetcher starting...")

        def queue_loop():
            if not self.outqueue or not self.inqueue:
                return
            while not self._quit:
                try:
                    if self.outqueue.full():
                        break
                    if self.http_client.free_size() <= 0:
                        break
                    # get task from scheduler2fetcher
                    task = self.inqueue.get_nowait()

                    # FIXME: decode unicode_obj should used after data selete from
                    # database, it's used here for performance
                    task = utils.decode_unicode_obj(task)
                    # fetcher开始抓取任务
                    self.fetch(task)
                except queue.Empty:
                    break
                except KeyboardInterrupt:
                    break
                except Exception as e:
                    logger.exception(e)
                    break

        tornado.ioloop.PeriodicCallback(queue_loop, 100, io_loop=self.ioloop).start()
        tornado.ioloop.PeriodicCallback(self.clear_robot_txt_cache, 10000, io_loop=self.ioloop).start()
        self._running = True

        try:
            self.ioloop.start()
        except KeyboardInterrupt:
            pass

        logger.info("fetcher exiting...")
```

<!--more-->

### async_fetch函数
在fetch函数中对`self.async`做了判断, 紧接着走到了`async_fetch`函数
```python
@gen.coroutine
def async_fetch(self, task, callback=None):
    '''
    异步抓取
    Do one fetch
    '''
    url = task.get('url', 'data:,')
    if callback is None:
        callback = self.send_result

    type = 'None'
    start_time = time.time()
    try:
        # 根据不同的抓取种类来判断使用那种抓取器
        # 如果是data开头的网址
        if url.startswith('data:'):
            type = 'data'
            result = yield gen.maybe_future(self.data_fetch(url, task))
        # 如果 fetch_type 是 js 的话
        elif task.get('fetch', {}).get('fetch_type') in ('js', 'phantomjs'):
            type = 'phantomjs'
            result = yield self.phantomjs_fetch(url, task)
        # 如果抓取类型是 splash
        elif task.get('fetch', {}).get('fetch_type') in ('splash', ):
            type = 'splash'
            result = yield self.splash_fetch(url, task)
        # 如果是http开头的, 返回tornado抓取回来的结果
        else:
            type = 'http'
            result = yield self.http_fetch(url, task)
    except Exception as e:
        logger.exception(e)
        result = self.handle_error(type, url, task, start_time, e)

    callback(type, task, result)
    # on_result是统计程序
    self.on_result(type, task, result)

    # 对于gen.coroutine要用这种方式返回值
    raise gen.Return(result)
```
该函数使用了`@gen.coroutine`装饰器, 表示该方法是异步的
该函数主要是对调度器传过来的任务类型进行了判断, 是`data`, `js`还是`http`类型的, 根据不同的类型来让不同的抓取函数进行抓取, 我们主要看`http`.
可以看到, 对于http的url使用了`self.http_fetch`

### http_fetch函数
```python
@gen.coroutine
def http_fetch(self, url, task):
    """对于http类型的url抓取"""
    start_time = time.time()

    # 暂时不用管该函数
    self.on_fetch('http', task)

    # 定义一个错误处理函数, 如果发生错误会输出状态码等错误日志信息
    handle_error = lambda x: self.handle_error('http', url, task, start_time, x)

    # 设置请求的头部参数, 返回来的fetch是一个字典
    fetch = self.pack_tornado_request_parameters(url, task)
    task_fetch = task.get('fetch', {})

    session = cookies.RequestsCookieJar()
    # fix for tornado request obj
    if 'Cookie' in fetch['headers']:
        c = http_cookies.SimpleCookie()
        try:
            c.load(fetch['headers']['Cookie'])
        except AttributeError:
            c.load(utils.utf8(fetch['headers']['Cookie']))
        for key in c:
            session.set(key, c[key])
        del fetch['headers']['Cookie']
    if 'cookies' in fetch:
        session.update(fetch['cookies'])
        del fetch['cookies']

    max_redirects = task_fetch.get('max_redirects', 5)
    # we will handle redirects by hand to capture cookies
    fetch['follow_redirects'] = False

    # TODO: 为什么用 while True 来发起请求
    while True:
        # 看url是否触碰了robots协议
        if task_fetch.get('robots_txt', False):
            can_fetch = yield self.can_fetch(fetch['headers']['User-Agent'], fetch['url'])
            if not can_fetch:
                error = tornado.httpclient.HTTPError(403, 'Disallowed by robots.txt')
                raise gen.Return(handle_error(error))

        try:
            # 实例化HTTPRequest, 加入初始化参数
            request = tornado.httpclient.HTTPRequest(**fetch)
            # if cookie already in header, get_cookie_header wouldn't work
            old_cookie_header = request.headers.get('Cookie')
            if old_cookie_header:
                del request.headers['Cookie']
            cookie_header = cookies.get_cookie_header(session, request)
            if cookie_header:
                request.headers['Cookie'] = cookie_header
            elif old_cookie_header:
                request.headers['Cookie'] = old_cookie_header
        except Exception as e:
            logger.exception(fetch)
            raise gen.Return(handle_error(e))

        try:
            # 真正发起网络请求, 使用的是CurlAsyncHTTPClient, 支持代理
            response = yield gen.maybe_future(self.http_client.fetch(request))
        except tornado.httpclient.HTTPError as e:
            if e.response:
                response = e.response
            else:
                raise gen.Return(handle_error(e))

        extract_cookies_to_jar(session, response.request, response.headers)

        # 如果返回状态码3开头
        if (response.code in (301, 302, 303, 307)
                and response.headers.get('Location')
                and task_fetch.get('allow_redirects', True)):
            if max_redirects <= 0:
                error = tornado.httpclient.HTTPError(
                    599, 'Maximum (%d) redirects followed' % task_fetch.get('max_redirects', 5),
                    response)
                raise gen.Return(handle_error(error))
            if response.code in (302, 303):
                fetch['method'] = 'GET'
                if 'body' in fetch:
                    del fetch['body']
            fetch['url'] = quote_chinese(urljoin(fetch['url'], response.headers['Location']))
            fetch['request_timeout'] -= time.time() - start_time
            if fetch['request_timeout'] < 0:
                fetch['request_timeout'] = 0.1
            max_redirects -= 1
            continue

        # 封装返回结果
        result = {}
        result['orig_url'] = url
        result['content'] = response.body or ''
        result['headers'] = dict(response.headers)
        result['status_code'] = response.code
        result['url'] = response.effective_url or url
        result['time'] = time.time() - start_time
        result['cookies'] = session.get_dict()
        result['save'] = task_fetch.get('save')
        if response.error:
            result['error'] = utils.text(response.error)

        # fetcher在控制台输出的结果
        # 如: [I 170227 13:20:09 tornado_fetcher:409] [200] gtobal_demo:692591ddf03da73a2d2ca0aa596f56ad http://juiw129n_442733.cn.gtobal.com/ 8.55s
        if 200 <= response.code < 300:
            logger.info("[%d] %s:%s %s %.2fs", response.code,
                        task.get('project'), task.get('taskid'),
                        url, result['time'])
        else:
            logger.warning("[%d] %s:%s %s %.2fs", response.code,
                           task.get('project'), task.get('taskid'),
                           url, result['time'])

        raise gen.Return(result)
```
该函数好长, 细节看代码中的注释就行了, 注意以下几个函数

- `pack_tornado_request_parameters`: 该函数主要用来封装`fetch`, 首先是设置tornado请求的头中的各种参数, 之后设置代理proxy相关, 之后就是etag, data等信息, 封装好fetch后发返回

- `tornado.httpclient.HTTPRequest(**fetch)`
fetch是一个字典, 字典中的参数都是`HTTPRequest`对象的参数, 参考[HTTPRequest文档](http://tornado-zh.readthedocs.io/zh/latest/httpclient.html)

- `response = yield gen.maybe_future(self.http_client.fetch(request))`
在这里对实例化的request发起了网络请求, 但是真正使用的是`MyCurlAsyncHTTPClient`


### 关于MyCurlAsyncHTTPClient
在`Fetcher`类的`__init__`方法中又如下内容:
```python
# binding io_loop to http_client here
if self.async:
    self.http_client = MyCurlAsyncHTTPClient(max_clients=self.poolsize,
                                             io_loop=self.ioloop)
else:
    self.http_client = tornado.httpclient.HTTPClient(MyCurlAsyncHTTPClient, max_clients=self.poolsize)
```
其中`MyCurlAsyncHTTPClient`类继承自`CurlAsyncHTTPClient`, 所以主要看`CurlAsyncHTTPClient`


### 异步HTTP客户端
tornado定义了两种实现方式`simple_httpclient`和`curl_httpclient`, 默认的实现是`simple_httpclient`, 但是`curl_httpclient`有些情况会更优
- `curl_httpclient` 更快.
- `curl_httpclient` 更有可能与不完全符合 HTTP 规范的网站兼容, 或者与 使用很少使用 HTTP 特性的网站兼容.
- `curl_httpclient` 有一些 simple_httpclient 不具有的功能特性, 包括对 HTTP 代理和使用指定网络接口能力的支持.

[文档](http://tornado-zh.readthedocs.io/zh/latest/httpclient.html)

