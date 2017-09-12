---
title: python名字查找机制
date: 2017-09-10 14:54:41
categories: Python
tags:
---

python中对任何变量的创建, 查找, 修改都是在**命名空间(namespace)**进行的, 命名空间分为
- 局部作用域(Local)
  如函数内部的变量声明
- 全局作用域(Global)
  定义在python模块中的变量名
- 嵌套作用域(enclosing function locals)
  多重函数嵌套情况, 嵌套情况下global并不起作用
    ```python
    def out():
    	string = "a"
    	def inner():
    		global string
    		string = "b"
    		print string
    	inner()
    	print string

    out()
    
    """
    b
    a
    """
    ```
- 内置作用域(Build-in)


python中访问一个变量的时候遵循**LEGB**原则(本地 --> 嵌套 --> 全局 --> 内置), 在第一个找到的地方停止搜索


### 采坑记录
1. UnboundLocalError
```python
def foo():
	a = 1
	def bar():
		b = a * 2
		a = b + 1
		print b
	return bar

foo()()


"""
Traceback (most recent call last):
  File "/Users/qinjianxun/Desktop/test.py", line 9, in <module>
    foo()()
  File "/Users/qinjianxun/Desktop/test.py", line 4, in bar
    b = a * 2
UnboundLocalError: local variable 'a' referenced before assignment
"""
```
原因在于编译为字节码的时候`a=b+1`会认为`a`是局部变量, 等到运行的时候执行到`b = a * 2`找到a局部变量, 但是没有任何值. 避免这种错误有以下两种方法

1. python2环境
```python
a = 1

def foo():
	global a
	def bar():
		global a
		b = a * 2
		a = b + 1
		print b
	return bar

foo()()


"""
2
"""
```
2. python3环境, 使用nonlocal关键字
```python


def foo():
	a = 1
	def bar():
		nonlocal a
		b = a * 2
		a = b + 1
		print(b)
	return bar

foo()()


"""
2
"""
```


