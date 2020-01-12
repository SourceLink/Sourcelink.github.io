---
layout: post
title:  "【从零开始写RTOS】st-link进行gdb调试"
date:   2020-01-06
catalog:  true
author: Sourcelink
tags:
    - RTOS
---

# 一. 建立监听


```
st-util -p 4500
```

`-p`是为指定监听端口


# 二. 启动gdb

```
arm-none-eabi-gdb bin/demo_proc_manager.elf
```

# 三. 建立目标

```
target extended-remote :4500
```


# 四. 再次更新板载程序

```
load
```

程序来源`bin/demo_proc_manager.elf`

示意图：
![](/images/rtos/20200106194934762_269684712.png)


# 五. 命令


## 5.1 显示函数列表

```
list 
```

缩写为`l`


## 5.2 继续运行

```
c
```


## 5.3 打断点


```
break
```

缩写`b`


exp:

```
b proc1_fun
```

## 5.4 重新运行

```
run
```

缩写`r`

## 5.5 单步运行


```
next
```

缩写`n`


## 5.6 查看寄存器信息

```
i r
```

全称`info register`





