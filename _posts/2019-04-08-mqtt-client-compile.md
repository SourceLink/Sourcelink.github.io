---
layout: post
title:  "mqtt客户端协议移植"
date:   2019-04-08 17:10:56
catalog:  true
author: sourcelink
tags:
    - mqtt client

---

# 一. 交叉编译ssl

## 1.1 源码下载

```
wget https://www.openssl.org/source/openssl-1.1.1b.tar.gz
```

##  1.2 配置环境

```
tar xf openssl-1.1.1b.tar.gz
cd openssl-1.1.1b/
./config no-asm shared --prefix=/home/sourcelink/opt/arm-ssl/ 
```

其中`/home/sourcelink/opt/arm-ssl/`是指定的安装目录, `shared`表示编译成动态库


## 1.3 修改makefile

```
CROSS_COMPILE=arm-linux-
```

接着删除第123行和第124行的`-m64`


## 1.4 编译和安装

```
make && make install
```

# 二. 交叉编译mqtt

## 2.1 源码下载


```
https://github.com/eclipse/paho.mqtt.c.git
```

## 2.2 修改makefile

```
cd paho.mqtt.c/
```

修改该目录下的Makefile  

```
CFLAGS += -I /home/sourcelink/opt/arm-ssl/include 
LDFLAGS += -L /home/sourcelink/opt/arm-ssl/lib
```

添加前面交叉编译出的ssl库路径

## 2.3 编译

```
make CC=arm-linux-gcc
```


最后编译出的文件和动态库都在`build/output`目录下


# 三. 测试

- 运行代理服务器

在你的电脑上开启mqtt服务:  

```
mosquitto &
```

- 订阅消息

将前面编译出的ssl动态库和mqtt协议的动态库拷贝到开发板上:  

![](/images/mqtt/mqtt_client_lib_tree.png)

然后修改订阅的ip地址后再重新编译:  

![](/images/mqtt/client_sub_ipconfig.png)

接着在开发板上运行`paho_cd_sub`可执行程序

```
./paho_cd_sub  -t "topic" &
```

如果出现`Connect failed return code`问题, 请检查你主机的ip地址和订阅的ip地址是否相同;  

- 发布消息

在电脑端发布消息

```
mosquitto_pub -t "topic" -m "sourcelink"
```

此时可以观察开发板的串口是否已经打印出`sourcelink`字符串;  

**备注**

这里的`mosquitto`和`mosquitto_pub`是上节编译代理服务器时编译出来的
