---
title: __getattr__和__getattribute__方法
date: 2017-08-25 00:06:11
categories: Python
tags:
---

`__getattr__`和`__getattribute__`都可以做实例属性的获取和拦截(**注意**仅对实例属性有效)



#### \__getattr__(self, item)

只有当对象不存在item属性的或属性不在实例的`__dict__`中, 属性不在其基类的`__dict__`中会调用该方法

```python
class A(object):
	def __init__(self, name):
		self.name = name

	def __getattr__(self, item):
		return 20

a = A('china')

# 访问实例存在的属性
print(a.name)    
# 访问实例不存在的属性, 调用__getattr__方法
print(a.age)


"""
china
20
"""
```

<!--more-->

#### \__setattr__(self, item, value)

注意下面的**错误**例子, 不能直接`self.item = value`这样调用, 因为这样调用会再次调用自己, 从而引发递归调用导致程序崩溃

```python
class A(object):
	def __init__(self, name):
		self.name = name

	def __setattr__(self, item, value):
		self.item = value


a = A('china')

print(a.name)    
# 设置新的属性
a.age = 200
print(a.age)


"""
Traceback (most recent call last):
  File "test.py", line 9, in <module>
    a = A('china')
  File "test.py", line 3, in __init__
    self.name = name
  File "test.py", line 6, in __setattr__
    self.item = value
  File "test.py", line 6, in __setattr__
    self.item = value
  File "test.py", line 6, in __setattr__
    self.item = value
  [Previous line repeated 327 more times]
RecursionError: maximum recursion depth exceeded
"""
```

正确的调用方法使用`self.__dict__[item] = value`

```python
class A(object):
	def __init__(self, name):
		self.name = name

	def __setattr__(self, item, value):
		self.__dict__[item] = value


a = A('china')

print(a.name)    
# 设置新的属性
a.age = 200
print(a.age)


"""
china
200
"""
```





####  \__getattribute__(self, item)

只存在于新式类当中, **如果重写了该方法**那么对于所有对象的访问都会经过该方法, 这也是和`__getattr__`的区别

要注意该方法的写法, 不然容易出现递归调用的情况, 

```python
class A(object):
	def __init__(self, name):
		self.name = name

	def __getattr__(self, item):
		return 20

	def __getattribute__(self, item):
		print('visit: {}'.format(item))
		# return object.__getattribute__(self, item)
		return self.__dict__[item]



a = A('china')
print(a.name)    
# 设置新的属性
print(a.age)


"""
......
  File "test.py", line 9, in __getattribute__
    print('visit: {}'.format(item))
RecursionError: maximum recursion depth exceeded while calling a Python object
"""
```

这是因为`return self.__dict__[item]`又会调用`__getattribute__(self, item)`方法, 一直循环下去

正确的写法如下:

```python
class A(object):
	def __init__(self, name):
		self.name = name

	def __getattr__(self, item):
		return 20

	def __getattribute__(self, item):
		print('visit: {}'.format(item))
		# return self.__dict__[item]    # 错误写法

		# 下面两种写法都正确
		# return super(A, self).__getattribute__(item) 
		return object.__getattribute__(self, item)




a = A('china')
print(a.name)    
# 设置新的属性
print(a.age)


"""
visit: name
china
visit: age
20
"""
```



#### 资料
<<改善Python程序的91条建议>>
