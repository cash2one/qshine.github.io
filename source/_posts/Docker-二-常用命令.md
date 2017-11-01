---
title: 'Docker(二)常用命令'
date: 2017-08-05 14:12:18
categories: Docker
tags:
---




### 常用命令
1. 启动/停止/重启服务
   ```
   sudo service docker start/stop/restart
   ```
2. 获取镜像到本地
   ```
   docker pull [选项] [Docker Registry地址]<仓库名>:<标签>
   ```
   如:
   ```
   docker pull ubuntu:16.04
   ```
   `ubuntu`是仓库的名字, `16.04`是版本标签
3. 查看镜像 `docker images`
   ```
   ql@ubuntu:~$ docker images
   REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
   ubuntu              16.04               d355ed3537e9        2 weeks ago
   ```
4. 查看容器
   - 所有容器 `docker ps -a`
   - 正在运行的容器 `docker ps`
5. 删除容器 `docker rm <CONTAINER ID>`

6. 删除镜像 `docker rmi <IMAGE ID>`

7. 运行容器 `docker run -it ubuntu:16.04`
   - `i`是交互式
   - `t`是终端
   此时便可以进入容器内部
8. 重新登录容器
    exit后会退出容器, 重新登录执行以下代码
    ```
    docker start <container_name>
    docker attach <container_name>    # 回车两次
    ```



### 容器转换为镜像
```
docker commit -m "测试提交" -a 'qshine' c02d76065be8 qshine/ubuntu-test:v1
```

- `-m`是提交说明信息
- `-a`表示指定用户信息
- `c02d76065be8`表示容器的`CONTAINER ID`(使用`docker ps -a`查看)
- `qshine/ubuntu-test:v1`

```
ql@ubuntu:~$ docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
qshine/ubuntu-test   v1                  e4384c0d750f        23 minutes ago      119.2 MB
ubuntu               16.04               d355ed3537e9        3 weeks ago         119.2 MB
```


**注:** 不要使用`docker commit`来定制镜像, 应该使用`Dockerfile`来定制

镜像使用的分层存储, 除当前层外, 之前的每一层是不会发生改变的, 就是说任何修改的结果仅仅在当前层进行标记, 添加, 修改, 而不会动上一层, 如果使用`docker commit`来制作镜像的话, 每一次修改都会导致镜像臃肿

### 上传至镜像仓库

官方地址为: [Docker Hub](https://hub.docker.com/), 现在创建的镜像只有本地能用, 同步到远程, 别人也能使用, 也能做保存使用. 感觉跟Git的本地仓库和远程仓库差不多

```
docker login    # 登录
docker push qshine/ubuntu-test:v1    # 同步到远程
```



