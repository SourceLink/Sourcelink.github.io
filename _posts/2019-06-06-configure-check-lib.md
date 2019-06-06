---
layout: post
title:  "如何让自己configure找到需要的库?"
date:   2019-06-06 14:10:56
catalog:  true
author: Sourcelink
tags:
    - configure
    - pkgconfig

---

# 一. 概述

原来下[czmq交叉编译](https://sourcelink.top/2019/03/26/arm-czmq-compile/)时有讲解过这个问题, 但是觉得还是很有比较单独拎出来讲解;  
当时在交叉编译libczmq时, 在执行configure时, 会检测`libzmq.so, libuuid.so`等, 不论我使用`LDFLAGS="-L xxx"`指定库的如今都无法check通过;  

# 二. 原因

原来`LDFLAGS="-L xxx"`仅仅只是在生成makefile的时候使用, 让其交叉编译时能进入该路径搜索动态库, 但是这个检查项目依赖的动态链接库的步骤是通过pkg-config工具去完成的;  

终于知道了问题的所在了? 那怎么办呢？

# 三. 对策

我们先了解下`pkg-config`工具, 维基百科是这样解释的**pkg-config 是一个在源代码编译时查询已安装的库的使用接口的计算机工具软件**, pkg-config工具是去读指定路径下所有.pc文件, 
而路径的指定则和环境变量`PKG_CONFIG_PATH`有关, 在我前面进行交叉编译时指定的路径如下:  

```
/opt/oecore-x86_64/sysroots/armv7a-vfp-neon-oe-linux-gnueabi/usr/lib/pkgconfig
```


如果该路径下没有**pkgconfig目录**则手动创建;  


我们可以看下.pc文件中的内容, 已libzmq为例, 该文件在安装目录下的lib目录`libzmq.pc`:  

```
prefix=/home/sourcelink/opt/arm-libzmq
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include

Name: libzmq
Description: 0MQ c++ library
Version: 4.3.2
Libs: -L${libdir} -lzmq
Libs.private: -lstdc++  -lpthread -lrt
Requires.private: 
Cflags: -I${includedir} -DZMQ_BUILD_DRAFT_API=1
```

这个文件告诉我们这些库可以在`/home/sourcelink/opt/arm-libzmq/lib`找到，头文件可以在`/home/sourcelink/opt/arm-libzmq/include`里找到，库的名字是 libzmq 并且版本号是4.3.2。它也提供了用于编译依赖于libzmq的源代码时需要的链接器参数。


接着将你前面交叉编译`libzmq, libuuid`出来的文件中pkgconfig目录下的`.pc`文件拷贝到该目录下, 再重新执行`./configure`就可以了;


