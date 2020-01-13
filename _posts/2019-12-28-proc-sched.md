---
layout: post
title:  "【从零开始写RTOS】进程调度"
date:   2019-12-28
catalog:  true
author: Sourcelink
tags:
    - RTOS
---

# 一. 调度原理

实质是触发一个上下文切换

```
port_os_ctxsw:
	push	{r4, r5}
    ldr     r4, =NVIC_INT_CTRL
    ldr     r5, =NVIC_PENDSVSET
    str     r5, [r4]
	pop		{r4, r5}
	
    bx      lr  
```

会引发一个服务函数`PendSV_Handler`， 该函数中完成一次进程的切换过程; 


# 二. 调度实现

> 1. 系统需要运行  
> 2. 得最高优先级就绪进程  
> 3. 得判断获取的就绪进程是否与当前进程  
> 4. 调用kos_cpu_ctxsw()函数触发上下文切换  

```
void kos_sched(void)
{
    unsigned int state = kos_cpu_enter_critical();

    if (!kos_sys_is_running()) {
        kos_cpu_exit_critical(state);
        return ;
    }

    kos_ready_proc = kos_rq_highest_ready_proc();

    if (kos_ready_proc == NULL || 
        kos_ready_proc == kos_curr_proc) {
            kos_cpu_exit_critical(state);
            return ;
    }

    kos_cpu_exit_critical(state);

    kos_cpu_ctxsw();
}
```

# 三. 中断调度


在中断处理函数中不允许进程调度;

所以需要实现当前是否在中断中的判断；


```
void kos_sched(void)
{
    unsigned int state = kos_cpu_enter_critical();

    if (!kos_sys_is_running() || kos_sysy_is_inirq()) {
        kos_cpu_exit_critical(state);
        return ;
    }

    kos_ready_proc = kos_rq_highest_ready_proc();

    if (kos_ready_proc == NULL || 
        kos_ready_proc == kos_curr_proc) {
            kos_cpu_exit_critical(state);
            return ;
    }

    kos_cpu_exit_critical(state);

    kos_cpu_ctxsw();
}
```