---
layout: post
title:  "scrcpy构建"
date:   2019-01-14 22:10:54
catalog:  true
tags:
    - scrcpy
    - android
---

# 一. 概述

scrcpy是个android投屏插件

Github：https://github.com/Genymobile/scrcpy  
下载地址：https://github.com/Genymobile/scrcpy/releases

# 二. 手动构建

因为笔者用的ubuntu16.04所以需要手动构建;

## 2.1 安装环境依赖 

```
# runtime dependencies
sudo apt install ffmpeg libsdl2-2.0.0

# client build dependencies
sudo apt install make gcc pkg-config meson ninja-build \
                 libavcodec-dev libavformat-dev libavutil-dev \
                 libsdl2-dev

# server build dependencies
sudo apt install openjdk-8-jdk
```

这里必须使用java8不然后面构建会有问题.


## 2.2 构建

- 1. 设置android studio sdk的位置

```
export ANDROID_HOME=~/android/sdk
```

- 2. 下载源码并构建  

```
git clone https://github.com/Genymobile/scrcpy
cd scrcpy
meson x --buildtype release --strip -Db_lto=true
cd x
ninja
```

- 3. 安装

```
sudo ninja install 
```
这样就可以使用`scrcpy`命令进行操作了


# 三. 错误处理

在运行时遇到一个提示如下:  

```
371 KB/s (19291 bytes in 0.050s)
ERROR: Exception on thread Thread[main,5,main]
java.lang.IllegalArgumentException: Crop must contains 4 values separated by colons: "''"
        at com.genymobile.scrcpy.Server.parseCrop(Server.java:91)
        at com.genymobile.scrcpy.Server.createOptions(Server.java:72)
        at com.genymobile.scrcpy.Server.main(Server.java:108)
        at com.android.internal.os.RuntimeInit.nativeFinishInit(Native Method)
        at com.android.internal.os.RuntimeInit.main(RuntimeInit.java:249)
```


查看源码发现是投屏尺寸问题,设置尺寸即可:  

```
scrcpy -c 800:800:0:0
```


