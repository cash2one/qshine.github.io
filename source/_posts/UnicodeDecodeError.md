---
title: UnicodeDecodeError
date: 2017-07-09 22:46:55
categories: Python
tags:
---

**UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 0: ordinal not in range(128)**

该错误经常碰到, 一般是因为脚本中有中文, 解决办法一般也很简单, 把下列内容加入文件开头

```python
import sys
reload(sys)
sys.setdefaultencoding('utf-8')
```

但是真实的原因并不知道, 终于在`<<改善python程序的91条建议>>`找到相关内容

<!--more-->

python默认的编码是`ASCII`编码

```python
In [1]: import sys

In [2]: sys.getdefaultencoding()
Out[2]: 'ascii'

In [3]: 

```

编写如下脚本

```python
# coding:utf-8

s = '中文测试' + u' Test'

print s
```

运行会报如下错误

```
ql@ubuntu:~$ python test.py
Traceback (most recent call last):
  File "test.py", line 3, in <module>
    s = '中文测试' + u' Test'
UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 0: ordinal not in range(128)
```

在做加法运算的时候, 左边为str, 右边是unicode, 计算的时候python会将左边的中文字符串转换为`Unicode`再与右边的`Unicode`进行连接, 将str转换为Unicode时使用系统默认的`ASCII`编码(即'中文测试'.decode('ascii'), 但是`"中"`子对应的编码值是214, **当编码值在`0-127`的时候Unicode和Ascii是兼容的, 转换没有问题, 但是当值大于128的时候, ASCII编码便不能正确处理该情况, 会抛出UnicodeDecodeError异常**

<!--more-->




