---
layout: post
title:  "mosquitto安装"
date:   2019-02-15 20:35:54
catalog:  true
tags:
    - mosquitto
    - mqtt
---


# 一. 下载源码

```
git clone git@github.com:eclipse/mosquitto.git
```


# 二. 修改config.mk


```
c-ares (libc-ares-dev on Debian based systems) - disable with make WITH_SRV=no
libuuid (uuid-dev) - disable with make WITH_UUID=no
libwebsockets (libwebsockets-dev) - enable with make WITH_WEBSOCKETS=yes
openssl (libssl-dev on Debian based systems) - disable with make WITH_TLS=no
xsltproc (xsltproc and docbook-xsl on Debian based systems) - only needed when building from git sources - disable with make WITH_DOCS=no
```


# 三. 编译

## 3.1 编译可执行命令

```
make binary
```

在编译时遇到的问题如3.2节, 得到的`mosquitto`在src目录, `mosquitto_pub`和`mosquitto_sub`在client目录;  


## 3.2 问题

- 问题描述  

```
mosquitto.c:49:29: fatal error: libwebsockets.h: No such file or directory
```

- 问题对策   

需安装`libwebsockets`, 详情如下:   

```
sudo apt-get install libssl-dev
git clone https://github.com/warmcat/libwebsockets.git
cd libwebsockets
mkdir build
cd build
cmake ..
make
sudo make install
sudo ldconfig
```


# 四. 命令测试

## 4.1 运行代理服务器  

```
mosquitto &
```

## 4.2 订阅消息  

```
mosquitto_sub -t "topic" &
```

## 4.3 发布消息  

```
mosquitto_pub -t "topic" -m "hello"
```


## 4.4 问题对策  

在运行`mosquitto_sub`时遇到如下问题:  

- 问题  

```
./mosquitto_sub: error while loading shared libraries: libmosquitto.so.1: cannot open shared object file: No such file or directory
```


- 对策 

```
sudo cp ./lib/libmosquitto.so.1 /usr/local/lib/
sudo ldconfig
```


