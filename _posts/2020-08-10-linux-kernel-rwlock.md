---
layout: post
title:  "详解Linux内核之自旋读写锁"
date:   2020-08-10
catalog:  true
author: Sourcelink
tags:
    - kernel
    - rwlock

---



# 一. 概述

> 内核版本: linux4.1.15  
> 宿主机: imx6ul  
> 架构:  armv7, arm32  

没有顺序没有优先级;  

读写锁功能:  

> 1. 当没有线程拥有写锁的时候, 允许多个线程去拥有读锁;  
> 2. 当有线程拥有写锁的时候, 其他线程需要等其释放锁后, 才可以去读或者写;  

# 二. 内核代码分析

```
typedef struct {
	arch_rwlock_t raw_lock;
} rwlock_t;
```

raw_lock是个32位的整形数据, 0～30的bit用来记录拥有读锁的数目，第31个bit用来记录拥有写锁的数目;  

![](/images/kernel/20200810180029901_203236627.png)

## 2.1 读写锁初始化

```
# define rwlock_init(lock)					\
do {								\
	static struct lock_class_key __key;			\
								\
	__rwlock_init((lock), #lock, &__key);			\
} while (0)
```


## 2.2 读锁

```
# define do_raw_read_lock(rwlock)	do {__acquire(lock); arch_read_lock(&(rwlock)->raw_lock); } while (0)
```


```
static inline void arch_read_lock(arch_rwlock_t *rw)
{
	unsigned long tmp, tmp2;

	prefetchw(&rw->lock);
	__asm__ __volatile__(
"1:	ldrex	%0, [%2]\n"                    // 将rw->lock的值装载到tmp中
"	adds	%0, %0, #1\n"                      // tmp中的值加1, tmp = tmp + 1
"	strexpl	%1, %0, [%2]\n"                // 如果tmp是正数或者零, 即非负值, 那么将tmp值存入rw->lock, 操作结果存放在tmp2中
	WFE("mi")                              // 如果tmp为负数则进入休眠
"	rsbpls	%0, %1, #0\n"                  // tmp = 0 - tmp2, tmp2等于0表示strexpl执行成功
"	bmi	1b"                                // 负数则跳转1处继续执行, 如果上条指令执行结果为负数, 说明strexpl未执行成功
	: "=&r" (tmp), "=&r" (tmp2)
	: "r" (&rw->lock)
	: "cc");

	smp_mb();
}
```

- 1) 函数开始会先读取lock中的值, 再进行加1操作, 表示有线程读, 后面接着判断其是否为负数, 如果是负数则说明此时有线程拥有写锁,
则不能进行拥有读锁操作, 即不能将前面加1操作的值更新到lock中;

- 2) 如果tmp为负数则进入休眠, 直到其他核释放锁的时候，通过硬件事件寄存器唤醒CPU;

- 3) 接着判断下前面strexpl是否成功执行, 没有成功执行则跳转1处继续(是从睡眠中被唤醒), 否则成功执行(顺序执行, 未进入休眠状态),读锁计数成功; 


如果执行到`smp_mb()`时, 调用者则成功获得读锁;  


ps: 其中指令`strexpl`的后缀pl意思是positive or zero, 即结果是正数或者0

## 2.3 写锁


```
void do_raw_write_lock(rwlock_t *lock)
{
	debug_write_lock_before(lock);
	arch_write_lock(&lock->raw_lock);
	debug_write_lock_after(lock);
}
```

```
static inline void arch_write_lock(arch_rwlock_t *rw)
{
	unsigned long tmp;

	prefetchw(&rw->lock);
	__asm__ __volatile__(
"1:	ldrex	%0, [%1]\n"            // 将rw->lock的值存入tmp
"	teq	%0, #0\n"                  // 判断tmp是否等于0
	WFE("ne")                      // 上条指令的执行结果, 不相等, 即tmp不等于0, 则进入休眠
"	strexeq	%0, %2, [%1]\n"        // 如果tmp等于0, 则执行rw->lock = 0x80000000
"	teq	%0, #0\n"                  // 判断上条指令执行是否成功
"	bne	1b"                        // 不相同表示不成功, 则跳转到1处继续执行
	: "=&r" (tmp)
	: "r" (&rw->lock), "r" (0x80000000)
	: "cc");

	smp_mb();
}
```

- 1) 函数开始会先读取lock中的值, 然后判断其值是否等于0,  不等于0表示现在有线程拥有读或写锁,   
则进入休眠状态, 等于0表示现在既没有线程拥有读锁也没有拥有写锁(不进入休眠);  

- 2) 判断tmp是否等于零才执行存储, 否则不存储, 只有为零, 才可以标记最高位为1, 即可以获取写锁;  

- 3) 最后判断`strexeq`操作是否成功, 不成功说明有其他线程在竞争, 则跳转1处继续执行, 否则成功获取写锁;  
