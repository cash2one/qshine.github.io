---
title: 'pyspider[六]调度器_check_task_done函数'
date: 2017-04-10 06:19:41
categories: 爬虫
tags:
  - pyspider
  - python
---

从`status_queue`中得到的数据格式为
```python
task = {
    'taskid': task['taskid'],
    'project': task['project'],
    'url': task.get('url'),
    'track': {
        'fetch': {
            'ok': response.isok(),
            'redirect_url': response.url if response.url != response.orig_url else None,
            'time': response.time,
            'error': response.error,
            'status_code': response.status_code,
            'encoding': getattr(response, '_encoding', None),
            'headers': track_headers,
            'content': response.text[:500] if ret.exception else None,
        },
        'process': {
            'ok': not ret.exception,
            'time': process_time,
            'follows': len(ret.follows),
            'result': (
                None if ret.result is None
                else utils.text(ret.result)[:self.RESULT_RESULT_LIMIT]
            ),
            'logs': ret.logstr()[-self.RESULT_LOGS_LIMIT:],
            'exception': ret.exception,
        },
        'save': ret.save,
    },
}
```

<!--more-->

## _check_task_done函数
```python
def _check_task_done(self):
    cnt = 0
    try:
        while True:
            task = self.status_queue.get_nowait()    
            # check _on_get_info result here
            if task.get('taskid') == '_on_get_info' and 'project' in task and 'track' in task:
                if task['project'] not in self.projects:
                    continue
                project = self.projects[task['project']]
                project.on_get_info(task['track'].get('save') or {})
                # 输出日志信息
                logger.info(
                    '%s on_get_info %r', task['project'], task['track'].get('save', {})
                )
                continue
            elif not self.task_verify(task):
                continue
            self.on_task_status(task)
            cnt += 1
    except Queue.Empty:
        pass
    return cnt
```
该函数主要是检查已经完成的任务, 即`processor`解析完成后压入`status_queue`中的任务
`self.status_queue.get_nowait()`是从`status_queue`中不断拿出`task`
第一次的`taskid`是`_on_get_info`, 之后会检查`task_verify()`函数

### task_verify(task)
```python
def task_verify(self, task):
    '''
    return False if any of 'taskid', 'project', 'url' is not in task dict
                    or project is not in task_queue
    '''
    for each in ('taskid', 'project', 'url', ):
        if each not in task or not task[each]:
            logger.error('%s not in task: %.200r', each, task)
            return False
    if task['project'] not in self.projects:
        logger.error('unknown project: %s', task['project'])
        return False

    project = self.projects[task['project']]
    if not project.active:
        logger.error('project %s not started, please set status to RUNNING or DEBUG',
                     task['project'])
        return False
    return True
```
该函数主要是对task的合法性进行判断, 比如task是否包含必须的字段, 所述的项目是否在self.projects中, 项目是否为激活状态, 如果不符合返回False, 直接忽略该task

### on_task_status(task)
该函数主要检查task是否被解析器(processor)解析正确, 如果为True, 则调用`on_task_done(task)`函数, 否则调用`on_task_failed(task)`函数

### on_task_done(task)
```python
def on_task_done(self, task):
    '''Called when a task is done and success, called by `on_task_status`'''
    task['status'] = self.taskdb.SUCCESS
    task['lastcrawltime'] = time.time()

    if 'schedule' in task:
        if task['schedule'].get('auto_recrawl') and 'age' in task['schedule']:
            task['status'] = self.taskdb.ACTIVE     #维持active状态
            # 加入下次执行的时间
            next_exetime = task['schedule'].get('age')
            task['schedule']['exetime'] = time.time() + next_exetime
            self.put_task(task)
        else:
            del task['schedule']
    self.update_task(task)
......
```
该函数检查task中是否有调度相关的参数, 如`auto_recrawl`和`age`, , 有的话更改`exetime`的值, 然后执行`put_task(task)`把该任务放入该项目对应的`TaskQueue`中, 最后从`taskdb`中更新该任务状态

