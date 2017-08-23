---
title: 'pyspider[九]任务队列TaskQueue'
date: 2017-05-01 16:28:19
categories: 爬虫
tags:
  - pyspider
  - python
---

在`task_queue.py`中主要有`TaskQueue`和`PriorityQueue`等, 这几个类主要和调度器的调度程序有关

## TaskQueue
每个项目都对应了一个实例化的`TaskQueue`, 初始化`TaskQueue`的工作在scheduler的`update_projects`中的`Project`类中:
```python
class Project(object):
    def __init__(self, scheduler, project_info):
        ......
        self.active_tasks = deque(maxlen=scheduler.ACTIVE_TASKS)
        self.task_queue = TaskQueue()
        ......

    def update(self, project_info):
        ......
        # 如果该项目为active状态, 则马上设置 rate/burst
        if self.active:
            self.task_queue.rate = project_info['rate']
            self.task_queue.burst = project_info['burst']
        else:
            self.task_queue.rate = 0
            self.task_queue.burst = 0
```

<!--more-->

接着看实例化的`TaskQueue`
```python
class TaskQueue(object):
    '''
    task queue for scheduler, have a priority queue and a time queue for delayed tasks
    '''
    processing_timeout = 10 * 60

    def __init__(self, rate=0, burst=0):
        self.mutex = threading.RLock()
        self.priority_queue = PriorityTaskQueue()       # 优先级队列
        self.time_queue = PriorityTaskQueue()           # 时间队列
        self.processing = PriorityTaskQueue()           # 调度队列
        # 这里实例化的另一个Bucket类, 每个项目对应一个Bucket类
        self.bucket = Bucket(rate=rate, burst=burst)    

    @property
    def rate(self):
        return self.bucket.rate

    @rate.setter
    def rate(self, value):
        self.bucket.rate = value

    @property
    def burst(self):
        return self.bucket.burst

    @burst.setter
    def burst(self, value):
        self.bucket.burst = value

    def check_update(self):
        '''
        Check time queue and processing queue
        put tasks to priority queue when execute time arrived or process timeout
        '''
        self._check_time_queue()
        self._check_processing()

    def _check_time_queue(self):
        now = time.time()
        self.mutex.acquire()    # 加锁
        while self.time_queue.qsize() and self.time_queue.top and self.time_queue.top.exetime < now:
            task = self.time_queue.get_nowait()
            task.exetime = 0
            # 压入优先级队列
            self.priority_queue.put(task)
        self.mutex.release()

    def _check_processing(self):
        now = time.time()
        self.mutex.acquire()
        while self.processing.qsize() and self.processing.top and self.processing.top.exetime < now:
            task = self.processing.get_nowait()
            if task.taskid is None:
                continue
            task.exetime = 0
            self.priority_queue.put(task)
            logger.info("processing: retry %s", task.taskid)
        self.mutex.release()

    def put(self, taskid, priority=0, exetime=0):
        now = time.time()
        task = InQueueTask(taskid, priority, exetime)
        self.mutex.acquire()

        # 一次放入对应的队列, priority_queue和time_queue
        if taskid in self.priority_queue:
            self.priority_queue.put(task)
        elif taskid in self.time_queue:
            self.time_queue.put(task)
        elif taskid in self.processing and self.processing[taskid].taskid:
            # force update a processing task is not allowed as there are so many
            # problems may happen
            pass
        else:
            # 如果exetime大于now则放入 time_queue, 否则放入 priority_queue
            if exetime and exetime > now:
                self.time_queue.put(task)
            else:
                self.priority_queue.put(task)
        self.mutex.release()

    def get(self):
        '''Get a task from queue when bucket available'''
        if self.bucket.get() < 1:
            return None
        now = time.time()
        self.mutex.acquire()
        try:
            task = self.priority_queue.get_nowait()    # 从priority_queue中拿出任务
            self.bucket.desc()
        except Queue.Empty:
            self.mutex.release()
            return None
        task.exetime = now + self.processing_timeout
        self.processing.put(task)    # 放入processing队列
        self.mutex.release()
        # 返回taskid
        return task.taskid

    def done(self, taskid):
        '''Mark task done'''
        if taskid in self.processing:
            self.mutex.acquire()
            if taskid in self.processing:
                del self.processing[taskid]
            self.mutex.release()
            return True
        return False

    def delete(self, taskid):
        if taskid not in self:
            return False
        if taskid in self.priority_queue:
            self.mutex.acquire()
            del self.priority_queue[taskid]
            self.mutex.release()
        elif taskid in self.time_queue:
            self.mutex.acquire()
            del self.time_queue[taskid]
            self.mutex.release()
        elif taskid in self.processing:
            self.done(taskid)
        return True

    def size(self):
        return self.priority_queue.qsize() + self.time_queue.qsize() + self.processing.qsize()

    def is_processing(self, taskid):
        '''
        return True if taskid is in processing
        '''
        return taskid in self.processing and self.processing[taskid].taskid

    def __len__(self):
        return self.size()

    def __contains__(self, taskid):
        if taskid in self.priority_queue or taskid in self.time_queue:
            return True
        if taskid in self.processing and self.processing[taskid].taskid:
            return True
        return False

```
- `put`方法
在scheduler中的`self._update_projects()`和`self._check_request()` 函数中会调用该put方法, 来向TaskQueue中压入任务
- `get`方法
在scheduler中的`self._check_select()`中会调用get方法, 拿出`taskid`, 然后根据`taskid`从`taskdb`中拿出`task`给`fetcher`



## PriorityQueue
该类主要继承了python内置的`Queue`, 并使用了`heapq`模块
```python
class PriorityTaskQueue(Queue.Queue):
    '''
    Same taskid items will been merged
    '''

    def _init(self, maxsize):
        self.queue = []
        self.queue_dict = dict()    # 键为taskid

    def _qsize(self, len=len):
        return len(self.queue_dict)

    def _put(self, item, heappush=heapq.heappush):
        # 如果taskid已经存在, 则去比较优先级大小
        if item.taskid in self.queue_dict:
            task = self.queue_dict[item.taskid]
            changed = False
            # 替换为较大的优先级
            if item.priority > task.priority:
                task.priority = item.priority
                changed = True
            # 替换为较小的时间
            if item.exetime < task.exetime:
                task.exetime = item.exetime
                changed = True
            # 如果改变了, 初始化堆
            if changed:
                self._resort()
        # 如果不存在, 直接压入队里
        else:
            heappush(self.queue, item)
            self.queue_dict[item.taskid] = item

    def _get(self, heappop=heapq.heappop):
        # 从堆队列中不断拿出任务
        while self.queue:
            item = heappop(self.queue)
            if item.taskid is None:
                continue
            self.queue_dict.pop(item.taskid, None)
            return item
        return None

    @property
    def top(self):
        while self.queue and self.queue[0].taskid is None:
            heapq.heappop(self.queue)
        if self.queue:
            return self.queue[0]
        return None

    def _resort(self):
        # 初始化队列
        heapq.heapify(self.queue)

    def __contains__(self, taskid):
        return taskid in self.queue_dict

    def __getitem__(self, taskid):
        return self.queue_dict[taskid]

    def __setitem__(self, taskid, item):
        assert item.taskid == taskid
        self.put(item)

    def __delitem__(self, taskid):
        self.queue_dict.pop(taskid).taskid = None
```


## 小结
下面计划来简单梳理一下`scheduler`的运行流程, 最近的工作太忙了, 断断续续的写, 自己也一直在遗忘
### `_update_projects()`函数
- scheduler启动后先拿出`projectdb`中需要更新的项目信息
- 初始化`Project`类, 在初始化过程中也实例化了该项目对应的`TaskQueue`, 在TaskQueue中实例化了`Bucket`类来设置`rate`和`burst`
- 该函数会输出类似的日志:
`[I 170415 17:40:57 scheduler:126] project baidu updated, status:RUNNING, paused:False, 0 tasks`


### `_check_task_done`函数
- 主要从`status_queue`中拿出已经完成的任务`task`, 并检查`task`的格式是否正确, 解析是否正确, 如果解析正确, 那么查看有没有`auto_recrawl`和`age`这两个字段, 有的话更改`exetime`的值后重新放入`TaskQueue`中
- 该函数会输出类似的日志:
`[I 170415 17:41:13 scheduler:906] task done baidu:on_start data:,on_start`


### `_check_request()`函数
- 该函数会尝试从`newtask_queue`拿出task, 然后根据`taskid`去检查`taskdb`中是否存在, 存在的话说明是老任务, 否则为新任务, 并根据情况去执行
- 新任务的话先把`task`插入`taskdb`中, 紧接着放入`TaskQueue`中
- 该函数会输出类似的日志:
`[I 170415 17:41:12 scheduler:808] new task baidu:on_start data:,on_start`

### `_check_select()`函数
- 对所有项目的`TaskQueue`进行遍历, 每个项目拿出一定数量的`taskid`, 根据这些`taskid`从`taskdb`拿出任务交给`scheduler2fetcher`队列
- 该函数谁输出类似日志:
`[I 170415 17:41:12 scheduler:965] select baidu:on_start data:,on_start`

