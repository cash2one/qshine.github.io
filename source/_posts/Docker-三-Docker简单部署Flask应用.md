---
title: 'Docker(三)Docker简单部署Flask应用'
date: 2017-08-06 23:17:29
categories: Docker
tags:
---

仅仅是记录了一下流程, 加深对`Docker`的理解和`Dockerfile`的编写


### 创建文件夹和应用
```
ql@ubuntu:~$ mkdir flask-web
ql@ubuntu:~$ cd flask-web/
```
编写Flask应用`app.py`
```
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    return 'hello world'


if __name__ == '__main__':
    app.run(
        debug=True,
        host='0.0.0.0'
    )
```
编写`requirements.txt`
```
Flask>=0.10
```

<!--more-->

### 编写Dockerfile
#### 使用`python2.7作为镜像`
```

FROM python:2.7

MAINTAINER test test@gmail.com

# 复制当前目录内容到容器
COPY . /app

# 指定容器内的工作目录
WORKDIR /app

# 安装应用所需环境
RUN pip install -r requirements.txt

ENTRYPOINT ["python"]
CMD ["app.py"]
```

#### 使用ubuntu作为镜像
```
FROM ubuntu:16.04

MAINTAINER test test@gmail.com

# 安装python和pip等工具
RUN apt-get update -y \
    && apt-get install -y python-pip python-dev build-essential

# 复制当前目录内容到容器
COPY . /app

# 指定容器内的工作目录
WORKDIR /app

RUN pip install -r requirements.txt

ENTRYPOINT ["python"]
CMD ["app.py"]

```

### 创建镜像和执行
```
# 创建镜像
docker build -t flask:v1.0 .

# 运行应用
docker run -d -p 5000:5000 flask:v1.0
```
此时应该就能访问到页面了


### 总结

#### `ENTRYPOINT`和`CMD`
> `ENTRYPOINT`的目的和`CMD`一样，都是在指定容器启动程序及参数, 但是当指定了`ENTRYPOINT`后, `CMD`就不是直接运行其命令, 而是将其内容作为参数传给`ENTRYPOINT`, 也就是执行时会变为
```
<ENTRYPOINT> "<CMD>"
```
[链接](https://yeasy.gitbooks.io/docker_practice/content/image/dockerfile/entrypoint.html)

#### 关于`docker run`命令
该命令的格式为
```
sudo docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...]
```
常用的参数有
- `-d`: 后台运行容器, 返回容器ID
- `-p`: 指定端口或映射, 如:
	```
	docker run -p 5000:5000 flask:v1.0
	```
	表示将容器的80端口映射到主机的80端口
- `-i`: 交互模式运行
- `-t`: 为容器重新分配一个伪输入终端, 一般和`-i`一起使用
- `--name`: 为容器指定一个名称

[Docker run常用参数](http://www.runoob.com/docker/docker-run-command.html)
