---
layout: post
title:  "【从零开始写RTOS】优先级就绪队列"
date:   2019-12-24
catalog:  true
author: Sourcelink
tags:
    - RTOS
---

# 一. 最高优先级获取

> 策略： 每次需要调度最高优先级就绪进程运行;

所以, 需要解决查找最高优先级进程的问题;  


> find first bit in word

```
static __always_inline unsigned long __ffs(unsigned long word)
{
	int num = 0;

#if BITS_PER_LONG == 64
	if ((word & 0xffffffff) == 0) {
		num += 32;
		word >>= 32;
	}
#endif
	if ((word & 0xffff) == 0) {
		num += 16;
		word >>= 16;
	}
	if ((word & 0xff) == 0) {
		num += 8;
		word >>= 8;
	}
	if ((word & 0xf) == 0) {
		num += 4;
		word >>= 4;
	}
	if ((word & 0x3) == 0) {
		num += 2;
		word >>= 2;
	}
	if ((word & 0x1) == 0)
		num += 1;
	return num;
}
```

**PS：优先级越高对应的数值越小**


# 二. 规划就绪队列


> 1. 需要一个数组来存储优先级的bitmap

```

static unsigned int find_first_bit(const unsigned int *_bitmap)
{
    unsigned int index = 0;
    unsigned int i;
    unsigned int bitmask = 0;

    for (i = 0; i < KOS_PRIO_TABLE_SIZE; i++) {
        if (_bitmap[i]) {
            bitmask = 1;
            break;
        }
    }

    if (bitmask) {
        index = KOS_PRIO_SLOT_SIZE * i;
    }

    return (__find_first_bit(_bitmap[i]) + index);
}
```

> 2. 每个优先级进程都会挂载到链表上

> 3. 最高优先级是多少


数据结构：

```
struct kos_ready_queue {
    unsigned int prio_bitmap[KOS_PRIO_TABLE_SIZE];
    struct list_head queue[KOS_READ_LIST_SIZE];
    unsigned int highest_prio;
};
```

# 三. 业务

## 3.1 初始化


初始化三部曲：

> 1. 初始化highest_prio  
> 2. 初始化链表  
> 3. 初始化bitmap  

```
void kos_rq_init(void)
{
    unsigned int i;

    // 1. 初始化highest_prio
    kos_rq.highest_prio = KOS_CONFIG_LOWEST_PRIO;
    // 2. 初始化链表
    for (i = 0; i < KOS_READ_LIST_SIZE; i++) {
        list_head_init(&kos_rq.queue[i]);
    }
    // 3. 初始化bitmap
    for (i = 0; i < KOS_PRIO_TABLE_SIZE; i++) {
        kos_rq.prio_bitmap[i] = 0;
    }
}
```

## 3.2 登记

- 主要是对bitmap的操作

```
static void __register_prio(unsigned int _prio)
{
    kos_rq.prio_bitmap[KOS_PRIO_OFFSET(_prio)] |= KOS_PRIO_BIT(_prio);
}
```

- 更新最高优先级

```
void register_prio(unsigned int _prio)
{
    __register_prio(_prio);
    if (_prio < kos_rq.highest_prio) {
        kos_rq.highest_prio = _prio;
    }
}
```


## 3.3 注销

```
static void __unsigned_prio(unsigned int _prio)
{
    kos_rq.prio_bitmap[KOS_PRIO_OFFSET(_prio)] &= ~KOS_PRIO_BIT(_prio);
}
```

# 四. 队列


## 4.1 添加就绪队列

```
void kos_rq_add_head(struct kos_proc *_proc)
{
    unsigned short prio = _proc->priority;
    struct list_head *rq_list = &kos_rq.queue[prio];

    if (list_empty(rq_list)) {
        register_prio(prio);
    }

    list_add(rq_list, &_proc->slot_list);
}


void kos_rq_add_tail(struct kos_proc *_proc)
{
    unsigned short prio = _proc->priority;
    struct list_head *rq_list = &kos_rq.queue[prio];

    if (list_empty(rq_list)) {
        register_prio(prio);
    }

    list_add_tail(rq_list, &_proc->slot_list);
}
```

## 4.2 删除就绪队列


```
unsigned int kos_rq_highest_prio(void)
{
    return find_first_bit(kos_rq.prio_bitmap) + 1;
}


void kos_rq_delete(struct kos_proc *_proc)
{
    unsigned int prio = _proc->priority;
    struct list_head *rq_list = &kos_rq.queue[prio]; 

    list_delete_init(&_proc->slot_list);

    if (list_empty(rq_list)) {
        __unsigned_prio(prio);
    }

    if (prio == kos_rq.highest_prio) {
       kos_rq.highest_prio =  kos_rq_highest_prio();
    }
}
```

## 4.3  获取最高优先级进程

```

struct kos_proc *kos_rq_highest_ready_proc(void)
{
    struct kos_proc *rproc = NULL;

    struct list_head *node = kos_rq.queue[kos_rq.highest_prio].next;

    rproc = kos_list_entry(node, struct kos_proc, slot_list);

    return rproc;
}
```

