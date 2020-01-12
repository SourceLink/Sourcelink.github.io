---
layout: post
title:  "【从零开始写RTOS】进程退出处理"
date:   2020-01-06
catalog:  true
author: Sourcelink
tags:
    - RTOS
---


# 一. 原因


根据上节调试结果，我们知道进程在退出以后产生了异常， 原因是返回后进程跳转到了异常的地方； 

**上节分析的记录**：

![](/images/rtos/20200112103656787_171415838.png)


# 二. 对策


所以我们现在需要来解决该问题，我们为所有的进程退出做一个统一的处理， 需要做到如下几点：

- 1. 清楚进程存储表  
- 2. 将进程从就绪表中移除  
- 3. 清楚进程堆栈指针  
- 4. 进行一次调度  


实现代码如下：


```
static void _proc_exit_handle(void)
{
    unsigned int state = kos_cpu_enter_critical();
    unsigned int pid = kos_curr_proc->pid;

    kos_proc_tab[pid] = NULL;
    kos_rq_delete(kos_curr_proc);
    kos_curr_proc->stack_pointer = NULL;
    kos_curr_proc = NULL;

    kos_cpu_exit_critical(state);

    kos_sched();
}
```

最后进行一次调度是为了让该进程让出cpu的资源，因为它已经没有需要执行的业务了；  




