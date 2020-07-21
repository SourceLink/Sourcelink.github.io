---
layout: post
title:  "Linux内核之中断下半部"
date:   2020-07-21
catalog:  true
author: Sourcelink
tags:
    - kernel
    - tasklet

---



# 一. 概述


在处理当前中断时，即使发生了其他中断，其他中断也不会得到处理，所以中断的处理要越快越好。
但是某些中断要做的事情稍微耗时，这时可以把中断拆分为上半部、下半部。

- 在上半部处理紧急的事情，在上半部的处理过程中，中断是被禁止的；
- 在下半部处理耗时的事情，在下半部的处理过程中，中断是使能的；

以按键中断为例， 使用`requst_irq()`注册一个硬件中断服务（中断上半部）；


```
	request_irq(virq,  buttons_irq, 0, "B1", &data);
```


其中buttons_irq为注册的中断服务函数， 大概内容如下：

```
static irqreturn_t buttons_irq(int irq, void *dev_id)
{
    // 做信息登记，等快速处理工作
    // 启动中断下半部，在下半部中，做复杂的耗时操作
	return IRQ_RETVAL(IRQ_HANDLED);
}
```

# 二. 使用

这篇笔记主要以`tasklet`来实现中断下半部进行讲解， 并分析其内核实现部分；


## 2.1 初始化tasklet

tasklet初始化有两种方式:

- 静态声明

```
#define DECLARE_TASKLET(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }、
```

- 动态声明

```
extern void tasklet_init(struct tasklet_struct *t,
			 void (*func)(unsigned long), unsigned long data);
```

## 2.2 调度tasklet

```
static inline void tasklet_schedule(struct tasklet_struct *t)
```

## 2.3 卸载tasklet

```
extern void tasklet_kill(struct tasklet_struct *t);
```


## 2.4 使能/禁止tasklet

```
static inline void tasklet_enable(struct tasklet_struct *t)
static inline void tasklet_disable(struct tasklet_struct *t)
```

## 2.5 使用例子


```
void test_tasklet_action(unsigned long t);

DECLARE_TASKLET(test_tasklet, test_tasklet_action, 0);        // 先静态声明test_tasklet，并注册了服务函数

void test_tasklet_action(unsigned long t)
{
	printk("tasklet is executing\n");
}

...


static irqreturn_t buttons_irq(int irq, void *dev_id)
{
    ...

	// 调度tasklet执行
	tasklet_schedule(&test_tasklet);

	return IRQ_RETVAL(IRQ_HANDLED);
}
```


总结：先定义tasklet，需要使用时调用tasklet_schedule，驱动卸载前调用tasklet_kill。  
tasklet_schedule 只是把tasklet放入内核队列，它注册服务函数会在软件中断的执行过程中被调用；


# 三. 内核实现分析


## 3.1 tasklet_init


```
void tasklet_init(struct tasklet_struct *t,
		  void (*func)(unsigned long), unsigned long data)
{
	t->next = NULL;                 // 用于挂载链表
	t->state = 0;                   // 这个tasklet有两种状态；
	atomic_set(&t->count, 0);       // 设置计数为0表示使能， 非零表示禁止
	t->func = func;                 // 注册服务函数
	t->data = data;                 // 服务函数的形参
}
```

tasklet的state两种状态：

```
enum
{
	TASKLET_STATE_SCHED,	/* Tasklet is scheduled for execution */
	TASKLET_STATE_RUN	/* Tasklet is running (SMP only) */
};
```

## 3.2 tasklet_schedule

```
static inline void tasklet_schedule(struct tasklet_struct *t)
{
	if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))        // 如果已经是调度状态，则不执行下面的函数，否则设置状态为调度状态并执行下面函数
		__tasklet_schedule(t);                        // 执行调度
}
```

test_and_set_bit是个原子位操作，这个state变量使用了前两位用于存储状态值，

- TASKLET_STATE_SCHED

表示第0状态位,　当这位为1的时候表示该tasklet已经放入调度链表中了；

- TASKLET_STATE_RUN

表示第1状态位,　当这位为1的时候表示该tasklet的func正在执行中；


### 3.2.1 __tasklet_schedule


```
void __tasklet_schedule(struct tasklet_struct *t)
{
	unsigned long flags;

	local_irq_save(flags);
	t->next = NULL;
	*__this_cpu_read(tasklet_vec.tail) = t;
	__this_cpu_write(tasklet_vec.tail, &(t->next));       // 将tasklet加入队列
	raise_softirq_irqoff(TASKLET_SOFTIRQ);                // 并触发TASKLET软中断
	local_irq_restore(flags);
}
```

所以从这里可以看出中断下半部的tasklet方法，是通过软中断实现的；

总结：重新缕下思路，在驱动设备在open的时候或者加载的时候需要先定义一个tasklet变量并且注册了对应的服务函数，
在硬件中断服务函数被调用的时候（中断上半部），调用`tasklet_schedule()`触发tasklet调度，即将tasklet挂载到`tasklet_vec`链表上，并触发了`TASKLET_SOFTIRQ`软中断；

猜测：在`TASKLET_SOFTIRQ`软中断服务函数中，会依次取出`tasklet_vec`链表山的tasklet，并执行其服务函数`func`；

## 3.3 softirq_init

现在看下`TASKLET_SOFTIRQ`对应的 服务函数在哪;

```
void __init softirq_init(void)
{
	int cpu;

	for_each_possible_cpu(cpu) {
		per_cpu(tasklet_vec, cpu).tail =
			&per_cpu(tasklet_vec, cpu).head;
		per_cpu(tasklet_hi_vec, cpu).tail =
			&per_cpu(tasklet_hi_vec, cpu).head;
	}

	open_softirq(TASKLET_SOFTIRQ, tasklet_action);            // 注册的服务函数
	open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```

### 3.3.1 tasklet_action

```
static void tasklet_action(struct softirq_action *a)
{
	struct tasklet_struct *list;

	local_irq_disable();
	list = __this_cpu_read(tasklet_vec.head);         // 先对tasklet_vec链表头进行设置
	__this_cpu_write(tasklet_vec.head, NULL);
	__this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
	local_irq_enable();

	while (list) {
		struct tasklet_struct *t = list;

		list = list->next;                            // 从链表中取出一个tasklet

		if (tasklet_trylock(t)) {                     // 详见 3.3.2
			if (!atomic_read(&t->count)) {            // 判断这个tasklet是否使能，count值为0表示使能，非零表示禁止
				if (!test_and_clear_bit(TASKLET_STATE_SCHED,
							&t->state))
					BUG();
				t->func(t->data);                     // 执行tasklet中的func函数并传递形参data
				tasklet_unlock(t);
				continue;
			}
			tasklet_unlock(t);
		}

		local_irq_disable();
		t->next = NULL;                                // 执行完func后将该tasklet从链表中移除
		*__this_cpu_read(tasklet_vec.tail) = t;
		__this_cpu_write(tasklet_vec.tail, &(t->next));
		__raise_softirq_irqoff(TASKLET_SOFTIRQ);
		local_irq_enable();
	}
}
```

### 3.3.2 tasklet_trylock

```
static inline int tasklet_trylock(struct tasklet_struct *t)
{
	return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
}
```

将要执行func的时候，会先将state上的TASKLET_STATE_RUN位置1，表示该tasklet正在执行中；

### 3.3.3 tasklet_unlock

```
static inline void tasklet_unlock(struct tasklet_struct *t)
{
	smp_mb__before_atomic();
	clear_bit(TASKLET_STATE_RUN, &(t)->state);
}
```

在执行完func后，又会将其run标志清除；


## 3.4 总结

没调用一次`tasklet_schedule()`后，将tasklet挂载到`tasklet_vec`链表上， 并触发一次`TASKLET_SOFTIRQ`软中断，  
则对应的会执行一次`tasklet_action()`服务函数， 在这里面它将从`tasklet_vec`链表上将tasklet取出，经过一系列判断，  
如果成功则执行其func函数，并在执行完毕后，将该tasklet从链表上移除；



这里重点介绍tasklet的实现过程，软中断的调度实现这里就不讲解了；