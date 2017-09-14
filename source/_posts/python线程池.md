---
title: python线程池
date: 2017-06-17 19:40:47
categories: Python
tags:
---

线程的生命周期: 创建, 就绪, 运行, 阻塞, 终止. 一个线程的运行时间分为三部分:

- 启动时间
- 运行时间
- 销毁时间

**线程池**适合处理突发性大量请求或者需要大量线程来完成的任务, 但是任务时间比较短的场景, 能有效避免创建过多线程导致系统负荷过大, 响应过慢的情况, 同时也能避免线程不断创建和销毁浪费的时间

如果仅仅需要创建少数线程, 可以直接全部创建, 但是任务量很大的时候线程池很有效


### 模拟实现
```python
#!/usr/bin/env python
#coding:utf-8

import threading
import Queue
import time


# 具体的工作
def do_job(i):
    time.sleep(1)
    print '任务: %s '%i


class WorkManager(object):
    """
    该类的主要作用就是把所有工作任务添加到队列中去,
    然后比如开启n个线程, 就可以保证只有n个线程在运行,
    直到队列中的所有任务被完成, 此时队列为空,
    注意使用 is_alive 设置守护线程
    """
    def __init__(self, work_num, thread_num):
        self.work_queue = Queue.Queue()
        self.threads = []
        self.__init__work_queue(work_num)
        self.__init__thread_pool(thread_num)

    # 初始化工作队列
    def __init__work_queue(self, work_num):
        for i in range(work_num):
            self.work_queue.put((do_job, i))

    # 初始化线程池
    def __init__thread_pool(self, thread_num):
        for i in range(thread_num):
            self.threads.append(Work(self.work_queue))

    # 守护线程, 保证线程结束
    def wait_allcomplete(self):
        for item in self.threads:
            if item.is_alive():
                item.join()


class Work(threading.Thread):
    """
    建立死循环, 从队列中取任务去完成
    能够保证同时有n个线程即n个任务在执行,
    执行完后循环在从队列中取任务
    注意使用 task_done 来通知队列该任务完成
    block=False 在队列为空时引发Empty异常
    """
    def __init__(self, work_queue):
        super(Work, self).__init__()
        self.work_queue = work_queue
        self.start()

    def run(self):
        # 死循环, 让线程不断从工作队列取出任务去执行
        while True:
            try:
                # 从队列依次去除任务执行
                # block=False 保证当队列为空时引发Empty异常, 默认为True
                do, args = self.work_queue.get(block=False)
                do(args)
                # 通知队列该任务完成, 队列中的任务减少
                self.work_queue.task_done()
            except:
                break


if __name__ == '__main__':
    start = time.time()
    work_manager = WorkManager(300, 50)
    # 开启守护线程
    work_manager.wait_allcomplete()
    end = time.time()
    print '花费时间: %s'%(end-start)
```

python的`multiprocessing`模块封装了线程池, 可以用以下命令直接使用, 比如代理ip的验证等场景
```
from multiprocessing.pool import ThreadPool
```

### 参考
- 伯乐在线多线程
- <<改善Pyhton程序的91条建议>>


