---
title: property属性
date: 2017-08-29 15:05:59
categories: Python
tags:
---


---
title: property属性
date: 2017-08-08 08:13:35
categories: Python
tags:
---

`property`是实现数据可管理性的内建数据类型, 其实质是一种特殊的数据描述符(实现了`__get__()`和`__set__`方法, 称为数据描述符)



property实现形式为`property(fget=None, fset=None, fdel=None, doc=None)`

下面两种方法等价, 不过后一种比较常见

```python
class Test(object):
	def __init__(self, age):
		self._age = age

	def get_age(self):
		print('get method')
		return self._age

	def set_age(self, value):
		print('set meghod')
		if value < 0 or value > 200:
			raise Exception('ERROR')
		self._age = value

	def del_age(self):
		print('del method')
		del self._age

	age = property(get_age, set_age, del_age)


if __name__ == '__main__':
	t = Test(20)
	print(t.age)
	t.age = 50
	print t.age
	# del t.age
	t.age = -1

"""
get method
20
set meghod
get method
50
set meghod
Traceback (most recent call last):
  File "/Users/qinjianxun/Desktop/test.py", line 30, in <module>
    t.age = -1
  File "/Users/qinjianxun/Desktop/test.py", line 15, in age
    raise Exception('ERROR')
Exception: ERROR
"""


```



```python
class Test(object):
	def __init__(self, age):
		self._age = age

	@property
	def age(self):
		print('get method')
		return self._age

	@age.setter
	def age(self, value):
		# 此处可以检查value的合法性
		print('set meghod')
		if value < 0 or value > 200:
			raise Exception('ERROR')
		self._age = value

	@age.deleter
	def age(self):
		print('del method')
		del self._age


if __name__ == '__main__':
	t = Test(20)
	print(t.age)
	t.age = 50
	print t.age
	# del t.age
	t.age = -1

"""
get method
20
set meghod
get method
50
set meghod
Traceback (most recent call last):
  File "/Users/qinjianxun/Desktop/test.py", line 30, in <module>
    t.age = -1
  File "/Users/qinjianxun/Desktop/test.py", line 15, in age
    raise Exception('ERROR')
Exception: ERROR
"""


```

property的作用相当于一个分发器, 对某个属性的访问会通过中间一层的`fget`函数, 赋值对应`fset`函数等



### 使用property控制权限

以下示例没有给`fset`方法, 所以赋值会报错

**注意**: 并不能真正的控制权限, 只是设置了一些障碍

```python
class Test(object):
	def __init__(self, age):
		self._age = age

	@property
	def age(self):
		print('get method')
		return self._age


if __name__ == '__main__':
	t = Test(20)
	print(t.age)
	t.age = 50


"""
get method
20
Traceback (most recent call last):
  File "/Users/qinjianxun/Desktop/test.py", line 14, in <module>
    t.age = 50
AttributeError: can't set attribute
"""


```


#### 参考
- <<改善python程序的91条建议>>

