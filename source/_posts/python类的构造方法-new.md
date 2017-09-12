---
title: python类的构造方法__new__
date: 2017-08-24 00:23:49
categories: Python
tags:
---

- `__new__`: 类的构造方法, 当返回类的对象的时候会自动调用`__init__`进行初始化, 没有对象返回的话则不会调用
- `__init__`: 类的对象创建好后进行变量初始化的方法

```python
class A(object):

  	# self代表实例
	def __init__(self, name):
		self.name = name
		print 'init'

    # cls代表类
	def __new__(cls, *args, **kwargs):
		print 'new'
		return super(A, cls).__new__(cls, *args, **kwargs)

a = A('china')


"""
new
init
"""
```

上例可以看出首先运行的是`__new__`方法, 创建好类的对象后才运行`__init__`方法初始化变量


### 总结
- `__init__()`并不是类的构造方法, `__init__()`的工作是在类的对象创建好后进行变量的初始化
- `__new__()`方法是真正的创建实例, 是类的构造方法
- 当需要覆盖`__new__()`和`__init__()`方法的时候要保证参数一致
- 当子类继承自**不可变类型(str, tuple)**等类型或使用**工厂方法**的时候需要覆盖该方法


### 资料
- <<改善python程序的91条建议-建议55>>

<!--more-->
