---
title: 'pyspider[十]令牌桶算法'
date: 2017-05-07 06:30:48
categories: 数据结构和算法
tags:
  - pyspider
---

算法一直是我的弱项, 特别是最近翻看了一位大牛的博客, 觉得自己比之前更渣了, 反正自己这么渣写的东西只当做备份, 日后自己看, 所以还是尽量写完吧 -- 最近的自我吐槽

pyspider使用了`令牌桶算法`来控制流量, 刚看到这个的时候是懵逼的, 在设置中一直和`漏桶算法`混淆, 下面是我学习的时候看的一些文章:
- [漏桶算法(Leaky Bucket)和令牌桶算法(Token Bucket)](http://www.javaranger.com/archives/1769)
- [令牌桶算法](http://ju.outofmemory.cn/entry/102231)
- [github上的实现](https://github.com/titan-web/rate-limit)

pyspider在`scheduler.py`中初始化了任务队列`TaskQueue`, 在`TaskQueue`中又使用了`令牌桶算法`

首先`TaskQueue`主要是用来存储任务队列, 供调度器拿出马上要执行的`task`放到`scheduler2fetcher`队列中, 所以在`scheduler.py`中直接作用的函数是`self._check_select()`

<!--more-->

### `_check_select`
该函数的主要作用就是去遍历`self.projects`中的所有的项目, 找到`status`为`running`的项目, 然后拿到该项目对应的`TaskQueue`, 并在`while`循环中去执行`task_queue.get()`方法, 尝试拿出本项目一定数量的`taskid`, 如果为`None`则跳出`while`循环
```python
for project in itervalues(self.projects):
    ......
    # task queue
    task_queue = project.task_queue

    # 检查time_queue和processing_queue
    task_queue.check_update()

    project_cnt = 0

    # check send_buffer here. when not empty, out_queue may blocked. Not sending tasks
    while cnt < limit and project_cnt < limit / 10:
        # 从TaskQueue中拿到task的taskid
        taskid = task_queue.get()
        if not taskid:
            break

        # 存入taskids
        taskids.append((project.name, taskid))
```

### `task_queue`中的`get`方法
在上述代码的`while`循环中调用的就是该`get`方法. 先去根据设置的`rate/burst`查看该段时间能不能从`TaskQueue`中拿出任务, 不能的话返回None跳过该项目, 能的话则拿出一个`taskid`, 执行`desc()`方法, 放入`processing`队列最后返回
```python
def get(self):
    '''Get a task from queue when bucket available'''
    if self.bucket.get() < 1:
        return None
    now = time.time()
    self.mutex.acquire()
    try:
        task = self.priority_queue.get_nowait()
        self.bucket.desc()
    except Queue.Empty:
        self.mutex.release()
        return None
    task.exetime = now + self.processing_timeout
    self.processing.put(task)
    self.mutex.release()
    return task.taskid
```
pyspider的并发控制是`令牌桶算法`控制的, 作用点就是这里

