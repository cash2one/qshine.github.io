---
title: asyncio入门
date: 2017-08-23 22:36:26
categories: Python
tags:
---

不断看到`asyncio`, 终于下定决心进行学习, 第一次接触对这种语法还是蛮不适应的, 特别是python3的版本不同导致关系很乱, 看了很多篇文章才逐步搞明白. 我直接使用anaconda安装的python3.6, 测试中python3.4的特性在3.6中也是可以使用的, 所以建议直接使用高版本

> - event_loop 事件循环: 程序开启一个无限的循环，把一些函数注册到事件循环上。当满足事件发生的时候，调用相应的协程函数
- coroutine 协程: 协程对象，指一个使用async关键字定义的函数，它的调用不会立即执行函数，而是会返回一个协程对象。协程对象需要注册到事件循环，由事件循环调用
- task 任务: 一个协程对象就是一个原生可以挂起的函数，任务则是对协程进一步封装，其中包含任务的各种状态
- future: 代表将来执行或没有执行的任务的结果
- async/await 关键字: python3.5 用于定义协程的关键字，async定义一个协程，await用于挂起阻塞的异步调用接口 



### python3.4环境

asyncio是python3.4引入的标准库, 关于写asyncio的几个步骤

- `@asyncio.coroutine`装饰函数为`coroutine`
- 获取`event_loop`的引用
- `run_until_complete`的参数是一个`future对象`, 如果传入一个协程的话会自动包装成一个`task对象`, `task对象`是`Future类`的子类。保存了协程运行后的状态，用于未来获取协程的结果

#### 示例1: 定义协程并执行
```python
import asyncio

# 把test函数改为异步函数
@asyncio.coroutine
def test(num):
  print(num)
    # 模拟等待1秒
  n = yield from asyncio.sleep(1)
  print('done !!!')

# 获取EventLoop
loop = asyncio.get_event_loop()

coroutine = test(100)
# 协程不能直接运行, 要把它加入事件循环
loop.run_until_complete(coroutine)
loop.close()

"""
100
done !!!
"""
```
执行过程中遇到`yield from`会让出cpu的执行权

#### 示例2: 显示创建task并输出task的状态
创建对象可以使用
- `loop.create_task(coroutine)`
- `asyncio.ensure_future(coroutine)`
```python
import asyncio

@asyncio.coroutine
def test(num):
  print(num)
    # 模拟等待1秒
  n = yield from asyncio.sleep(1)
  print('done !!!')

# 获取EventLoop
loop = asyncio.get_event_loop()

coroutine = test(100)

# 创建task
task = loop.create_task(coroutine)

# 输出task的状态
print(task)
# 协程不能直接运行, 要把它加入事件循环
loop.run_until_complete(task)

print(task)
loop.close()

"""
<Task pending coro=<test() running at test.py:4>>
100
done !!!
<Task finished coro=<test() done, defined at test.py:4> result=None>
"""
```

#### 示例3: 执行多个任务
```python
import asyncio

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

`async`替换以上的`@asyncio.coroutine`, `await`替换`yield from`, 同时引入了`async with(异步上下文)`和`async for(异步迭代器) `

使用async可以定义协程对象，使用await可以针对耗时的操作进行挂起，就像生成器里的yield一样，函数让出控制权。协程遇到await，事件循环将会挂起该协程，执行别的协程，直到其他的协程也挂起或者执行完毕，再进行下一个协程的执行

#### 示例1: 使用async/await
##### 使用wait关键字
```python
import asyncio

@asyncio.coroutine
def test(num):
    print(num)
    # asyncio.sleep(1)也是一个coroutine, 所以线程不会阻塞, 而是中断执行下一个消息循环, 实际应用中可以改为对应的IO任务, 如网络IO等
    n = yield from asyncio.sleep(1)
    return '%d done !!!'%num


tasks = [
    asyncio.ensure_future(test(1)), 
    asyncio.ensure_future(test(2)),
    asyncio.ensure_future(test(3)),
    asyncio.ensure_future(test(4)),
]

loop = asyncio.get_event_loop()

loop.run_until_complete(asyncio.wait(tasks))
loop.close()

for task in tasks:
    print(task.result())


"""
1
2
3
4
1 done !!!
2 done !!!
3 done !!!
4 done !!!
"""

```

##### 使用gather关键字
```python
import asyncio

@asyncio.coroutine
def test(num):
    print(num)
    # asyncio.sleep(1)也是一个coroutine, 所以线程不会阻塞, 而是中断执行下一个消息循环, 实际应用中可以改为对应的IO任务, 如网络IO等
    n = yield from asyncio.sleep(1)
    return '%d done !!!'%num


tasks = [
    test(1), 
    test(2),
    test(3),
    test(4),
]

loop = asyncio.get_event_loop()

results = loop.run_until_complete(asyncio.gather(*tasks))
loop.close()

for res in results:
    print(res)


"""
2
1
4
3
1 done !!!
2 done !!!
3 done !!!
4 done !!!
"""

```

**区别**:
- wait接受一个tasks列表
- gather接受一堆tasks

#### 示例2: 使用同步和异步的耗时
##### requests同步获取
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
  
##### aiohttp异步获取
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

- [python黑魔法asyncio - 推荐阅读](http://python.jobbole.com/87310/)
- [Python并发编程之协程/异步IO](https://www.ziwenxie.site/2016/12/19/python-asyncio/)
- [使用Python进行并发编程-asyncio篇](http://www.dongwm.com/archives/%E4%BD%BF%E7%94%A8Python%E8%BF%9B%E8%A1%8C%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B-asyncio%E7%AF%87/)




