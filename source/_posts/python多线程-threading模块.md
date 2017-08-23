---
title: python多线程-threading模块
date: 2017-06-15 01:59:54
categories: Python
tags:
---

本篇主要内容来自[threading模块](http://python.jobbole.com/81546/), 部分是我的读书笔记,  因为多线程这块内容很多, 所以当做复习一遍, 也算是备份方便之后自己参考

**注意:** python3中取消了`thread`模块, 改为`_thread`模块, 从命名上可以看出使用`threading`比较好

python2使用了`thread`和`threading`来创建多线程, 区别在于
> `thread`模块提供了多线程底层模块支持, 以低级原始的方式来处理和控制线程, 使用起来比较复杂
> `threading`模块对`thread`进行了封装, 将线程的操作对象化

`threading`和`multiprocessing`模块的接口是很类似的, 多进程可以参考[python多进程](https://qshine.github.io/2017/06/09/Python%E5%A4%9A%E8%BF%9B%E7%A8%8B%E7%AC%94%E8%AE%B0/)

<!--more-->

### threading.Thread
使用`threading`创建线程的两种方法
1. 继承`threading.Thread`类, 重写`run`方法
   ```python
   #!/usr/bin/env python
   # coding:utf-8

   import time
   import threading

   count = 0

   class Counter(threading.Thread):
   	"""
   	继承threading.Thread
   	"""
   	def __init__(self, lock):
   		# 显示调用父类的初始化函数
   		super(Counter, self).__init__()
   		self.lock = lock

   	def run(self):
   		global count
   		self.lock.acquire()
   		for i in range(100):
   			count += 1
   		self.lock.release()

   # 生成锁对象
   lock = threading.Lock()

   # 开启5个线程
   for i in range(5):
   	Counter(lock).start()

   time.sleep(5)
   print count    # 输出500
   ```
   run和start都是继承自Thread, 首先由start开启线程, 然后run才执行
2. 创建一个`threading.Thread`对象, 在它的初始化函数`(__init__)`中将调用对象作为参数传入
   ```python
   #!/usr/bin/env python
   # coding:utf-8

   import time
   import threading

   count = 0
   lock = threading.Lock()

   def add():
   	global count, lock
   	lock.acquire()
   	for i in range(100):
   		count += 1
   	lock.release()

   # 创建5个线程并开启
   threads = []
   for i in range(5):
   	t = threading.Thread(target=add, args=())
   	threads.append(t)

   for t in threads:
   	t.start()

   time.sleep(5)
   print count    # 输出500
   ```

   - `target`是一个可调用的对象, 通常是一个函数
   - `args`是要出入这个对象的参数

<!--more-->

### threading.join
阻塞主进程的执行, 看下面两个程序的输出就能看出来
1. 不加`join`
   ```python
   #!/usr/bin/env python
   # coding:utf-8

   import time
   import threading

   def test():
   	print 'threading start ...  '
   	time.sleep(5)
   	print 'threading end !'

   t = threading.Thread(target=test, args=())
   t.start()

   print 'end'

   """
   threading start ...  
    end
   threading end !
   """
   ```

   可以发现主进程并没有等待子进程结束后才执行, 而是主进程先结束的

2. 加入`join`, 会阻塞主程序的执行, 等到子进程结束后, 主进程才结束
   ```python
   #!/usr/bin/env python
   # coding:utf-8

   import time
   import threading

   def test():
   	print 'threading start ...  '
   	time.sleep(5)
   	print 'threading end !'

   t = threading.Thread(target=test, args=())
   t.start()
   t.join()

   print 'end'

   """
   threading start ...  
   threading end !
   end
   """
   ```

### threading.daemon
该属性必须在`start`方法前加, 如果设置`daemon=True`, 则表示`主线程的推出不用等待子线程完成`
1. 不加该属性(参考上例中第一个程序的输出)
2. 设置`daemon=True`
   ```Python
   #!/usr/bin/env python
   # coding:utf-8

   import time
   import threading

   def test():
   	print 'threading start ...  '
   	time.sleep(5)
   	print 'threading end !'

   t = threading.Thread(target=test, args=())
   t.daemon = True
   t.start()

   print 'end'

   """
   threading start ...  end
   """

   ```
   主进程结束后程序全部结束

### RLock和Lock
- `Lock`: acquire和release必须成对出现, 并且调用acquire后下一个必须是release, 否则会造成死锁
- `RLock`: 可重入锁, 即统一线程中可以进行多次`acquire`

### threading.Condition
默认在内部维护了一个所对象(默认是RLock), 提供了如下方法(__只有在acquire后才能调用__)
- __wait([timeout])__
  释放占用的锁, 线程挂起, 只有`等到通知或超时`才能继续执行
- __notify()__
  唤醒挂起的线程, __不会释放占用的锁__
- __notifyAll()__
  唤醒所有挂起的线程, __不会释放占用的锁__

```python
#!/usr/bin/env python
# coding:utf-8

"""
假设这个游戏由两个人来玩，一个藏(Hider)，一个找(Seeker)。
游戏的规则如下：
    1. 游戏开始之后，Seeker先把自己眼睛蒙上，蒙上眼睛后，就通知Hider；
    2. Hider接收通知后开始找地方将自己藏起来，藏好之后，再通知Seeker可以找了；
    3. Seeker接收到通知之后，就开始找Hider。Hider和Seeker都是独立的个体，在程序中用两个独立的线程来表示，
    在游戏过程中，两者之间的行为有一定的时序关系，我们通过Condition来控制这种时序关系。
"""
import time
import threading


class Hider(threading.Thread):
	"""藏匿着"""
	def __init__(self, cond, name):
		super(Hider, self).__init__()
		self.cond = cond
		self.name = name

	def run(self):
		# 先运行Seeker
		time.sleep(1)
		self.cond.acquire()
		print '{}: 我已经藏好了，你快来找我吧'.format(self.name)
		self.cond.notify()    # 通知seeker
		self.cond.wait()    # 挂起等待, 释放锁
		print '{}: 被你找到了，哎'.format(self.name)
		self.cond.notify()
		self.cond.release()



class Seeker(threading.Thread):
	"""寻找者"""
	def __init__(self, cond, name):
		super(Seeker, self).__init__()
		self.cond = cond
		self.name = name

	def run(self):
		self.cond.acquire()
		print '{}: 我已经把眼睛蒙上了'.format(self.name)
		self.cond.wait()
		print '{}: 我找到你了'.format(self.name)
		self.cond.notify()
		self.cond.wait()
		print '{}: 我赢了'.format(self.name)
		self.cond.release()



cond = threading.Condition()

seeker = Seeker(cond, 'seeker')
hider = Hider(cond, 'hider')

seeker.start()
hider.start()


"""
seeker: 我已经把眼睛蒙上了
hider: 我已经藏好了，你快来找我吧
seeker: 我找到你了
hider: 被你找到了，哎
seeker: 我赢了
"""
```



### threading.Event
与Condition类似, 但是通过设置内部标识符来实现, 有以下几种方法
- __wait([timeout])__
  堵塞线程直到Event内部对象标识符为True或超时
- __set()__
  将标识符设为True
- __clear__
  将标识符设为False
- __isSet()__
  判断标识符是否为True

```python
# coding:utf-8

import threading
import time


def countdown(n, started_evt):
    print '开始执行'
    while n > 0:
        print n
        n -= 1
        time.sleep(1)
    # 设置标志位为True, 通知运行结束
    started_evt.set()

# 创建一个event实例
started_evt = threading.Event()

t = threading.Thread(target=countdown, args = (5, started_evt))
t.start()

# 程序挂起, 等待通知, 保证下面的输出在最后面
started_evt.wait()

print '正在运行......'


"""
开始执行
5
4
3
2
1
正在运行......
"""
```





### threading.Timer
定时执行
```python
# coding:utf-8

import threading

def test(i):
	print i

# 定时5秒后执行
t = threading.Timer(5, test, args=('hello world', ))
t.start()
```

使用`threading.Timer`实现的循环器

```python
# coding:utf-8

import threading

string = 'hello world'

def test(content):
	print content


def set_interval(func, second):
	def wrapper():
		func(string)
		set_interval(func, second)
	t = threading.Timer(second, wrapper)
	t.start()
	return t

set_interval(test, 1)


```





### 其他方法
1. 获取当前活动的线程个数
   - threading.active_count()
   - threading.activeCount()
2. 获取当前线程对象
   - threading.current_thread()
   - threading.currentThread()

还有很多其它方法



### 资料

- [官方文档](https://docs.python.org/2/library/threading.html)
- [threading模块](http://python.jobbole.com/81546/)































