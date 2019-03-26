---
layout: post
title:  "czmq交叉编译"
date:   2019-03-26 21:11:54
catalog:  true
tags:
    - czmq
    - libcurl
    - libuuid
    - libzmq
    
---

# 一. libzmq交叉编译

- 下载源码:

```
git clone git://github.com/zeromq/libzmq.git
```

- 生成环境

```
cd libzmq && ./autogen.sh
```

- 配置交叉编译

```
./configure --host=arm-linux --prefix=/home/sourcelink/opt/arm-libzmq/
```

其中 **--host** 指交叉编译工具链, **--prefix**指库安装路径;  


- 编译并安装

```
make && make install
```

# 二. libuuid交叉编译

- 下载源码

```
https://sourceforge.net/projects/libuuid/
```

- 配置交叉编译环境

```
cd libuuid-1.0.3
./configure --host=arm-linux --prefix=/home/sourcelink/opt/arm-uuid/
```

- 编译并安装

```
make && make install
```

# 二. libcurl交叉编译

- 下载源码

下载地址:  

```
https://curl.haxx.se/download/
```

下载:  

```
wget https://curl.haxx.se/download/curl-7.63.0.tar.gz
```

- 配置交叉编译环境

```
cd curl-7.63.0
./configure --host=arm-linux --prefix=/home/sourcelink/opt/arm-libcurl/
```


- 编译并安装

```
make && make install
```

# 三. libczmq交叉编译


- 下载源码

```
git clone git://github.com/zeromq/czmq.git
```

- 生成环境

```
cd czmq && ./autogen.sh
```

- 配置交叉编译

```
./configure --host=arm-linux --prefix=/home/sourcelink/opt/arm-czmq/ CFLAGS="-I/home/sourcelink/opt/arm-libzmq/include -I/home/sourcelink/opt/arm-uuid/include"  LDFLAGS="-L/home/sourcelink/opt/arm-libzmq/lib -L/home/sourcelink/opt/arm-uuid/lib -L/home/sourcelink/opt/arm-libcurl/lib" 
```

其中**CFLAGS**指定头文件路径, **LDFLAGS**指定库路径;  

在执行configure时, 会检测`libzmq.so`, `libuuid.so`等, 如果使用的是交叉工具链包则需要添加搜索路径, 在此路径下添加.pc文件

```
/opt/oecore-x86_64/sysroots/armv7a-vfp-neon-oe-linux-gnueabi/usr/lib/pkgconfig
```

为什么是这个路径?

因为我的**PKG_CONFIG_PATH**环境变量是设置为这个; 

如果该路径下没有pkgconfig目录则手动创建, 随后将你前面交叉编译出来的文件中pkgconfig目录下的.pc文件拷贝到该目录下, 再重新执行`./configure`;  

如果不明白可以搜索下`PKG_CONFIG_PATH`或`pkgconfig`的使用;  

- 编译并安装

```
make && make install
```














