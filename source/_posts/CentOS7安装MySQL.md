---
title: CentOS7安装MySQL
date: 2017-09-20 14:09:44
categories: 数据库
tags:
---

### MySQL安装步骤
```bash
# 安装服务端, 环境和客户端
yum install mysql-server mysql-devel mysql

# npm包(用到wget)
wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm

# 安装rmp包
rpm -ivh mysql-community-release-el7-5.noarch.rpm

# 检查是否安装成功
ls -1 /etc/yum.repos.d/mysql-community*

# 重新安装server
yum install mysql-server
```

### 启动
```bash
# 启动mysql
service mysqld start

# 创建root用户
mysqladmin -u root password 密码

# 进入mysql
mysql -uroot -p
```

### 新建用户并授权
#### 1. 创建用户
创建新用户, 直接使用root进行远程连接会导致登录不上
```bash
# 新创建用户
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
```
- username: 用户名
- host: 指定主机有权限登录, 本地是localhost, 任意远程登录是`%`
- password: 用户名

#### 2. 授权
```bash
GRANT privileges ON databasename.tablename TO 'username'@'host'
```
- privileges: 用户权限, 如select, insert, update等, 所有权限使用**all**
- databasename.tablename: 库名和表名, 使用`*.*`表示所有

如果新建一个用户仅仅用于远程连接使用如下
```MySQL
GRANT all ON *.* TO 'username'@'host'
```
以上命令授权的用户不能对其它用户授权

### 完全卸载MySQL
```
yum remove  mysql mysql-server mysql-libs mysql-server
find / -name mysql    # 删除找出的全部内容
rpm -qa|grep mysql    # yum remove 全部内容
```
完全卸载后便可以重装
