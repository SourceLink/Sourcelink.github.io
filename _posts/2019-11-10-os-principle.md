---
layout: post
title:  "【从零开始写RTOS】操作系统原理"
date:   2019-11-10
catalog:  true
author: Sourcelink
tags:
    - 
---


# 一. 什么是操作系统?

缩写: OS, Operating System, 管理计算机硬件和软件资源的系统软件; 

- 1. 可以尽可能的有效利用单片机的硬件资源
- 2. 可以高效的运行和处理我们的业务逻辑


# 二. 使用RTOS


## 2.1 无OS


```
int main()
{

    while (1) {
        采集传感器数据;
        
        上传数据;
        
        根据数据需要控制某些东西;
    }

    return 0;
}
```


## 2.2 OS


```

void proc_1()
{
    while (1) {
        采集传感器数据;
    }

}


void proc_2()
{
    while (1) {
        上传数据;
    }

}


void proc_3()
{
    while (1) {
        根据数据需要控制某些东西;
    }

}


int main()
{

    OS_INIT();

    OS_PROC_CREATE(proc_1);
    OS_PROC_CREATE(proc_2);
    OS_PROC_CREATE(proc_3);
    
    OS_START();
    
    return 0;
}
```


# 三. 操作系统原理


## 3.1 调度器


### 3.1.1 合作式调度器

有点像这样:

```
int main()
{

    while (1) {
        proc_1();
        proc_2();
        proc_3();
    }    

    return 0;
}
```

- 这里进程是不存在抢占的, 必须等到其他进程自愿释放CPU的控制权, 其他进程才可以得到执行;


### 3.1.2 抢占式调度器


- 1. 每个进程都有对应的优先级
- 2. 最高优先级的任务一旦就绪, 总是可以得到cpu的控制权


### 3.1.3 时间片调度器


```
void proc_2()
{
    while (1) {
        上传数据;
        ms_sleep(100);
    }
}

void proc_3()
{
    while (1) {
        上传数据;
    }
}
```

时间片调度器会为每个进程分配时间片, 当时间片被耗尽的时候, 该进程会被挂起; 


## 3.2 进程管理

- 1. 进程状态

- 2. 进程间通信

- 3. 进程间同步



##  3.3 时间管理

- 1. 软件定时器

- 2. 休眠管理
