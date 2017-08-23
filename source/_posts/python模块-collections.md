---
title: python模块-collections
date: 2017-08-10 06:16:04
categories: Python
tags:
  - python模块
---

在python进阶过程中很多重要的模块是必须掌握的, 虽然或多或少使用过, 但是并没有记录下来这部分内容, 所以计划记录其中较重要和经常使用的内容方便之后自己参考



`collections`模块提供了以下几种方法

#### defaultdict

该方法不需要检查某个键是否存在

```python
from collections import defaultdict

peoples = [
	[20, 'jack'],
	[15, 'tom'],
	[20, 'jenny']
]

<!--more-->

# 参数要传入字典value的数据类型
result = defaultdict(list)

for age, name in peoples:
	result[age].append(name)

print result
print result[20]
print result[99]    # 输入的键不存在是不报错


"""
defaultdict(<type 'list'>, {20: ['jack', 'jenny'], 15: ['tom']})
['jack', 'jenny']
[]
"""
```

#### namedtuple()

构造可以通过名称来访问的对象,  增强代码可读性

```python
from collections import namedtuple

people = [
	[17, 'tom'],
	[20, 'jack']
]

People = namedtuple('people_info', ['age', 'name'])

for p in people:
	info = People._make(p)    # 调用_make方法来构造
	print info
	print info.name    # 可以直接通过名称来访问

"""
people_info(age=17, name='tom')
tom
people_info(age=20, name='jack')
jack
"""
```



#### deque()

`deque`是双端队列，与list最大的区别:

- list: 两端增加或删除的`时间复杂度`会随着元素数量的增加线性增长, 即为`O(n)`
- deque: 两端增加或删除的`时间复杂度`不变, 保持`O(1)`

方法和list的操作基本一样

```python
from collections import deque

data = range(3)

print data

d = deque(data)
d.append('b')
d.appendleft('a')

print d

d.pop()
print d
d.remove(2)
print d

"""
[0, 1, 2]
deque(['a', 0, 1, 2, 'b'])
deque(['a', 0, 1, 2])
deque(['a', 0, 1])
"""
```

除了以上之外还有一个`rotate`, 可以使列表旋转

```python
from collections import deque

a = range(4)
print a
d = deque(a)
print d
d.rotate(1)    # 向右偏移一位
print d

"""
[0, 1, 2, 3]
deque([0, 1, 2, 3])
deque([3, 0, 1, 2])
"""
```



#### Counter

该函数可以非常方便统计字符创中某个字符出现数量

```python
from collections import Counter

string = 'hello china'

c = Counter(string)

print c.most_common()     # 对字符出现次数进行排序
print c.most_common(2)    # 出现平率最高的两个字符和次数

"""
[('h', 2), ('l', 2), ('a', 1), (' ', 1), ('c', 1), ('e', 1), ('i', 1), ('o', 1), ('n', 1)]
[('h', 2), ('l', 2)]
"""
```



#### OrderedDict

该方法可以为我们提供有序的字典结构

```python
from collections import OrderedDict

data = {
	'C': 3,
	'B': 2,
	'A': 1
}

for k,v in data.items():
	print k, v

print '---------'

# 按照值进行排序
ordered_data = OrderedDict(sorted(data.items(), key=lambda x: x[1]))

for k,v in ordered_data.items():
	print k,v

"""
A 1
C 3
B 2
---------
A 1
B 2
C 3
"""

```








