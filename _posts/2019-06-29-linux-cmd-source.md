---
layout: post
title:  "linux命令源码查看"
date:   2019-06-29 11:46:56
catalog:  true
author: Sourcelink
tags:
    - 命令源码
    - linux

---



# 一. 概述

想参考`tar`命令进行文件打包, 所以去网上找下该命令的源码, 发现一个比较特别的方式在此录下;  

>宿主机 ： Ubuntu 16.04 LTS / X64  


# 二. 命令定位

先使用`which`命令查看下`tar`命令在哪, 具体命令如下:  

```
➜  ~ which tar
/bin/tar
```

# 三. 命令来源

查看`tar`命令属于哪个源码包, 具体命令如下:  

```
➜  ~ dpkg -S /bin/tar 
tar: /bin/tar
```


# 四. 下载源码包

从上一步中可以知道`tar`命令的实现在包`tar`中, 用apt安装该包的源代码然后解压:  

```
sudo apt-get source tar
tar -xf tar_1.28.orig.tar.xz
```




