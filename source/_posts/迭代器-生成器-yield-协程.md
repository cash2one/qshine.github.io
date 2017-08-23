---
title: 迭代器/生成器/yield/协程
date: 2017-08-23 12:03:40
tags: Python
---

关于yield关键字可以看[stackoverflow回答](https://stackoverflow.com/questions/231767/what-does-the-yield-keyword-do), 总结一下几个概念

### 可迭代对象(iterable)

 所有能使用`for ... in ...`操作的对象都是可迭代对象, 比如列表, 字符串等

```python
In [1]: for i in range(3):
   ...:     print i
   ...:
0
1
2

In [2]:
```

### 迭代器(iterator)

 能使用`next()`方法的对象, 任何实现`__iter__`和`next()`方法的对象都是迭代器, 其中`__iter__`返回迭代器自身, 调用next()方法, 直到遇到`StopIteration`

```python
In [10]: nums = range(3)

In [11]: iter_nums = iter(nums)

In [12]: dir(iter_nums)
Out[12]:
['__class__',
 '__delattr__',
 '__doc__',
 '__format__',
 '__getattribute__',
 '__hash__',
 '__init__',
 '__iter__',
 '__length_hint__',
 '__new__',
 '__reduce__',
 '__reduce_ex__',
 '__repr__',
 '__setattr__',
 '__sizeof__',
 '__str__',
 '__subclasshook__',
 'next']

In [13]:

In [13]: iter_nums.__iter__
Out[13]: <method-wrapper '__iter__' of listiterator object at 0x10bde2f90>

In [14]:

In [14]: iter_nums.next()
Out[14]: 0

In [15]: iter_nums.next()
Out[15]: 1

In [16]:
```

### 生成器(generator)

 仅仅需要一个yield关键字, 返回值不需要return, 仅仅需要yield, 在每次进行迭代时返回一个值，直到遇到`StopIteration`异常结束

```python
In [16]: def test():
    ...:     for i in range(3):
    ...:         yield i
    ...:

In [17]: t = test()

In [18]: dir(t)
Out[18]:
['__class__',
 '__delattr__',
 '__doc__',
 '__format__',
 '__getattribute__',
 '__hash__',
 '__init__',
 '__iter__',
 '__name__',
 '__new__',
 '__reduce__',
 '__reduce_ex__',
 '__repr__',
 '__setattr__',
 '__sizeof__',
 '__str__',
 '__subclasshook__',
 'close',
 'gi_code',
 'gi_frame',
 'gi_running',
 'next',
 'send',
 'throw']

In [19]:

In [19]: t.next()
Out[19]: 0

In [20]: t.next()
Out[20]: 1
```

上例可以看到同样包含`__iter__`和`next()`

```python
In [21]: nums = [i**2 for i in range(3)]

In [22]: nums
Out[22]: [0, 1, 4]

In [23]:

In [23]: iter_nums = (i**2 for i in range(3))

In [24]: iter_nums
Out[24]: <generator object <genexpr> at 0x10bdc8500>

In [25]: for i in iter_nums:
    ...:     print i
    ...:
0
1
4

```

通过`()`来定义生成器表达式



### 理解send

1. 注意以下几个例子区别

   - 调用`next()`
     
     ```python
     In [13]: def test():
     ...:     for i in range(3):
     ...:         yield i
     ...:

     In [14]:

     In [14]: t = test()

     In [15]: next(t)
     Out[15]: 0

     In [16]: next(t)
     Out[16]: 1

     In [17]: next(t)
     Out[17]: 2

     In [18]: next(t)
     ---------------------------------------------------------------------------
     StopIteration                             Traceback (most          recent call last)
     <ipython-input-18-9494367a8bed> in <module>()
     ----> 1 next(t)

     StopIteration:

     In [19]:
     ```

   - 使用`send(None)`
     ```python
     In [7]: def test():
     ...:     for i in range(3):
     ...:         yield i
     ...:

     In [8]:
    
     In [8]: t = test()
    
     In [9]: t.send(None)
     Out[9]: 0
    
     In [10]: t.send(None)
     Out[10]: 1
    
     In [11]: t.send(None)
     Out[11]: 2
    
     In [12]: t.send(None)
     ---------------------------------------------------------------------------
     StopIteration                             Traceback (most recent call last)
     <ipython-input-12-91c226651b6e> in <module>()
     ----> 1 t.send(None)
    
     StopIteration:
    
     In [13]:
     ```
     以上两个例子可以看出`next()`其实就是`send(None)`, 但是`send()`可以向生成器函数传入参数

   - 使用send函数输出`hello world`
   ```python
   In [19]: def test(r):
    ...:     n = yield r
    ...:     yield n
    ...:

    In [20]: t = test('hello')

    In [21]: t.send(None)    # 注意第一步必须是发送None
    Out[21]: 'hello'

    In [22]: t.send('world')
    Out[22]: 'world'

    In [23]:
   ```
   生成器函数最大的特点是可以接受外部传入的一个变量，并根据变量内容计算结果后返回. 
   上例中的执行步骤:
   -  执行`t.send(None)`或`next(t)`来启动生成器函数, 执行到第一个yield语句位置, 此时并没有给n赋值, 所以会返回r的值`hello`. **注意**: 在启动生成器函数时只能`send(None)`, 不能放松其它信息
   - 执行`t.send('world')`会赋值给n, 然后走到下一个yield语句返回`n`并挂起


### 协程实现生产者和消费者
> 协程看上去也是子程序，但执行过程中，在子程序内部可中断，然后转而执行别的子程序，在适当的时候再返回来接着执行。

以下代码是[廖雪峰老师的文章](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432090171191d05dae6e129940518d1d6cf6eeaaa969000#0)
```python
import threading

def consumer():
	r = ''
	while True:
		n = yield r
		# if not n:
		# 	return
		print 'CONSUMER: {}, {}'.format(n, threading.currentThread())
		r = '200 ok'

def product(c):
	c.send(None)
	num = 0
	while num < 3:
		num += 1
		print 'PRODUCT: {}, {}'.format(num, threading.currentThread())
		n = c.send(num)
		print n
	c.close()

c = consumer()
product(c)


"""
PRODUCT: 1, <_MainThread(MainThread, started 140736345928640)>
CONSUMER: 1, <_MainThread(MainThread, started 140736345928640)>
200 ok
PRODUCT: 2, <_MainThread(MainThread, started 140736345928640)>
CONSUMER: 2, <_MainThread(MainThread, started 140736345928640)>
200 ok
PRODUCT: 3, <_MainThread(MainThread, started 140736345928640)>
CONSUMER: 3, <_MainThread(MainThread, started 140736345928640)>
200 ok
"""

```
上例可以由一个线程在执行
- c.send(None)启动consumer
- c.send(num)发送num给consumer中的`n`
- consumer中的`yield r`把r的值赋给`product中的n`


### 资料
- [yield - stackoverflow](https://stackoverflow.com/questions/231767/what-does-the-yield-keyword-do)
- [深入理解python中的生成器 - 伯乐在线](http://python.jobbole.com/81911/)
- [协程 - 廖雪峰](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432090171191d05dae6e129940518d1d6cf6eeaaa969000#0)
