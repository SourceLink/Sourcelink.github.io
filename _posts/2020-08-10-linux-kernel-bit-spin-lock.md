---
layout: post
title:  "详解Linux内核之自旋位锁"
date:   2020-03-10
catalog:  true
author: Sourcelink
tags:
    - kernel
    - bit_spinlock

---



# 一. 概述

自旋位锁与自旋锁有类似的地方：均是多个CPU竞争同一把锁，如果不成功，就通过自旋的方式，轮询锁状态直到成功获得锁为止。  
但在一个自旋锁中，使用32位的字段表示锁的状态。而在自旋位锁中，每一个unsigned long类型的字段中，表示32/64个位锁。其中每一位表示一个锁的状态。  


# 二. 内核代码详解

> include/linux/bit_spinlock.h


## 2.1 bit_spin_lock

```
static inline void bit_spin_lock(int bitnum, unsigned long *addr)
{
	preempt_disable();                                                 // 关闭抢占功能
#if defined(CONFIG_SMP) || defined(CONFIG_DEBUG_SPINLOCK)
	while (unlikely(test_and_set_bit_lock(bitnum, addr))) {            // 判断锁对应的bitnum位是否已经置1, 并设置成1
		preempt_enable();                                              // 如果被置1说明该锁已经被其他线程占有, 则开启抢占
		do {
			cpu_relax();                                               // 内存屏障或休眠等操作 
		} while (test_bit(bitnum, addr));                              // 继续判断位锁是否被占有, 占有则自旋
		preempt_disable();                                             // 关闭抢占
	}
#endif
	__acquire(bitlock);                                                // 运行到这说明已经顺利占有位锁
}
```


## 2.2 bit_spin_unlock


```
static inline void bit_spin_unlock(int bitnum, unsigned long *addr)
{
#if defined(CONFIG_SMP) || defined(CONFIG_DEBUG_SPINLOCK)
	clear_bit_unlock(bitnum, addr);                                     // 清除对应的位
#endif
	preempt_enable();                                                   // 使能抢占
	__release(bitlock);
}
```


**总结:** 当对应的位锁被占用后, 都会将其位置高, 释放的时候会将其清零;