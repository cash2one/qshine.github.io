---
title: python模块-itertools
date: 2017-08-11 06:16:04
categories: Python
tags:
  - python模块
---

`itertools`模块所包含的所有函数都将构造和返回迭代器, 迭代器相比列表来说好处是`延迟计算`, 所以此模块返回的迭代器都可以与for语句结合使用

__注意:__ python2和python3中某些函数有变化, 具体参考官方文档

### chain(iter1, iter2, ..., iterN)

将多个迭代器联合起来并返回一个新的迭代器

```python
In [7]: import itertools
    
In [8]: res = itertools.chain([1,2,3], 'abc')
In [9]: list(res)
Out[9]: [1, 2, 3, 'a', 'b', 'c']

In [10]: 
```

<!--more-->

### accumulate(iter), python3存在

累加

```python
In [6]: data = range(4)

In [7]: list(data)
Out[7]: [0, 1, 2, 3]

In [8]: res = itertools.accumulate(data)

In [9]: list(res)
Out[9]: [0, 1, 3, 6]

In [10]: 

```

### combinations(itertable, n)

返回itertable中所有`长度为n`的子序列, 顺序为`itertable`的顺序, 不允许重复

```python
In [2]: res = itertools.combinations([1,2,3,4], 2)

In [3]: list(res)
Out[3]: [(1, 2), (1, 3), (1, 4), (2, 3), (2, 4), (3, 4)]

In [4]: 
```

### combinations_with_replacement(iterable, n)

返回iterable中长度为n的子序列, 允许元素重复

```python
In [4]: res = itertools.combinations_with_replacement([1,2,3], 2)

In [5]: list(res)
Out[5]: [(1, 1), (1, 2), (1, 3), (2, 2), (2, 3), (3, 3)]

In [6]: 
```

### islice(iterable, start, stop, [step])

用于截取迭代器

```python
In [4]: res = itertools.islice('abcde', 0, 3, 1)

In [5]: list(res)
Out[5]: ['a', 'b', 'c']

```



### count(start=0, step=1)

一个计数器, 可以指定起始位置和步长, 没有结束位置, 所以和islice`结合

```python
In [10]: data = itertools.count(10, 2)

In [11]: res = itertools.islice(data, 0, 5, 1)

In [12]: list(res)
Out[12]: [10, 12, 14, 16, 18]

In [13]:
```



### cycle(itertable)

循环指定的itertable

```python
In [15]: res = itertools.islice(itertools.cycle('abc'), 0, 7)

In [16]: list(res)
Out[16]: ['a', 'b', 'c', 'a', 'b', 'c', 'a']

In [17]: 
```



### dropwhile(predicate, iterable)

从`iterable`中删除元素, 只要元素符合`predicate`的条件

```python
In [17]: res = itertools.dropwhile(lambda x:x<4, range(10))

In [18]: list(res)
Out[18]: [4, 5, 6, 7, 8, 9]

In [19]: 
```



### groupby(iterable[, key])

对`iterable`按照指定条件进行分组

```python
In [27]: res = itertools.groupby(range(10), key=lambda x: x < 5)

In [28]: 

In [28]: for k, v in res:
   ....:     print k, list(v)
   ....:     
True [0, 1, 2, 3, 4]
False [5, 6, 7, 8, 9]

In [29]: 
```



### ifilter(predicate, iterable)

从iterable中筛选出条件为True的元素

```python
In [30]: res = itertools.ifilter(lambda x:x<4, range(10))

In [31]: list(res)
Out[31]: [0, 1, 2, 3]

In [32]: 
```



### ifilterFalse(predicate, iterable)

筛选出条件为Flase的元素, 注意python3中为`filterflase`

```python
In [32]: res = itertools.ifilterfalse(lambda x:x<4, range(10))

In [33]: list(res)
Out[33]: [4, 5, 6, 7, 8, 9]

In [34]:
```



### permutations(iterable[,n])

从`iterable`中返回指定长度n的所有组合

```python
In [36]: list(itertools.permutations('abc', 2))
Out[36]: [('a', 'b'), ('a', 'c'), ('b', 'a'), ('b', 'c'), ('c', 'a'), ('c', 'b')]

In [37]:
```



### product(iterable[,repeat])

多个可迭代对象的笛卡尔积

```python
In [41]: res = itertools.product('abc', [1,2,3])

In [42]: list(res)
Out[42]: 
[('a', 1),
 ('a', 2),
 ('a', 3),
 ('b', 1),
 ('b', 2),
 ('b', 3),
 ('c', 1),
 ('c', 2),
 ('c', 3)]

In [43]: 

```



该模块还有很多其它方法, 具体参考官方文档

- [python2官方文档](https://docs.python.org/2/library/itertools.html)
- [python3官方文档](https://docs.python.org/3/library/itertools.html)








