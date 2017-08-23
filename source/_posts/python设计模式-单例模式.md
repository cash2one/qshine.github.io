---
title: python设计模式-单例模式
date: 2017-08-16 23:03:43
categories: 设计模式
tags:
  - python
---

__单例模式(Singleton Pattern)__是一个重要的设计模式, 不管是使用还是面试, 当在某个系统中希望某个类只能出现一个实例时, 可以使用单例模式, 比如说在游戏中有一个`sun(太阳)类`, 要确保只能有一个的时候, 同时使用单例模式可以节约内存资源

下面列出实现单例模式的几种方式

### 使用`__new__`

```python
class Singleton(object):
    # 使用类变量来关联
	_instance = None
	def __new__(cls, *args, **kwargs):
		if not cls._instance:
			cls._instance = super(Singleton, cls).__new__(cls, *args, **kwargs)
		return cls._instance


class MyClass(Singleton):
	def get(self):
		pass


if __name__ == '__main__':
	m1 = MyClass()
	m2 = MyClass()

	print id(m1), id(m2)
	print id(m1) == id(m2)

```

<!--more-->

### 使用模块

把相关的函数或数据定义在一个模块中, 便可以实现一个单例对象, 应为模块在第一次导入的时候会生成`.pyc`文件, 第二次导入的时候, 就会直接加载`.pyc`, 并不会再次运行代码

```python
# Singleleton.py
class Sun(object):
	def __init__(self):
		pass
	def rise(self):
		pass
	def down(self):
		pass
```

使用

```python
from Singleleton import sun

sun.down()
```



### 使用装饰器

```python
import functools

def singleton(cls):
	instances = {}
	@functools.wraps(cls)
	def getinstance(*args, **kwargs):
		if cls not in instances:
			instances[cls] = cls(*args, **kwargs)
		return instances[cls]
	return getinstance


@singleton
class MyClass(object):
	def get(self):
		pass


if __name__ == '__main__':
	m1 = MyClass()
	m2 = MyClass()

	print id(m1), id(m2)
	print id(m1) == id(m2)


```

通过装饰器来将某各类存入`instances`字典中, 如果不存在则添加, 某则直接返回

### 参考资料

- [python单例模式](http://python.jobbole.com/87294/)
- `<<改善python的91条建议>>`


