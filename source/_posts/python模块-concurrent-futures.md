---
title: python模块-concurrent.futures
date: 2017-08-15 06:16:04
categories: Python
tags:
  - python模块
---

python提供了`threading`和`multiprocessing`来编写多线程和多进程, 但是项目很大的时候需要频繁的创建和销毁线程或进程, 这样的操作非常耗资源, 所以可以创建`线程池/进程池`.

**concurrent.futures**是python3.2中的一个模块, 它为异步执行可调用的对象提供了高级接口, 其中又提供了

- ThreadPoolExecutor: 通过`线程池`实现异步调用
- ProcessPoolExecutor: 通过`进程池`实现异步调用

两者都是通过**Executor**这个抽象类来定义的, 使用的话也是通过使用这两个类来使用, 把相应的task直接放入线程池/进程池, 不需要维护`Queue`来担心死锁的问题, 会自动进行处理

### Future理解
`Future`类封装了一个可调用对象的异步执行过程, `Future`对象是通过`Executor.submit()`来创建的. 一个`Future`对象代表了一些尚未完成的结果, 在将来的某个时刻完成后可以得到这个结果

### 样例

1. 使用ThreadPoolExecutor和submit
调用submit后会生成`future`对象, 当这个对象调用`as_completed`后会去执行对应的函数
```python
import time
import json
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed


def get_num(num):
	r = requests.get('http://httpbin.org/get?a={}'.format(num))
	return json.loads(r.content)['args']['a']

nums = range(10)
start = time.time()

# 使用with语句确保线程能够被及时清理
with ThreadPoolExecutor(max_workers=10) as executor:
	# 生成future对象
	futures = [executor.submit(get_num, num) for num in nums]
	# 调用as_completed开始执行
	for future in as_completed(futures):
		print(future.result())


print(time.time() - start)


"""
8
5
0
6
7
9
3
2
1
4
0.5397617816925049
"""
```

2. 使用ProcessPoolExecutor和map
```python
import time
import json
import requests
from concurrent.futures import ProcessPoolExecutor

def get_num(num):
	r = requests.get('http://httpbin.org/get?a={}'.format(num))
	return json.loads(r.content)['args']['a']

nums = range(10)
start = time.time()

with ProcessPoolExecutor(max_workers=5) as executor:
	result = executor.map(get_num, nums)
	print(result)
	print(list(result))



print(time.time()-start)



"""
<itertools.chain object at 0x108e58358>
['0', '1', '2', '3', '4', '5', '6', '7', '8', '9']
1.0261192321777344
"""
```

### 关于map和submit的区别: 
- 观察上面两个例子的结果, map可以保证输出的顺序, submit输出的顺序是乱的
- 如果你要提交的任务的函数是一样的，就可以简化成map。但是假如提交的任务函数是不一样的，或者执行的过程之可能出现异常（使用map执行过程中发现问题会直接抛出错误）就要用到submit


### 资料

- [官方文档]( https://docs.python.org/3/library/concurrency.html)
- [中文翻译](https://www.bwangel.me/2016/09/23/concurrent-futures/)
- [董伟明-推荐阅读]( http://www.dongwm.com/archives/%E4%BD%BF%E7%94%A8Python%E8%BF%9B%E8%A1%8C%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B-PoolExecutor%E7%AF%87/)


<!--more-->
