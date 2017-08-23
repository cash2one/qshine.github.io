---
title: Django使用Nginx和Gunicorn部署文档
date: 2017-05-25 08:58:46
categories: Web后端
tags:
  - Django
---

实时爬虫使用的Django做后端, 最近要放到生产服务器上了, 想起之前部署的时候一段痛苦的经历, 这次一定把每一个步骤记下来.

环境: `Ubuntu16.04`, `gunicorn`

收藏资料
- [Django安全配置settings详解](https://segmentfault.com/a/1190000003756582)
- [Nginx配置文件详解](https://segmentfault.com/a/1190000002797601)
- [Django+Nginx+Gunicorn部署文档](http://www.isaced.com/post-248.html)

### 一 . 新建Django项目

首先创建一个虚拟环境来隔离不同的版本.

新建一个目录`Deploy_test`, 在其中

```shell
mkdir Deploy_test
cd Deploy_test
```

进入`virtualenv`创建的虚拟环境来新建django项目

```shell
django-admin.py startproject mysite
```

此时我的项目目录

```shell
(xh_venv) ql@ubuntu:~/Deploy_test/mysite$ tree
.
├── manage.py
└── mysite
    ├── __init__.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py

1 directory, 5 files
(xh_venv) ql@ubuntu:~/Deploy_test/mysite$ 
```



进行测试

```shell
cd mysite
python manage.py makemigrations    
python manage.py migrate            
python manage.py createsuperuser 
python manage.py runserver 0.0.0.0:8000
```

打开浏览器进行测试, 访问`ip:8000/admin`, 此时正常的话会现实后台页面

__注意:__ 如果你提前设置了`settings.py`的`DEBUG=False`, 那么后台是没有样式的


<!--more-->

### 二. 测试gunicorn

还是在虚拟环境中安装gunicorn

```shell
pip install gunicorn
```

还是在`mysite`目录内

```shell
gunicorn mysite.wsgi:application -b 0.0.0.0:8000
```

没有在这里卡住, 此时再去测试后台页面是否能正常打开

__注意:__ 使用gunicorn后不管`settings.py`中`DEBUG`的值是多少, 都是没有样式的, 因为不知道静态文件在哪, 如果发现有样式, 别慌, 把浏览器缓存干掉就没了...



### 三. 使用Nginx

- 修改`settings.py`

  ```python
  DEBUG = False
  ALLOWED_HOSTS = [('*')]   # 测试方便使用 *
  ......
  # 在最后加上这一句
  STATIC_ROOT = os.path.join(BASE_DIR, 'static')
  ```

​      `ALLOWED_HOSTS`是为了限定请求中的host值,以防止黑客构造包来发送请求.只有在列表中的host才能访问.强烈建议不要使用`*`通配符去配置,另外当`DEBUG`设置为`False`的时候必须配置这个配置.否则会抛出异常

- 收集`所有的静态文件到该目录中`

  ```
  python manage.py collectstatic
  ```

  然后就能看到项目中多了一个static文件夹

- 安装Nginx

  ```
  sudo apt update
  sudo apt upgrade
  sudo apt install nginx
  ```


- 停止Nginx

  ```
  sudo service nginx stop
  ```

- 备份Nginx默认配置文件

  Nginx的默认配置文件路径: `/etc/nginx/sites-available/default`

  ```
  cd /etc/nginx/sites-available/
  sudo cp default default.bak
  ```

- 修改配置文件`sudo vim default`为如下内容

  ```
  server {
          listen   80;

          server_name _;
          access_log  /var/log/nginx/isaced.log;

          location / {
                  proxy_pass http://127.0.0.1:8000;
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          }

          location /static/ {
                  root /home/ql/Deploy_test/mysite;
          }
  }
  ```

  - listen: 监听80端口
  - server_name: 配置基于名称的虚拟主机, 可以基于正则表达式, 暂时不管
  - access_log
  - location /
    - proxy_pass: 请求转向backend定义的服务器列表, 即反向代理
    - 剩余3个暂且这样设置
  - location /static/
    - root: 后面必须是mysite的根路径, Nginx会去这里面找到收集的静态文件

  修改完后保存退出

- 测试

  - 启动gunicorn, 进入该项目的路径

    ```
    nohup gunicorn mysite.wsgi:application -b 127.0.0.1:8000 -w 4 &
    ```

    `-w 4`表示启动4个进程, 默认为1

  - 启动Nginx

    ```
    sudo service nginx start
    ```

    ​

### 注意

Nginx那里坑特别多, 再加上对那块不熟悉, 导致可能为了搜错误来一趟谷歌旅游

关于这块的一些建议:

- 做好配置文件的备份

- 千万不要去kill掉nginx的进程号, 使用`sudo service nginx stop`来停止

- 使用`sudo nginx -t`进行测试

  ```
  ql@ubuntu:/etc/nginx/sites-available$ sudo nginx -t
  nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
  nginx: configuration file /etc/nginx/nginx.conf test is successful
  ```

  成功的话问题应该就不大了


