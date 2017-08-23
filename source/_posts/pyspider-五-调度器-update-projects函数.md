---
title: 'pyspider[五]调度器_update_projects函数'
date: 2017-03-29 16:18:28
categories: 爬虫
tags:
  - pyspider
  - python
---

## `_update_projects()`函数执行流程

- 首先更新项目信息，未到更新时间的不更新, 其中更新时间间隔为`5*60s`
```python
if (
        not self._force_update_project
        and self._last_update_project + self.UPDATE_PROJECT_INTERVAL > now
):
    return
```
然后从projectdb中拿出需要更新的项目, 执行`self._update_project(project)`
其中`project`的格式为
```python
#project的格式为:
{
    u'status': 'RUNNING',
    u'updatetime': 1490062448.005,
    u'group': None,
    u'name': 'baidu',
    u'script': '......',
    u'burst': 3.0,
    u'comments': None,
    u'rate': 1.0
}
```

<!--more-->

- 实例化该项目的`Project`类
```python
self.projects[project['name']] = Project(self, project)
```
在实例化的时候会创建一个`双端队列`和实例化一个`TaskQueue`
```python
self.active_tasks = deque(maxlen=scheduler.ACTIVE_TASKS)    # ACTIVE_TASKS默认为100
# 实例化任务队列, TaskQueue的作用是放调度器扔进来的任务
self.task_queue = TaskQueue()
```
之后去执行`update`方法, 并输出日志:
```
[I 170330 20:54:14 scheduler:126] project baidu updated, status:RUNNING, paused:False, 0 tasks
```
比对脚本的md5值, 来更新`self._send_on_get_info`和`self.waiting_get_info`参数的值(True/False)
调用`self.on_select_task()`函数, 参数是一个`task`
```python
self.on_select_task(
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
)
```

- `def on_select_task(self, task)`函数
```python
def on_select_task(self, task):
    """
    传过来的task的结构:
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
    # 在这里输出日志信息, 如:select baidu:_on_get_info data:,_on_get_info
    logger.info('select %(project)s:%(taskid)s %(url)s', task)

    project_info = self.projects.get(task['project'])
    assert project_info, 'no such project'

    # 对task进行一次封装
    task['type'] = self.TASK_PACK
    task['group'] = project_info.group
    task['project_md5sum'] = project_info.md5sum
    task['project_updatetime'] = project_info.updatetime

    # lazy join project.crawl_config
    if getattr(project_info, 'crawl_config', None):
        task = BaseHandler.task_join_crawl_config(task, project_info.crawl_config)

    # 封装好后的task格式
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
    # 把task和当前时间戳放到 Project 的双端队列中
    project_info.active_tasks.appendleft((time.time(), task))

    # 调用 send_task 函数来向fetcher发送任务
    self.send_task(task)
    return task
```

- `send_task(self, task, force=True)` 函数
```python
def send_task(self, task, force=True):
    '''
    发送任务给 fetcher
    send_buffer 当做 out_queue 满后的缓冲区
    out queue may have size limit to prevent block, a send_buffer is used
    '''
    try:
        # 向 scheduler --> fetcher 中压入了 task
        self.out_queue.put_nowait(task)
    except Queue.Full:
        if force:
            # 放入 _send_buffer 中
            self._send_buffer.appendleft(task)
        else:
            raise
```

- 接着再回到`def _update_project(self, project)`函数
执行完`on_select_task(task)`函数后, 接着执行
```python
if project.active:
    if not project.task_loaded:
        self._load_tasks(project)   # 从数据库加载task
        project.task_loaded = True
else:
    if project.task_loaded:
        project.task_queue = TaskQueue()
        project.task_loaded = False

    if project not in self._cnt['all']:
        self._update_project_cnt(project.name)
```
可以看到执行了`_load_tasks(project)`函数

- `def _load_tasks(self, project)`函数
```python
def _load_tasks(self, project):
    """
    从taskdb中加载
    """
    # 拿出该项目在 Project 中实例化的 TaskQueue
    task_queue = project.task_queue

    for task in self.taskdb.load_tasks(
            self.taskdb.ACTIVE, project.name, self.scheduler_task_fields
    ):
        taskid = task['taskid']
        _schedule = task.get('schedule', self.default_schedule)
        priority = _schedule.get('priority', self.default_schedule['priority'])
        exetime = _schedule.get('exetime', self.default_schedule['exetime'])

        # 把task的id, priority和exetime等放入队列中
        task_queue.put(taskid, priority, exetime)


    project.task_loaded = True
    logger.debug('project: %s loaded %d tasks.', project.name, len(task_queue))

    if project not in self._cnt['all']:
        self._update_project_cnt(project)
    self._cnt['all'].value((project.name, 'pending'), len(project.task_queue))
```
分别拿出`taskid`, `schedule`, `priority`和`exetime`等相关信息, 全部放入实例化的`TaskQueue`中

