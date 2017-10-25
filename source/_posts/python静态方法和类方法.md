---
title: python静态方法和类方法
date: 2017-09-25 22:38:00
categories: Python
tags:
---

除了在类里定义实例方法外, 还能定义两种函数
1. 静态方法`@staticmethod`
静态方法实际就是普通函数, 只是由于某种原因要定义在类里面, **不需要self**参数, 通过类和实例都能调用

2. 类方法`@classmethod`
必须有一个表示其调用类的参数, 通常用**cls**表示, 在类方法中不能使用`self`来调用其它方法和属性, 必须使用`cls`来调用

### 实例
```python
# coding:utf-8


class A(object):
	@staticmethod
	def get(num):
		print('staticmethod is called', num)

	@classmethod
	def post(cls, num):
		cls.get('cls called')
		print('classmethod is called', cls,  num)



if __name__ == '__main__':
	a = A()
	a.get(100)    # 实例调用静态方法
	A.get(200)    # 类调用静态方法

	a.post(300)   # 实例调用类方法
	A.post(400)   # 类调用类方法



"""
('staticmethod is called', 100)
('staticmethod is called', 200)
('staticmethod is called', 'cls called')
('classmethod is called', <class '__main__.A'>, 300)
('staticmethod is called', 'cls called')
('classmethod is called', <class '__main__.A'>, 400)
"""
```


### 类计数器
```python
# coding:utf-8


class A(object):
	counter = 0

	def __init__(self):
		A.counter += 1

	@classmethod
	def count(cls):
		return A.counter


if __name__ == '__main__':
	a = A()
	b = A()
	c = A()

	print a.count()


"""
表示创建了3个A对象
3
"""
	



"""
('staticmethod is called', 100)
('staticmethod is called', 200)
('staticmethod is called', 'cls called')
('classmethod is called', <class '__main__.A'>, 300)
('staticmethod is called', 'cls called')
('classmethod is called', <class '__main__.A'>, 400)
"""
```
