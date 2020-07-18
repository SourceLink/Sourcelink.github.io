---
layout: post
title:  "Linux内核之休眠与唤醒"
date:   2020-07-15
catalog:  true
author: Sourcelink
tags:
    - wait queue 
    - kernel 
---


# 一. 概述


在驱动中有当等待某件事件发生时可以使当前进程以休眠的方式进行等待，这样不会占用cpu资源； 当事件发生的时候则将其唤醒；

> 内核版本: linux4.1.15
> 主机: imx6ul

# 二. 等待唤醒API详解

## 2.1 等待方式

```
#define wait_event_interruptible(wq, condition)				\
({									\
	int __ret = 0;							\
	might_sleep();							\
	if (!(condition))						\
		__ret = __wait_event_interruptible(wq, condition);	\
	__ret;								\
})
```

上面的宏执行后可以将当前进程加入等待队列`wq`, 下面有详细的注释：

```
/**
 * wait_event_interruptible - sleep until a condition gets true
 * @wq: the waitqueue to wait on
 * @condition: a C expression for the event to wait for
 *
 * The process is put to sleep (TASK_INTERRUPTIBLE) until the
 * @condition evaluates to true or a signal is received.
 * The @condition is checked each time the waitqueue @wq is woken up.
 *
 * wake_up() has to be called after changing any variable that could
 * change the result of the wait condition.
 *
 * The function will return -ERESTARTSYS if it was interrupted by a
 * signal and 0 if @condition evaluated to true.
 */
```

当驱动使用上面方式使进程休眠，退出条件有两个， 当收到`signal`或者被唤醒且事件发生的时候（即等待条件为真）

还有一个宏，是只能等事件发生才能退出，信号不能打断：

```
#define wait_event(wq, condition)					\
do {									\
	might_sleep();							\
	if (condition)							\
		break;							\
	__wait_event(wq, condition);					\
} while (0)
```


## 2.1 唤醒方式



唤醒等待队列中的一个线程：

```
#define wake_up_interruptible(x)	__wake_up(x, TASK_INTERRUPTIBLE, 1, NULL)
```

唤醒等待队列中的`nr`个线程：

```
#define wake_up_interruptible_nr(x, nr)	__wake_up(x, TASK_INTERRUPTIBLE, nr, NULL)
```

唤醒等待队列上的所有线程:

```
#define wake_up_interruptible_all(x)	__wake_up(x, TASK_INTERRUPTIBLE, 0, NULL)
```

## 2.3 使用流程


当用户空间进程读取某一设备时，如果设备此时没有数据则调用`wait_event_interruptible`等函数，该进程将进入等待状态， 等待某一事件发生且条件成立；
当某事件发生后，调用`wake_up`等函数进行唤醒，唤醒后，其会判断条件是否成立， 如果不成立则继续进入休眠状态；
反之条件成立，则立即退出等待状态；


# 三. 内核代码详解


## 3.1 DECLARE_WAIT_QUEUE_HEAD 

```
struct __wait_queue_head {
	spinlock_t		lock;
	struct list_head	task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
```

上面是等待队列头的结构体，有两个成员分别是一个链表和一个锁，链表主要用于挂载等待队列的线程；


内核为我们提供了一个等待队列头的声明方式：

```
#define __WAIT_QUEUE_HEAD_INITIALIZER(name) {				\
	.lock		= __SPIN_LOCK_UNLOCKED(name.lock),		\
	.task_list	= { &(name).task_list, &(name).task_list } }

#define DECLARE_WAIT_QUEUE_HEAD(name) \
	wait_queue_head_t name = __WAIT_QUEUE_HEAD_INITIALIZER(name)
```

## 3.2 ___wait_event


现在以`wait_event_interruptible()`为例进行讲解， 因为该函数是个宏，现在来看下他的原型： 

```
#define __wait_event_interruptible(wq, condition)			\
	___wait_event(wq, condition, TASK_INTERRUPTIBLE, 0, 0,		\
		      schedule())
```

可以看到最终的原型在`___wait_event`， 到这里还是宏，并不是一个函数， 而且还调用了`schedule()`：

```
#define ___wait_event(wq, condition, state, exclusive, ret, cmd)	\
({									\
	__label__ __out;						\
	wait_queue_t __wait;						\
	long __ret = ret;	/* explicit shadow */			\
									\
	INIT_LIST_HEAD(&__wait.task_list);				\
	if (exclusive)							\
		__wait.flags = WQ_FLAG_EXCLUSIVE;			\
	else								\
		__wait.flags = 0;					\
									\
	for (;;) {							\
		long __int = prepare_to_wait_event(&wq, &__wait, state);\   ①
									\
		if (condition)						\
			break;						\
									\
		if (___wait_is_interruptible(state) && __int) {		\
			__ret = __int;					\
			if (exclusive) {				\
				abort_exclusive_wait(&wq, &__wait,	\
						     state, NULL);	\
				goto __out;				\
			}						\
			break;						\
		}							\
									\
		cmd;							\                              ②
	}								\
	finish_wait(&wq, &__wait);					\                  ③
__out:	__ret;								\
})
```

这个宏最主要就是上面标记的三个地方，其中cmd是调度函数`schedule()`， 接下来详细分析下这几个点；


### 3.2.1 prepare_to_wait_event

```
long prepare_to_wait_event(wait_queue_head_t *q, wait_queue_t *wait, int state)
{
	unsigned long flags;

	if (signal_pending_state(state, current))
		return -ERESTARTSYS;

	wait->private = current;                            ①
	wait->func = autoremove_wake_function;

	spin_lock_irqsave(&q->lock, flags);
	if (list_empty(&wait->task_list)) {                 ②
		if (wait->flags & WQ_FLAG_EXCLUSIVE)
			__add_wait_queue_tail(q, wait);
		else
			__add_wait_queue(q, wait);
	}
	set_current_state(state);                           ③
	spin_unlock_irqrestore(&q->lock, flags);

	return 0;
}
```

我们先看下`wait_queue_head_t`的数据结构：

```
typedef struct __wait_queue wait_queue_t;

struct __wait_queue {
	unsigned int		flags;
	void			*private;        // 用于保存本进程数据结构task_struct
	wait_queue_func_t	func;    // 唤醒的调用函数
	struct list_head	task_list;   // 链表元素， 用于挂载到wait_queue_head_t上
};
```

> ①: 构造等待队列结构体， 获取当前线程信息， 设置唤醒函数  
> ②: 先判断当前wait_queue_t是否已经挂载到了链表中，如果没有则将构造好的wait_queue_t加入wait_queue_head_t链表中  
> ③: 修改当前线程状态  


### 3.2.2 autoremove_wake_function

```
int autoremove_wake_function(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    // 唤醒进程，将该进程加入运行队列中
	int ret = default_wake_function(wait, mode, sync, key);

    // 将该进程的wait_queue_t从等待队列链表中移除
	if (ret)
		list_del_init(&wait->task_list);
	return ret;
}
```

可以看到其最终是调用`try_to_wake_up`将进程唤醒；  

```
int default_wake_function(wait_queue_t *curr, unsigned mode, int wake_flags,
			  void *key)
{
	return try_to_wake_up(curr->private, mode, wake_flags);
}
```



### 3.2.3 finish_wait

```
void finish_wait(wait_queue_head_t *q, wait_queue_t *wait)
{
	unsigned long flags;

	__set_current_state(TASK_RUNNING);                    // 重新设置进程状态为running
	/*
	 * We can check for list emptiness outside the lock
	 * IFF:
	 *  - we use the "careful" check that verifies both
	 *    the next and prev pointers, so that there cannot
	 *    be any half-pending updates in progress on other
	 *    CPU's that we haven't seen yet (and that might
	 *    still change the stack area.
	 * and
	 *  - all other users take the lock (ie we can only
	 *    have _one_ other CPU that looks at or modifies
	 *    the list).
	 */
	if (!list_empty_careful(&wait->task_list)) {           // 判断是否挂载在链表上，是则将其从链表上移除
		spin_lock_irqsave(&q->lock, flags);
		list_del_init(&wait->task_list);
		spin_unlock_irqrestore(&q->lock, flags);
	}
}
```



### 3.2.4 总结

在调用了`prepare_to_wait_event()`函数后， 设置了唤醒函数与进程状态， 接着就调用`schedule()`进行调度，该进程会进入休眠状态；
当该进程被唤醒后，检查下`condition`是否满足，满足则跳出否for循环， 调用`finish_wait()`将`wait_queue_t`从等待队列链表中移除；  



PS: 这里的进程一般是指用户空间的进程；


## 3.3 __wake_up



```
void __wake_up(wait_queue_head_t *q, unsigned int mode,
			int nr_exclusive, void *key)
{
	unsigned long flags;

	spin_lock_irqsave(&q->lock, flags);
	__wake_up_common(q, mode, nr_exclusive, 0, key);
	spin_unlock_irqrestore(&q->lock, flags);
}
```

主要是通过调用`__wake_up_common()`:

### 3.3.1 __wake_up_common

```
static void __wake_up_common(wait_queue_head_t *q, unsigned int mode,
			int nr_exclusive, int wake_flags, void *key)
{
	wait_queue_t *curr, *next;

	list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
		unsigned flags = curr->flags;

		if (curr->func(curr, mode, wake_flags, key) &&
				(flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
			break;
	}
}
```

这里看到在迭代等待队列的链表， 并且他有计数限制`nr_exclusive`;
每次迭代出一个`wait_queue_t`的块， 然后调用其唤醒函数， 该函数前面3.2.2讲过是统一设置成了`autoremove_wake_function()`；

都是调用统一的函数那它怎么知道，将哪个进程唤醒呢？

我们前面在讲3.2.1时候，构造`wait_queue_t`结构体的时候，已经将进程信息与该结构体捆绑， 所以这样就知道唤醒目标了; 


### 3.3.2 try_to_wake_up

3.2.2节已经分析过了唤醒函数最终是调用的`try_to_wake_up()`实现的进程唤醒，现在来详细看下:

```
static int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
	unsigned long flags;
	int cpu, success = 0;

	/*
	 * If we are going to wake up a thread waiting for CONDITION we
	 * need to ensure that CONDITION=1 done by the caller can not be
	 * reordered with p->state check below. This pairs with mb() in
	 * set_current_state() the waiting thread does.
	 */
	smp_mb__before_spinlock();
	raw_spin_lock_irqsave(&p->pi_lock, flags);          ①
	if (!(p->state & state))                            ②
		goto out;

	success = 1; /* we're going to change ->state */
	cpu = task_cpu(p);

	if (p->on_rq && ttwu_remote(p, wake_flags))         ③
		goto stat;

    ....

	ttwu_queue(p, cpu);                                 ④
stat:
	ttwu_stat(p, cpu, wake_flags);
out:
	raw_spin_unlock_irqrestore(&p->pi_lock, flags);

	return success;
}
```

> ①: 关闭中断  
> ②: 判断当前进程状态与我们要唤醒的状态是否统一

前面介绍过有可打断的和不可打断等调用，如果现在是不可打断的等待状态，而你用可打断方式去唤醒，是不可唤醒的；  
即你当初是调用`wait_event()`进入的休眠， 现在想调用`wake_up_interruptible()`进行唤醒， 这样是不可以的；

> ③: 判断该进程是否已经在等待队列，如果在，则设置进程状态为running后， 则跳转到stat；  
> ④: 详见3.3.3节 

### 3.3.3 ttwu_queue

```
static void ttwu_queue(struct task_struct *p, int cpu)
{
	struct rq *rq = cpu_rq(cpu);

#if defined(CONFIG_SMP)
	if (sched_feat(TTWU_QUEUE) && !cpus_share_cache(smp_processor_id(), cpu)) {
		sched_clock_cpu(cpu); /* sync clocks x-cpu */
		ttwu_queue_remote(p, cpu);
		return;
	}
#endif

	raw_spin_lock(&rq->lock);
	ttwu_do_activate(rq, p, 0);
	raw_spin_unlock(&rq->lock);
}
```


### 3.3.4 ttwu_do_activate

```
ttwu_do_activate(struct rq *rq, struct task_struct *p, int wake_flags)
{
#ifdef CONFIG_SMP
	if (p->sched_contributes_to_load)
		rq->nr_uninterruptible--;
#endif

	ttwu_activate(rq, p, ENQUEUE_WAKEUP | ENQUEUE_WAKING);    // 详见3.3.5 
	ttwu_do_wakeup(rq, p, wake_flags);                        // 将进程的状态设置为running 
}
```


### 3.3.5 ttwu_activate

```
static void ttwu_activate(struct rq *rq, struct task_struct *p, int en_flags)
{
	activate_task(rq, p, en_flags);                        // 将进程加入运行队列
	p->on_rq = TASK_ON_RQ_QUEUED;                          // 将进程的on_rq标志设置为TASK_ON_RQ_QUEUED

	/* if a worker is waking up, notify workqueue */
	if (p->flags & PF_WQ_WORKER)
		wq_worker_waking_up(p, cpu_of(rq));
}
```


这样就完成一次唤醒操作， 但是进程醒来以后还是需要判断下条件是否成立， 不成立则会继续进入休眠状态；  
在这里可以看到唤醒只是将其加入运行队列，让其可以运行，但是并没有将其从等待队列的链表中移除，只有才条件成立后，退出等待状态才会将其移除；  


PS: 具体的休眠和唤醒的流程可以参看3.2.4


# 附录. 用户空间进程与内核关系


当用户空间运行一个进程时， 其中进程中包含用户空间线程， 当与内核进行交互时， 还会有内核态线程；


# 附录. 进入休眠状态

![](/images/kernel/20200715101010810_1162712835.png)
