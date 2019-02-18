---
layout: post
title:  "Binder机制情景分析之linux环境适配"
date:   2018-11-25
catalog:  true
tags:
    - android
    - binder
    - Linux
    - 环境
---

# binder安装


# 一. 环境
    - 运行环境：linux4.1.15
    - 开发板为天嵌imx6ul

**注意：**
>不使用脚本编译  
>自己更改makefile中的ARCH CROSS_COMPILE

# 二. 内核修改

# 2.1 打开内核配置菜单

```
make menuconfig 
```

# 2.2 修改配置
- 1. 配置驱动  

转到Device Drivers->Android，选中`Andoid Drivers`和`Android Binder IPC Driver`  

示例如下:  

![android_dirvers](/images/binder/make_menuconfig_android_dirvers.png)

- 2. 配置binder驱动中使用到接口  

转到Device Drivers->Staging drivers->Android，选中:  
`Enable the Anonymous Shared Memory Subsystem`,   
`Synchronization framework`,  
`Software synchronization objects`,   
`Userspace API for SW_SYNC`  ;

示例如下:  
![android sync](/images/binder/make_menucconfig_androidsyn.png)

# 2.3 重新编译

```
make zImage -j4 
```

# 三. 查看
 
 将重新编译好的内核更新到开发板中;
 用`ls`命令查看`/dev`下是否有个设备为`binder`



