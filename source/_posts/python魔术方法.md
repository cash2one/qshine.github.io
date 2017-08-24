---
title: python魔术方法
date: 2017-08-24 20:48:17
categories: Python
tags:
---



```python
class A(object):
	def __init__(self, data):
		self.data = data

	def __len__(self):
		return len(self.data)

	def __getitem__(self, item):
		return self.data[item]

	def __setitem__(self, item, value):
		self.data[item] = value

	def __delitem__(self, item):
		del self.data[item]

	def __contains__(self, item):
		return True if item in self.data else False


data = {'name':'Tom', 'age':20}

a = A(data)

print(a.data)

print(a['name'])
a['address'] = 'china'
print(a['address'])
del a['name']
print('age' in a)
print(a.data)
print(len(a))

"""
{'name': 'Tom', 'age': 20}
Tom
china
True
{'age': 20, 'address': 'china'}
2
"""
```



```python
class Fibs(object):
	def __init__(self):
		self.a = 0
		self.b = 1

	def next(self):
		self.a, self.b = self.b, self.a+self.b
		return self.a

	def __iter__(self):
		return self


f = Fibs()
print(f.next())
print(f.next())
print(f.next())
print(f.next())
print(f.next())
print(f.next())

"""
1
1
2
3
5
8
"""
```



