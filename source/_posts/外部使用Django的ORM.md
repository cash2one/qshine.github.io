---
title: 外部使用Django的ORM
date: 2017-06-22 22:36:54
categories: Web后端
tags:
  - Django
---

写爬虫如果使用的是MySQL数据库的话那么使用Django的ORM来存库还是很方便的, 还可以自动建立起后端

#### 测试实例

目录结构

```shell
(github) ql@ubuntu:~/django_test$ tree
.
├── mysite
│   ├── manage.py
│   └── mysite
│       ├── __init__.py
│       ├── settings.py
│       ├── urls.py
│       └── wsgi.py
└── spider
    └── spider_test.py
```

想要在`spider_test`中使用django的ORM, 必须在开头加上如下内容

```python
import os, sys

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
# 加入pathon的搜索路径
sys.path.append(os.path.join(BASE_DIR, 'mysite'))
# 设置django的环境
os.environ['DJANGO_SETTINGS_MODULE'] = 'mysite.settings'
from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```

<!--more-->
