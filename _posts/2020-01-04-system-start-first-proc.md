---
layout: post
title:  "【从零开始写RTOS】系统启动第一个进程"
date:   2020-01-04
catalog:  true
author: Sourcelink
tags:
    - RTOS
---


# 一. 准备工作

## 1.1 初始化

```
void kos_sys_init(void)
{
    /* 1. 初始化就绪队列 */
    kos_rq_init();
    /* 2.初始化启动标志 */
    kos_running = 0;
    /* 3. 空闲进程的创建 */
    _idle_proc_create();
}
```


## 1.2 空闲进程实现

```
void *kos_proc_idle(void *_arg)
{
    unsigned int state = 0;
    while(1) {
        state = kos_cpu_enter_critical();
        idle_proc_count++;
        kos_cpu_exit_critical(state);
    }
    return NULL; 
}

static void _idle_proc_create(void)
{
    kos_proc_create(&idle_proc, idle_proc_stack, 64, KOS_CONFIG_LOWEST_PRIO, kos_proc_idle, NULL, "idle_proc");
}
```


# 二. 系统启动

```
void kos_sys_start(void)
{
    kos_ready_proc = kos_rq_highest_ready_proc();
    kos_cpu_sched_start();
}
```

# 三. 创建第一个进程

```
#define PROC1_STACK_SIZE	64

struct kos_proc proc1;
unsigned int proc1_stack[PROC1_STACK_SIZE];

void *proc1_fun(void *_arg)
{
	printf("proc 1\r\n");

    return NULL; 
}


int main(void)
{
	board_init();

	printf("systme init ok\r\n");

	kos_sys_init();

	kos_proc_create(&proc1, proc1_stack, PROC1_STACK_SIZE, 1, proc1_fun, NULL, "proc1");

	kos_sys_start();

    return 0;
}
```

