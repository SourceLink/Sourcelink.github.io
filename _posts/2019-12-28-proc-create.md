---
layout: post
title:  "【从零开始写RTOS】进程创建"
date:   2019-12-28
catalog:  true
author: Sourcelink
tags:
    - RTOS
---


# 一. 需求

> 1.进程堆栈指针
> 2. 进程优先级
> 3. 进程的操作链表
> 4. 进程状态
> 5. 进程名
> 6. 休眠链表
> 7. 进程休眠时间
> 8. 进程ID


# 二. 进程结构体


```
struct kos_proc {
    unsigned int *stack_pointer;
    
    struct list_head slot_list;
    
    unsigned int  priority;
    
    unsigned int  state;
    
    const char *proc_name;
    
    struct list_head tick_list;
    unsigned int     wait_time;

    unsigned int pid;
};
```

# 三. 进程创建

## 3.1 通过pid找到进程

```
    state = kos_cpu_enter_critical();

    for (i = 1; i <= KOS_CONFIG_PID_LIST; i++) {
        if (kos_proc_tab[i] == NULL) {
            pid = i;
            break;
        }
    }

    if (pid == 0) {
        kos_cpu_exit_critical(state);
        return -1;
    }

    kos_proc_tab[pid] = _proc;
```


## 3.2 创建堆栈

```
_proc->stack_pointer = kos_proc_stack_init((void *)entry, _arg, _stack_addr, _stack_size);
```
## 3.3 初始化控制变量

```
    _proc->priority = _prio;
    _proc->proc_name = name;
    _proc->state = KOS_PROC_READY;
    _proc->tick_wait = 0;
```

## 3.4 初始化进程链表

```
    list_head_init(&_proc->slot_list);
    list_head_init(&_proc->tick_list);
```

## 3.5 添加到就绪队列

```
kos_rq_add(_proc);
```

## 3.6 执行调度

```
kos_sched();
```

