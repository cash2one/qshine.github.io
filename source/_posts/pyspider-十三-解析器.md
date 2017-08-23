---
title: 'pyspider[十三]解析器'
date: 2017-06-4 19:39:04
categories: 爬虫
tags:
  - pyspider
---

和其它组件一样, 都是在`run`函数中开启循环来从`fetcher2processor`队列中拿出抓取完成的`task`和`response`, 然后交由`on_task`函数来处理

```
task, response = self.inqueue.get(timeout=1)
self.on_task(task, response)
```

<!--more-->

### on_task函数

```python
    def on_task(self, task, response):
        '''Deal one task'''
        start_time = time.time()
        # 对result进行封装, response是一个Response实例, 有在浏览器脚本中使用的各种方法
        response = rebuild_response(response)

        try:
            assert 'taskid' in task, 'need taskid in task'
            project = task['project']
            updatetime = task.get('project_updatetime', None)
            md5sum = task.get('project_md5sum', None)
            
            """
            project_data格式:
                {
                    'loader': loader,       # 实例化的ProjectLoader()类
                    'module': module,       # 对脚本进行了编译为一个模块
                    'class': _class,        # 对应的脚本中的子类
                    'instance': instance,   # 脚本中子类的实例化
                    'exception': None,
                    'exception_log': '',
                    'info': project,        # 该项目的名字
                    'load_time': time.time(),
                }
            """
            project_data = self.project_manager.get(project, updatetime, md5sum)
            assert project_data, "no such project!"
            # 正常情况exception为None
            if project_data.get('exception'):
                ret = ProcessorResult(logs=(project_data.get('exception_log'), ),
                                      exception=project_data['exception'])
            else:
                # 这里开始运行脚本, run_task是父类BaseHandler中的方法
                # 返回结果是ProcessorResult的实例
                ret = project_data['instance'].run_task(
                    project_data['module'], task, response)
        except Exception as e:
            logstr = traceback.format_exc()
            ret = ProcessorResult(logs=(logstr, ), exception=e)
        process_time = time.time() - start_time

        if not ret.extinfo.get('not_send_status', False):
            if ret.exception:
                track_headers = dict(response.headers)
            else:
                track_headers = {}
                for name in ('etag', 'last-modified'):
                    if name not in response.headers:
                        continue
                    track_headers[name] = response.headers[name]

            # 封装好压入 status_queue 的内容
            status_pack = {
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
            if 'schedule' in task:
                status_pack['schedule'] = task['schedule']

            # FIXME: unicode_obj should used in scheduler before store to database
            # it's used here for performance.

            # 压入status_queue队列, 至此解析工作完成
            self.status_queue.put(utils.unicode_obj(status_pack))

        # FIXME: unicode_obj should used in scheduler before store to database
        # it's used here for performance.
        # 如果有新的任务把新的任务压入 newtask_queue 中
        if ret.follows:
            for each in (ret.follows[x:x + 1000] for x in range(0, len(ret.follows), 1000)):
                self.newtask_queue.put([utils.unicode_obj(newtask) for newtask in each])

        for project, msg, url in ret.messages:
            try:
                self.on_task({
                    'taskid': utils.md5string(url),
                    'project': project,
                    'url': url,
                    'process': {
                        'callback': '_on_message',
                    }
                }, {
                    'status_code': 200,
                    'url': url,
                    'save': (task['project'], msg),
                })
            except Exception as e:
                logger.exception('Sending message error.')
                continue

        if ret.exception:
            logger_func = logger.error
        else:
            logger_func = logger.info

        # processor输出的日志
        logger_func('process %s:%s %s -> [%d] len:%d -> result:%.10r fol:%d msg:%d err:%r' % (
            task['project'], task['taskid'],
            task.get('url'), response.status_code, len(response.content),
            ret.result, len(ret.follows), len(ret.messages), ret.exception))
        return True
```



解析器这块感觉还是蛮复杂的, 很多小的细节没有看懂, 有前辈看到的话望能指出错误地方








