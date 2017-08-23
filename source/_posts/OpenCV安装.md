---
title: OpenCV安装
date: 2017-05-18 06:38:51
categories: 工具
tags:
  - OpenCV
---

一直做爬虫的工作, 公司让爬取的信息比较敏感, 各个网站的保护措施越来越好, 验证码是绕不开的, 首先进行环境安装吧
## 使用虚拟环境安装

本篇翻译自[ubuntu16.04: opencv3.0和python2.7+的安装](http://www.pyimagesearch.com/2016/10/24/ubuntu-16-04-how-to-install-opencv/)

#### 安装教程

- [ubuntu14.04: opencv3.0和python2.7+的安装](http://www.pyimagesearch.com/2015/06/22/install-opencv-3-0-and-python-2-7-on-ubuntu/)
- [ubuntu16.04: opencv3.0和python2.7+的安装](http://www.pyimagesearch.com/2016/10/24/ubuntu-16-04-how-to-install-opencv/)

#### 参考资料

- [官方教程](https://opencv-python-tutroals.readthedocs.io/en/latest/)
- [github](https://github.com/opencv/opencv)

-----

我的环境`ubuntu16.04`和`python2.7`

#### 1. 安装依赖环境

```
# 更新
sudo apt-get update
sudo apt-get upgrade

# 开发环境
sudo apt-get install build-essential cmake pkg-config

sudo apt-get install libjpeg8-dev libtiff5-dev libjasper-dev libpng12-dev

sudo apt-get install libavcodec-dev libavformat-dev libswscale-dev libv4l-dev
sudo apt-get install libxvidcore-dev libx264-dev

sudo apt-get install libgtk-3-dev

sudo apt-get install libatlas-base-dev gfortran

sudo apt-get install python2.7-dev python3.5-dev
```

<!--more-->

#### 2. 下载`opencv`

```shell
cd ~
wget -O opencv.zip https://github.com/Itseez/opencv/archive/3.1.0.zip
unzip opencv.zip
```

接着下载`opencv_contrib`
```shell
wget -O opencv_contrib.zip https://github.com/Itseez/opencv_contrib/archive/3.1.0.zip
unzip opencv_contrib.zip
```

#### 3. 创建虚拟环境

创建一个关于opencv的虚拟环境cv, 进入虚拟环境后执行后续操作, 使用`virtualenv`和`virtualenvwrapper`, 详细内容可见原文
```shell
mkvirtualenv cv -p python2/python3
workon cv    # 应该会自动进入
```
在虚拟环境中安装包
```shell
pip install numpy
```

#### 5. 配置和编译opencv

之前, 一定确保处于创建的虚拟环境中

目前我解压后的`opencv`和`open_contrib`都位于根目录下`~`, 因为下面都是使用的该路径
```shell
cd ~/opencv-3.1.0/
mkdir build
cd build
cmake -D CMAKE_BUILD_TYPE=RELEASE \
>     -D CMAKE_INSTALL_PREFIX=/usr/local \
>     -D INSTALL_PYTHON_EXAMPLES=ON \
>     -D INSTALL_C_EXAMPLES=OFF \
>     -D OPENCV_EXTRA_MODULES_PATH=~/opencv_contrib-3.1.0/modules \
>     -D PYTHON_EXECUTABLE=~/.virtualenvs/cv/bin/python \
>     -D BUILD_EXAMPLES=ON ..

```

这里会暂停一会下载下面这个东西, __勿动__.
```
-- ICV: Downloading ippicv_linux_20151201.tgz...
```

我们创建了一个build文件夹, 这个文件夹里是真正编译后的内容
如果以上过程没有任何问题:
```shell
make -j4
```
`-j4`表达的意思是开启4个进程
开启多进程可以加快编译速度, 但是可能会冲突造成失败, 如果失败, 请刷新编译, 使用单核进行, __失败后执行以下语句:__
```shell
make clean
make
```
编译成功后最后一步则是安装opencv
```shell
sudo make install
sudo ldconfig
```

#### 5. 完成安装
执行完`sudo make install`后, 可以执行一下命令进行验证
```shell
(cv) ql@ql:~/opencv-3.1.0/build$ ls -l /usr/local/lib/python2.7/site-packages/
总用量 1972
-rw-r--r-- 1 root staff 2016608 5月   4 11:16 cv2.so
(cv) ql@ql:~/opencv-3.1.0/build$
```

__Note:__ 在某些情况下, 你会发现`OpenCV`被安装在`*/usr/local/lib/python2.7/dist-packages*`而不是`*/usr/local/lib/python2.7/site-packages*`(dist-packages和site-packages是相对的), 如果`cv2.so`不是在`site-packages`, 请检查`dist-packages`

最后一步是绑定`cv2.so`到`cv`虚拟环境
```shell
cd ~/.virtualenvs/cv/lib/python2.7/site-packages/
ln -s /usr/local/lib/python2.7/site-packages/cv2.so cv2.so
```



#### 6. 测试OpenCV安装结果
进行测试前确保以下内容:
- 新开一个终端
- 进入cv虚拟环境
- 尝试导入python

```shell
cd ~
workon cv
```
```python
In [1]: import cv2

In [2]:

In [2]: cv2.__version__
Out[2]: '3.1.0'

In [3]:
```
安装成功后可以删除压缩包
```shell
cd ~
rm -rf opencv-3.1.0 opencv_contrib-3.1.0 opencv.zip opencv_contrib.zip
```

## 不使用虚拟环境安装opencv
```shell
sudo apt-get install build-essential
sudo apt-get install cmake git libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev
sudo apt-get install python-dev python-numpy libtbb2 libtbb-dev libjpeg-dev libpng-dev libtiff-dev libjasper-dev libdc1394-22-dev

wget https://codeload.github.com/Itseez/opencv/zip/2.4.12
cd opencv-2.4.12
mkdir release
cd release
cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local ..

make
sudo make install
```

