---
title: 'pyspider[八]调度器_check_select函数'
date: 2017-04-25 06:26:25
categories: 爬虫
tags:
  - pyspider
  - python
---

该函数的主要工作是从`TaskQueue`中挑出一个`task`放入`scheduler2fetcher`队列, 给抓取器`fetcher`去执行抓取

```python
def _check_select(self):
    '''Select task to fetch & process'''
    while self._send_buffer:
        # 从双端队列中弹出一个任务
        _task = self._send_buffer.pop()
        try:
            # use force=False here to prevent automatic send_buffer append and get exception
            self.send_task(_task, False)
        except Queue.Full:
            self._send_buffer.append(_task)
            break
    ......
```
首先检查双端队列中有没有任务, 优先从`_send_buffer`中拿出task, 交给`send_task`放入输出队列`scheduler2fetcher`中

```python
taskids = []
cnt = 0
cnt_dict = dict()
limit = self.LOOP_LIMIT     # default 1000

# self.projects存的是所有项目的 Project 实例
for project in itervalues(self.projects):
    # 只拿出激活的项目
    if not project.active:
        continue
    # only check project pause when select new tasks, cronjob and new request still working
    if project.paused:
        continue
    if project.waiting_get_info:
        continue
    if cnt >= limit:
        break

    # 每个项目对应的 TaskQueue
    task_queue = project.task_queue

    # 对time_queue和processing_queue进行检查和初始化
    task_queue.check_update()

    project_cnt = 0

    # check send_buffer here. when not empty, out_queue may blocked. Not sending tasks
    # 使用get方法从TashQueue中拿出taskid, 最多拿出100条
    while cnt < limit and project_cnt < limit / 10:
        # 从TaskQueue中拿到taskid
        taskid = task_queue.get()
        if not taskid:
            break

        # 把taskid和对应的项目名称放入列表
        taskids.append((project.name, taskid))
        if taskid != 'on_finished':
            project_cnt += 1
        cnt += 1
        ......

# 对taskids进行循环
for project, taskid in taskids:
    self._load_put_task(project, taskid)

```
对`self.projects`进行循环, 依次检查每个项目对应的`TaskQueue`, 然后从每个项目的`TaskQueue`内拿出一定数量的`taskid`, 最后所有项目的`taskid`汇总到一块, 根据这些`taskid`, 从`taskdb`中拿出具体的`task`放入`scheduler2fetcher`队列交给fetcher去抓取

### _load_put_task(project, taskid)
```python
def _load_put_task(self, project, taskid):
    try:
        # 根据taskid从taskdb拿出整个任务信息
        task = self.taskdb.get_task(project, taskid, fields=self.request_task_fields)
    except ValueError:
        logger.error('bad task pack %s:%s', project, taskid)
        return
    if not task:
        return
    # 再次把task发入输出队列 scheduler --> fetcher
    task = self.on_select_task(task)
```

### on_select_task(task)
```python
def on_select_task(self, task):
    """
    task的结构:
    {
        'taskid': '_on_get_info',
        'project': project.name,
        'url': 'data:,_on_get_info',
        'status': self.taskdb.SUCCESS,
        'fetch': {
            'save': self.get_info_attributes,
        },
        'process': {
            'callback': '_on_get_info',
        },
    }

    """
    """
    Called when a task is selected to fetch & process
    """
    # inject informations about project
    logger.info('select %(project)s:%(taskid)s %(url)s', task)

    project_info = self.projects.get(task['project'])
    assert project_info, 'no such project'
    # 对task进行封装
    task['type'] = self.TASK_PACK
    task['group'] = project_info.group
    task['project_md5sum'] = project_info.md5sum
    task['project_updatetime'] = project_info.updatetime

    # lazy join project.crawl_config
    if getattr(project_info, 'crawl_config', None):
        task = BaseHandler.task_join_crawl_config(task, project_info.crawl_config)

    # 封装后的task格式
    """
    {
        'status': 2,
        'project_updatetime': 1489221076.322148,
        'group': None,
        'url': 'data:,_on_get_info',
        'project_md5sum': '5b8536c234e09275f7a2c60c9e03b777',
        'project': u'baidu',
        'process': {
            'callback': '_on_get_info'
        },
        'taskid': '_on_get_info',
        'type': 1,
        'fetch': {
            'save': ['min_tick', 'retry_delay', 'crawl_config']
        }
    }
    """
    project_info.active_tasks.appendleft((time.time(), task))

    # 压入了out_queue(scheduler --> fetcher)队列
    self.send_task(task)
    return task
```
该函数的主要任务是对从taskdb中拿出来的任务进行再一次`封装`, 封装好一定的格式后才真正的交给`输出队列(scheduler --> fetcher)`

