---
title: Python多进程笔记
date: 2017-06-09 04:41:40
categories: Python
tags:
---

多进程笔记

__原文__:
- [伯乐在线-python多进程编程](http://python.jobbole.com/82045/)

__资料:__
- [Vamei-多进程初步](http://www.cnblogs.com/vamei/archive/2012/10/12/2721484.html)
- [Vamei-多进程探索](http://www.cnblogs.com/vamei/archive/2012/10/13/2722254.html)

#### 一. Process类

__常用方法和属性__

- `pid`: 进程号
- `name`: 进程名


- `is_alive()`: 进程是否为激活状态
- `join([timeout])`
- `run()`
- `daemon`: 
  - 当在`start()`方法前加上`daemon=True`时候, 主进程结束的时候, 子进程也会结束
  - 不加的话主进程结束还会等待子进程结束
- `start()`: 启动进程
- `terminate()`
- `active_children()`: 未结束运行的子进程
- `multiprocessing.cpu_count()`: 计算cpu核
- `multiprocessing.current_process()`: 当前进程的对象

<!--more-->

#### 创建函数开启多个进程

```python
#!/usr/bin/env python
# coding:utf-8

import time
import multiprocessing
from multiprocessing import Process


def worker_1():
    time.sleep(4)
    print 'worker_1'

def worker_2():
    time.sleep(4)
    print 'worker_2'

if __name__ == '__main__':
    print 'cup num is: %d'%multiprocessing.cpu_count()
    p1 = Process(target=worker_1, args=())
    p2 = Process(target=worker_2, args=())
    p1.start()
    p2.start()
    for p in multiprocessing.active_children():
        print 'child p.name:{},  p.pid:{}'.format(p.name, p.pid)
    print 'done'

"""
cup num is: 1
child p.name:Process-1,  p.pid:15072
child p.name:Process-2,  p.pid:15073
done
worker_2
worker_1
"""

```



#### 将类定义为进程

```python
#!/usr/bin/env python
# coding:utf-8

import time
import multiprocessing


class TestProcess(multiprocessing.Process):
    """继承Process类"""
    def __init__(self, interval):
        super(TestProcess, self).__init__()
        self.interval = interval

    def run(self):
        """重写父类的run方法"""
        n = 1
        while True:
            print n
            time.sleep(self.interval)
            n += 1


if __name__ == '__main__':
    for i in range(2, 4):
        t = TestProcess(i)
        t.start()    # start方法会开启run方法


"""
1
1
2
2
3
3
4
5
4
6
5
7
"""
```

#### daemon属性

- 不加daemon

  ```python
  #!/usr/bin/env python
  # coding:utf-8

  import time
  import multiprocessing

  """
  不加daemon, 主进程结束后子进程并不会受影响也结束
  """

  def worker():
      for i in range(3):
          print i
          time.sleep(4)

  if __name__ == '__main__':
      p = multiprocessing.Process(target=worker, args=())
      p.start()
      print 'end'
      
  """
  end
  0
  1
  2
  """
  ```

- 加上daemon

  ```python
  #!/usr/bin/env python
  # coding:utf-8

  import time
  import multiprocessing

  """
  加上daemon, 主进程不会等待子进程结束, 同时当主进程结束时子进程也会结束
  必须在start()方法前加
  """

  def worker():
      for i in range(3):
          print i
          time.sleep(4)

  if __name__ == '__main__':
      p = multiprocessing.Process(target=worker, args=())
      p.daemon = True
      p.start()
      print 'end'

  """
  end
  """
  ```

#### join属性

```python
#!/usr/bin/env python
# coding:utf-8

import time
import multiprocessing

"""
join方法必须在start()方法后
"""

def worker():
    for i in range(3):
        print i
        time.sleep(4)

if __name__ == '__main__':
    p = multiprocessing.Process(target=worker, args=())
    p.daemon = True
    p.start()
    p.join()    # 阻塞主进程在子进程结束后再结束
    print 'end'

"""
0
1
2
end
"""
```



### 二. Lock

使用Lock来避免多进程共享资源的时候发生冲突

```python
lock = multiprocessing.Lock()
lock.acquire()    # 加锁
lock.release()    # 释放锁
```



### 三. Semaphore

信号量, 用来控制对共享资源的访问数量, 例如池的最大数量

```python
#!/usr/bin/env python
# coding:utf-8

import time
import multiprocessing

def worker(s, i):
    s.acquire()
    print multiprocessing.current_process().name + ' acquire'
    time.sleep(i)
    print multiprocessing.current_process().name + ' release'
    s.release()


if __name__ == '__main__':
    s = multiprocessing.Semaphore(2)
    for i in range(5):
        p = multiprocessing.Process(target=worker, args=(s, i*2))
        p.start()


```

`s = multiprocessing.Semaphore(2)`表示最多有两个连接数



### 四. Event

用来实现进程间同步通信, 具体看顶部链接



### 五. Queue

多进程间的数据传递要依靠队列, `Queue`是多进程安全的队列

常用方法

- `put: 插入数据到队列, 参数有`

  - blocked
  - timeout

  样例

  ```python
  put(blocked=True, timeout=3)    # 如果此时Queue已满, 则会阻塞3秒, 直到Qeueu有空位, 否则报Queue.Full异常
  put(blocked=False)    # 如果此时Qeueu已满, 会立即报Queue.Full异常
  ```

- `get: 从队列中拿出元素`

  - blocked
  - timeout


```python
#!/usr/bin/env python
# coding:utf-8

import time
import multiprocessing

def write(q, item):
    try:
        time.sleep(3)
        q.put(item, block=True, timeout=3)
    except:
        pass

def read(q):
    try:
        # 使用block=True会等到队列中有数据
        print q.get(block=True)
    except:
        pass


if __name__ == '__main__':
    q = multiprocessing.Queue()
    write = multiprocessing.Process(target=write, args=(q, 'hello world'))
    read = multiprocessing.Process(target=read, args=(q,))
    write.start()
    read.start()
    write.join()
    write.join()


"""
hello world
"""

```



###　六. Pipe管道

Pipe方法返回`(conn1, conn2)`表示管道两端

参数: `duplex`

- `True`: 两端均可收发
- `False`: conn1接收, conn2

方法:

- `send`
- `recv`: 如果没有消息接收, recv会一直阻塞, 直到管道被关闭, 会报`EOFError`

```python
#!/usr/bin/env python
# coding:utf-8

import multiprocessing
import time

def send_1(pipe):
    while True:
        for i in xrange(1000):
            pipe.send(i)
            print 'send: %d'%i
            time.sleep(1)

def recv_2(pipe):
    while True:
        print 'recv: ', pipe.recv()
        time.sleep(1)


if __name__ == '__main__':
    # 注意pipe有两个参数
    pipe = multiprocessing.Pipe()
    p1 = multiprocessing.Process(target=send_1, args=(pipe[0], ))
    p2 = multiprocessing.Process(target=recv_2, args=(pipe[1], ))
    p1.start()
    p2.start()
    p1.join()
    p2.join()


```





### 七. 进程池Pool

进程池可以控制进程的数量, 避免进程过多

参数:

- `apply_async(func[, args[, kwds[, callback]]]) `是__非阻塞__的
- `apply(func[, args[, kwds]])`是**阻塞**的
- `close`是关闭进程池pool, 使其不再接收新任务
- `terminate()`结束工作进程, 不再处理未完成的任务
- `join()`: 阻塞主进程, 等待所有子进程退出, 要在`close`或`terminate`之后运行

#### 对比`apply_async`和`apply`区别

- apply_async 非阻塞

  ```python
  #!/usr/bin/env python
  # coding:utf-8

  import multiprocessing
  import time


  def func(msg):
      print 'msg: ', msg
      time.sleep(3)
      print 'end'


  if __name__ == '__main__':
      pool = multiprocessing.Pool(processes=3)
      for i in range(10):
          msg = 'hello %d'%i
          pool.apply_async(func, (msg, ))

      print '-----&&&&&&&&-----'
      pool.close()    # 执行完close后不会有新的进程加入到pool
      pool.join()    # join必须在close后执行, join函数等待所有子进程结束
      print 'done!!!!!!!!!!!!!!'


  """
  -----&&&&&&&&-----
  msg:  hello 0
  msg:  hello 1
  msg:  hello 2
  end
  msg:  hello 3
  end
  msg:  hello 4
  end
  msg:  hello 5
  end
  end
  end
  msg:  hello 7
  msg:  hello 6
  msg:  hello 8
  end
  msg:  hello 9
  end
  end
  end
  done!!!!!!!!!!!!!!
  """
  ```

- apply 阻塞, **因为阻塞不能实现真正的并行**

  ```python
  #!/usr/bin/env python
  # coding:utf-8

  import multiprocessing
  import time


  def func(msg):
      print 'msg: ', msg
      time.sleep(3)
      print 'end'


  if __name__ == '__main__':
      pool = multiprocessing.Pool(processes=3)
      for i in range(10):
          msg = 'hello %d'%i
          pool.apply(func, (msg, ))

      print '-----&&&&&&&&-----'
      pool.close()    # 执行完close后不会有新的进程加入到pool
      pool.join()    # join必须在close后执行, join函数等待所有子进程结束
      print 'done!!!!!!!!!!!!!!'


  """
  msg:  hello 0
  end
  msg:  hello 1
  end
  msg:  hello 2
  end
  msg:  hello 3
  end
  msg:  hello 4
  end
  msg:  hello 5
  end
  msg:  hello 6
  end
  msg:  hello 7
  end
  msg:  hello 8
  end
  msg:  hello 9
  end
  -----&&&&&&&&-----
  done!!!!!!!!!!!!!!
  """
  ```

  ​
