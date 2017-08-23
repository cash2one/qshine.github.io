---
title: python开发环境安装
date: 2017-04-20 06:37:27
categories: 工具
tags:
  - python开发环境
---

环境: ubuntu16.04
主要来安装`virtualenv`和`virtualenvwrapper`, 前者可以让python环境相互隔离, 或者可以方便的进入虚拟环境

## 安装pip
```shell
cd ~
wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
```

## 安装`virtualenv`和`virtualenvwrapper`
```shell
sudo pip install virtualenv virtualenvwrapper
sudo rm -rf ~/get-pip.py ~/.cache/pip
```

## 更新`~/.bashrc`, 在最下面加上以下内容
```shell
# virtualenv and virtualenvwrapper
export WORKON_HOME=$HOME/.virtualenvs
source /usr/local/bin/virtualenvwrapper.sh
```

## 在终端下执行以下内容
```shell
echo -e "\n# virtualenv and virtualenvwrapper" >> ~/.bashrc
echo "export WORKON_HOME=$HOME/.virtualenvs" >> ~/.bashrc
echo "source /usr/local/bin/virtualenvwrapper.sh" >> ~/.bashrc
```

## 重启`~/.bashrc`
```shell
source ~/.bashrc
```


## 创建虚拟环境
```shell
# 使用python2
mkvirtualenv venv -p python2
# 使用python3
mkvirtualenv venv -p python3
```
使用
```shell
workon venv
```

## 删除已创建环境
```shell
# 列出所有环境
workon
# 删除某个环境
rmvirtualenv name
```

---------

## centos下安装
参考： http://www.debugrun.com/a/blYVrDo.html

<!--more-->
