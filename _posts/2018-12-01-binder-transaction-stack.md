---
layout: post
title:  "Binder机制情景分析之transaction_stack"
date:   2018-12-01
catalog:  true
tags:
    - android
    - binder
    - Linux
    - driver
    - transaction_stack
---
# 一. 概述

这里以注册服务为例,当`led_control_service`请求注册服务时是通过handle找到的`ServiceManager`,但是`ServiceManager`是如何找到`led_control_service`进行回复的呢?

答:这里驱动中用到了一个传送栈记录了发送和接收进程,线程的信息;  接下来我们讲下具体的流程;


# 二. transaction_stack的来源

主要代码在`binder_transaction()`和`binder_thread_read()`中;  

## 2.1 BC_TRANSACTION

从`led_control_service`调用`BC_TRANSACTION`开始,此时驱动会调用`binder_transaction()`函数;  

如下是该函数中的一段代码,这段代码前面在深入驱动时未讲解:  
```
    if (!reply && !(tr->flags & TF_ONE_WAY))                  ①
        t->from = thread;
    else
        t->from = NULL;
    t->sender_euid = task_euid(proc->tsk);
    t->to_proc = target_proc;                                 ②
    t->to_thread = target_thread;
    t->code = tr->code;
    t->flags = tr->flags;
    t->priority = task_nice(current);
```

> ①: 判断是否需要回复,需要则记录下当前进程的thread信息;   
> ②: 记录下要目标进程信息和线程信息;  

这里`t`是`struct binder_transaction `结构,和`transaction_stack`类型相同;   

```
struct binder_transaction {
	int debug_id;
	struct binder_work work;
	struct binder_thread *from;                                 ①
	struct binder_transaction *from_parent;                     ②
	struct binder_proc *to_proc;                                ③
	struct binder_thread *to_thread;                            ④
	struct binder_transaction *to_parent;                       ⑤
	unsigned need_reply:1;
    .....
};
```
> ①: 记录发送线程信息;  
> ②: 记录发送线程的传输栈的父栈;  
> ③: 记录接收进程的进程信息;  
> ④: 记录接收线程的进程信息;  
> ⑤: 记录接收进程的传输栈的父栈;  

此时`t`变量中的主要信息如下:  

| transaction_starck| |
|---|---|
|from| led_control_service'thread|
|to_proc| ServiceManager|
|to_thread| ServiceManager'thread|


此时已经记录下了接收方的信息了,继续往下看(`binder_transaction()`函数内):  

```
    if (reply) {
        BUG_ON(t->buffer->async_transaction != 0);
        binder_pop_transaction(target_thread, in_reply_to);
    } else if (!(t->flags & TF_ONE_WAY)) {                     ①
        BUG_ON(t->buffer->async_transaction != 0);
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;            ②
        thread->transaction_stack = t;                         ③
    } else {
        BUG_ON(target_node == NULL);
        BUG_ON(t->buffer->async_transaction != 1);
        if (target_node->has_async_transaction) {
            target_list = &target_node->async_todo;
            target_wait = NULL;
        } else
            target_node->has_async_transaction = 1;
    }
```

> ①: 判断是否需要回复;  
> ②: 这里一个入栈操作将当前线程的传输压入,但是`thread->transaction_stack`此时为`NULL`,因为第一次执行前面没有赋值;   
> ③: 接下给当前线程的传输栈赋值,;

此时`led_control_service`线程的传输栈信息如下:  

| transaction_starck| |
|---|---|
|from| led_control_service'thread|
|to_proc| ServiceManager|
|to_thread| ServiceManager'thread|
|from_parent| NULL|


## 2.2 BR_TRANSACTION

前面驱动讲解过,在`led_contrl_server`执行`binder_transaction()`后,`ServiceManager`的进程会被唤醒,则`ServiceManager`会从休眠中醒来继续执行`binder_thread_read`;  

`binder_thread_read`有段代码如下:  

```
    if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
        t->to_parent = thread->transaction_stack;                ①
        t->to_thread = thread;                                   ②
        thread->transaction_stack = t;                           ③
    } else {
        t->buffer->transaction = NULL;
        kfree(t);
        binder_stats_deleted(BINDER_STAT_TRANSACTION);
    }
```

> ①: 将当前线程传输栈入栈;  
> ②: 记录接收进程的信息,也就是自己本身,因为他本身是被唤醒的;  
> ③: 当前线程传输栈记录下线程信息; 

这里的`t`是从带处理事务的链表中取出来的,也就是前面`led_control_service`挂到`ServiceManager`的todo链表上;  

则`ServiceManager`线程的传输栈的信息如下:  

| transaction_starck| |
|---|---|
|from| led_control_service'thread|
|to_proc| ServiceManager|
|to_thread| ServiceManager'thread|
|from_parent| NULL|
|to_parent| NULL|

到这里线程传输栈的数据来源和构造讲完;

# 三. transaction_stack的使用

`ServiceManager`的用户态在处理完注册信息后,调用`BC_REPLAY`命令回复注册结果给`led_control_service`,此时驱动中也是调用到`binder_transaction()`; 

## 3.1 BC_REPLAY

`binder_transaction`代码片如下:  

```
if (reply) {
		in_reply_to = thread->transaction_stack;                   ①
		ALOGD("%s:%d,%d %s in_reply_to = thread->transaction_stack\n", proc->tsk->comm, proc->pid, thread->pid, __FUNCTION__);
		if (in_reply_to == NULL) {
			binder_user_error("%d:%d got reply transaction with no transaction stack\n",
					  proc->pid, thread->pid);
			return_error = BR_FAILED_REPLY;
			goto err_empty_call_stack;
		}
		binder_set_nice(in_reply_to->saved_priority);
		if (in_reply_to->to_thread != thread) {                    ②
			return_error = BR_FAILED_REPLY;
			in_reply_to = NULL;
			goto err_bad_call_stack;
		}
		thread->transaction_stack = in_reply_to->to_parent;        ③
		target_thread = in_reply_to->from;                         ④
		if (target_thread == NULL) {
			return_error = BR_DEAD_REPLY;
			goto err_dead_binder;
		}
		if (target_thread->transaction_stack != in_reply_to) {
			return_error = BR_FAILED_REPLY;
			in_reply_to = NULL;
			target_thread = NULL;
			goto err_dead_binder;
		}
		target_proc = target_thread->proc;
	} else {
```

> ①: 用个临时变量记录下当前线程额传输栈信息;  
> ②: 判断下接收的线程是否为自己本身,如果不是则出错,看迷糊的可以看下再2.2节; 
> ③: 一次出栈操作,此时`thread->transaction_stack`值为`NULL`了;  
> ④: 获取到目标线程,`from`中记录着发送线程`led_contol_service`的信息;  

这里就通过`ServiceManager`在read的时候入栈的传送栈信息,获取到发送进程的信息,即回复进程的信息;  

接着往下看:  

```
    if (reply) {
        BUG_ON(t->buffer->async_transaction != 0);
        binder_pop_transaction(target_thread, in_reply_to);    ①
    } else if (!(t->flags & TF_ONE_WAY)) {
        BUG_ON(t->buffer->async_transaction != 0);
        t->need_reply = 1;
        t->from_parent = thread->transaction_stack;
        thread->transaction_stack = t;
    } else {
      ...
    }
```

> ①: 一个出栈操作; 

此时`led_control_service`的传输栈也指向了父栈,即为空且清除了`in_reply_to`中`from`的信息;  





