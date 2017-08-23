---
title: 'pyspider[七]调度器_check_request函数'
date: 2017-04-20 24:22:27
categories: 爬虫
tags:
  - pyspider
  - python
---

该函数主要是从`newtask_queue`中不断拿出任务, 默认最多可拿1000个task
```python
# 默认 LOOP_LIMIT 是 1000
while len(tasks) < self.LOOP_LIMIT:
    try:
        # 从newtask_queue中拿任务, 采用非阻塞方式, 为空后会报错
        task = self.newtask_queue.get_nowait()
    except Queue.Empty:
        break
```
然后对拿出来的所有task, 执行`on_request`函数
```python
for task in itervalues(tasks):
    self.on_request(task)
```

### on_request(task)
```python
def on_request(self, task):
    if self.INQUEUE_LIMIT and len(self.projects[task['project']].task_queue) >= self.INQUEUE_LIMIT:
        logger.debug('overflow task %(project)s:%(taskid)s %(url)s', task)
        return

    # 从taskdb中看是否之前已经存在
    oldtask = self.taskdb.get_task(task['project'], task['taskid'],
                                   fields=self.merge_task_fields)

    if oldtask:
        return self.on_old_request(task, oldtask)
    else:
        return self.on_new_request(task)
```
根据taskid从`taskdb`中拿任务, 如果没有拿到说明是新任务, 执行`on_new_request`, 否则执行`on_old_request`

### on_new_request(task)
```python
def on_new_request(self, task):
    '''Called when a new request is arrived'''
    task['status'] = self.taskdb.ACTIVE
    # 针对新任务, 首先插入taskdb
    self.insert_task(task)

    # 之后把task压入task_queue
    self.put_task(task)

    project = task['project']
    ......
    logger.info('new task %(project)s:%(taskid)s %(url)s', task)
    return task
```

### on_old_request(task, old_task)
```python
def on_old_request(self, task, old_task):
    '''Called when a crawled task is arrived'''
    now = time.time()
    # 拿到新的调度参数
    _schedule = task.get('schedule', self.default_schedule)
    # 老的调度参数
    old_schedule = old_task.get('schedule', {})

    if _schedule.get('force_update') and self.projects[task['project']].task_queue.is_processing(task['taskid']):
        # when a task is in processing, the modify may conflict with the running task.
        # postpone the modify after task finished.
        logger.info('postpone modify task %(project)s:%(taskid)s %(url)s', task)
        self._postpone_request.append(task)
        return

    restart = False
    schedule_age = _schedule.get('age', self.default_schedule['age'])

    # 根据版本号(itag)和调度时间(scheduler_age)和强制更新(force_update)来设置restart=True
    if _schedule.get('itag') and _schedule['itag'] != old_schedule.get('itag'):
        restart = True
    elif schedule_age >= 0 and schedule_age + (old_task.get('lastcrawltime', 0) or 0) < now:
        restart = True
    elif _schedule.get('force_update'):
        restart = True

    if not restart:
        logger.debug('ignore newtask %(project)s:%(taskid)s %(url)s', task)
        return

    if _schedule.get('cancel'):
        logger.info('cancel task %(project)s:%(taskid)s %(url)s', task)
        task['status'] = self.taskdb.BAD
        self.update_task(task)
        self.projects[task['project']].task_queue.delete(task['taskid'])
        return task

    task['status'] = self.taskdb.ACTIVE

    # 从数据库taskdb中更新和放入TaskQueue中
    self.update_task(task)
    self.put_task(task)

    project = task['project']
    if old_task['status'] != self.taskdb.ACTIVE:
        self._cnt['5m'].event((project, 'pending'), +1)
        self._cnt['1h'].event((project, 'pending'), +1)
        self._cnt['1d'].event((project, 'pending'), +1)
    if old_task['status'] == self.taskdb.SUCCESS:
        self._cnt['all'].event((project, 'success'), -1).event((project, 'pending'), +1)
    elif old_task['status'] == self.taskdb.FAILED:
        self._cnt['all'].event((project, 'failed'), -1).event((project, 'pending'), +1)
    logger.info('restart task %(project)s:%(taskid)s %(url)s', task)
    return task
```

<!--more-->
