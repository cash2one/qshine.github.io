---
title: asyncio入门
date: 2017-08-23 22:36:26
categories: Python
tags:
---

在知乎上不断看到`asyncio`, 终于下定决心进行学习, 第一次接触对这种语法还是蛮不适应的, 特别是python3的版本不同导致关系很乱, 看了很多篇文章才逐步搞明白. 我直接使用anaconda安装的python3.6, 测试中python3.4的特性在3.6中也是可以使用的, 所以建议直接使用高版本

### python3.4环境

asyncio是python3.4引入的标准库, 关于写asyncio的几个要点

- `@asyncio.coroutine`装饰函数为`coroutine`
- 获取`EventLoop`的引用
- 使用`run_until_complete`执行`coroutine`

```python
import asyncio

# 把hello函数改为异步函数
@asyncio.coroutine
def test(num):
	print(num)
    # 模拟等待1秒
	n = yield from asyncio.sleep(1)
	print('done !!!')

# 获取EventLoop
loop = asyncio.get_event_loop()
loop.run_until_complete(test(100))
loop.close()

"""
100
done !!!
"""
```
执行过程中遇到`yield from`会让出cpu的执行权

下面执行多个任务
```python
import asyncio

# 把hello变为异步函数
@asyncio.coroutine
def test(num):
	print(num)
    # asyncio.sleep(1)也是一个coroutine, 所以线程不会阻塞, 而是中断执行下一个消息循环, 实际应用中可以改为对应的IO任务, 如网络IO等
	n = yield from asyncio.sleep(1)
	print('done !!!')

tasks = [test(1), test(2)]

loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.gather(*tasks))
loop.close()

"""
2
1
done !!!
done !!!
"""
```

<!--more-->


### python3.5

`async`替换了`@asyncio.coroutine`, `await`替换了`yield from`, 同时引入了`async with(异步上下文)`和`async for(异步迭代器) `

下例来测试获取ip地址时使用同步和异步的区别

- requests同步获取
  ```python
  import time
  import requests

  def test(num):
  	return (num, requests.get('http://www.icanhazip.com').content)

  start = time.time()

  results = [test(i) for i in range(5)]
  for i in results:
  	print(i)

  print(time.time()-start)

  """
  (0, b'106.38.132.221\n')
  (1, b'106.38.132.221\n')
  (2, b'106.38.132.221\n')
  (3, b'106.38.132.221\n')
  (4, b'106.38.132.221\n')
  15.404923915863037
  """
  ```
  
- aiohttp异步获取
  ```python
  import time
  import aiohttp
  import asyncio

  async def test(num):
  	async with aiohttp.request('GET', 'http://www.icanhazip.com') as r:
  		data = await r.text()
  	return (num, data)


  start = time.time()
  tasks = [test(i) for i in range(5)]
  loop = asyncio.get_event_loop()
  results = loop.run_until_complete(asyncio.gather(*tasks))
  loop.close()

  for i in results:
  	print(i)

  print(time.time()-start)


  """
  (0, '106.38.132.221\n')
  (1, '106.38.132.221\n')
  (2, '106.38.132.221\n')
  (3, '106.38.132.221\n')
  (4, '106.38.132.221\n')
  4.268573045730591
  """
  ```
  `asyncio.gather`可以保证按照tasks的顺序返回
  巨大的时间差可以看出同步和异步的差距


### 资料

- [Python并发编程之协程/异步IO](https://www.ziwenxie.site/2016/12/19/python-asyncio/)
- [使用Python进行并发编程-asyncio篇](http://www.dongwm.com/archives/%E4%BD%BF%E7%94%A8Python%E8%BF%9B%E8%A1%8C%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B-asyncio%E7%AF%87/)




