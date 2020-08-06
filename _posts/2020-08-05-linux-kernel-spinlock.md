---
layout: post
title:  "详解Linux内核之自旋锁"
date:   2020-08-05
catalog:  true
tags:
    - kernel
    - spin_lock

---

# 一. 概述

> 内核版本: linux4.1.15  
> 主机: ubuntu16.04  

自旋锁是为了防止多处理器并发而引入的一种锁，它在内核中大量应用于中断处理等部分。简单的说，它允许CPU执行一个死循环等待锁，直到锁被其他CPU释放;

打个比方: 

> 自旋锁就好比一个外卖店维护两个账本， 一个账本是记录了今天需要送的外卖的总数， 另一个账本记录了已经送了外卖的总数；  
> 当有新的订单来了则要送外卖总数账本加1, 当送完一笔订单， 则已送的账本计数加1；  
> 所以不难看出， 这家外卖店在什么时候空闲下来（当今天需要送的外卖都送完），即已经送的外卖数等于需要送的外卖数；  



我们来看下自旋锁在内核中的定义：

```
typedef struct {
	union {
		u32 slock;
		struct __raw_tickets {
#ifdef __ARMEB__                            // 大端字节序
			u16 next;
			u16 owner;
#else                                                // 小端字节序
			u16 owner;
			u16 next;
#endif
		} tickets;
	};
} arch_spinlock_t;
```

以小端字节序为例，next就好比外卖进行需要送的外卖总数， 而owner就好比已经送的外卖总数； 
当next与owenr数值相同时，则自旋锁是空闲状态， 反之锁是忙碌状态；

再来捋捋：  
当有线程需要获取该锁的时候，则将next计数加1, 然后判断当前锁是否被占有，即判断**owenr与next计数加1之前**是否相同，相同则表示空闲，成功获取锁，  
当释放该锁的时候，只要将owenr数值加1；  


# 二. 内核代码分析

## 2.1 定义一个自旋锁

```
static arch_spinlock_t die_lock = __ARCH_SPIN_LOCK_UNLOCKED;
```

以上是一个定义自旋锁的例子， 其中`__ARCH_SPIN_LOCK_UNLOCKED`定义如下：

```
#define __ARCH_SPIN_LOCK_UNLOCKED	{ { 0 } }
```

从锁被创建的时候next与owenr值是相同的， 即空闲状态；  

## 2.2 获取锁

> arch/arm/include/asm/spinlock.h

```
static inline void arch_spin_lock(arch_spinlock_t *lock)
{
	unsigned long tmp;
	u32 newval;
	arch_spinlock_t lockval;

	prefetchw(&lock->slock);
	__asm__ __volatile__(
"1:	ldrex	%0, [%3]\n"                                //  将lock->slock赋值给lockval
"	add	%1, %0, %4\n"                                  //   newval = localval + （1 << TICKET_SHIFT）
"	strex	%2, %1, [%3]\n"                            //  将newval赋值给lock->slock, 赋值成功将tmp清零
"	teq	%2, #0\n"                                      //  判断tmp是否等于0
"	bne	1b"                                            //  不相等，则跳转至1处继续执行
	: "=&r" (lockval), "=&r" (newval), "=&r" (tmp)
	: "r" (&lock->slock), "I" (1 << TICKET_SHIFT)
    : "cc");

	while (lockval.tickets.next != lockval.tickets.owner) {   // 判断原来锁中的next与owenr字段是否相同
		wfe();                                                // 不相同， 则表示有人在使用该锁，则进入休眠状态
		lockval.tickets.owner = ACCESS_ONCE(lock->tickets.owner);  // 被唤醒后， 从内存中重新读取owner值 
	}

	smp_mb();
}
```


首先看这段代码：

```
"	add	%1, %0, %4\n" 
```

其实 `localval + （1 << TICKET_SHIFT）`相当于给自旋锁其成员`next`执行加1操作；

为什么这样做可以将lockval.tickets.next的值加1？

根据其数据结构定义，以小端字节序为例， 成员`slock`与`tickets`是共用一块内存的, 示意图如下：

![](/images/kernel/20200804224758126_1989376105.png)

注意所加的值是`1 << TICKET_SHIFT`, 而`TICKET_SHIFT`值位16, 不就是刚好是next变量的第0位吗, 所以最后是加变相给`lockval.tickets.next`的值加1；  


再看这段代码:  

```
while (lockval.tickets.next != lockval.tickets.owner)
```

如果在执行计数加1前next与owner相同的时候就不需要进入休眠状态，即成功拿到了锁；

**PS**: 每个锁的数据结构arch_spinlock_t中维护两个字段：next和owner，只有当next和owner相等时才能获取锁;  

## 2.3 释放锁

> arch/arm/include/asm/spinlock.h

```
static inline void arch_spin_unlock(arch_spinlock_t *lock)
{
	smp_mb();
	lock->tickets.owner++;                //    owener数值加1
	dsb_sev();
}
```

**PS**: 当进程在释放锁的时候owner值会增加;  

# 三. arm64获取锁解析

```
static inline void arch_spin_lock(arch_spinlock_t *lock)
{
	unsigned int tmp;
	arch_spinlock_t lockval, newval;

	asm volatile(
	/* Atomically increment the next ticket. */
"	prfm	pstl1strm, %3\n"                  // 预取自旋锁的值到缓存行中，以加快后续内存访问指令的执行
"1:	ldaxr	%w0, %3\n"                    // 加载lock的值到lockval中, 其中w表示32位宽度
"	add	%w1, %w0, %w5\n"                  // newval = lockval + (1 << TICKET_SHIFT)
"	stxr	 %w2, %w1, %3\n"              // 将newval的值存放到lock中, 操作结果放在tmp中, 0:表示存储成功, 1:表示失败
"	cbnz	 %w2, 1b\n"                   // 如果存储失败, 则跳转跳转到1处继续执行, 即这是个原子操作
	/* Did we get the lock? */
"	eor	%w1, %w0, %w0, ror #16\n"         // lockval ^ (lockval >> 16) == lockval.tickets.owenr ^ lockval.tickets.next
                                          // “%w0, ror #16” 代表将 %w0 循环右移 16 位，即颠倒了 owner 和 next 位置
"	cbz	%w1, 3f\n"                        // 上条指令异或的结果存放在tmp中的, 判断它是否为0, 0则跳转到3处, 即成功获取锁
"	sevl\n"                               // 否则发送一条本地事件指令
"2:	wfe\n"                                // 并进入休眠状态
"	ldaxrh	%w2, %4\n"                    // 休眠被唤醒后会执行该指令, 重新读取lock的值到tmp中
"	eor	%w1, %w2, %w0, lsr #16\n"         // lock->tickets.owenr ^ lockval.tickets.next, 异或结果存在tmp中
                                          // “%w0, ror #16” 代表将 %w0 逻辑右移 16 位，next 向右移动16位, 高位补零
"	cbnz	 %w1, 2b\n"                       // 不等于0, 则跳转到2处执行, 即自旋
	/* We got the lock. Critical section starts here. */
"3:"
	: "=&r" (lockval), "=&r" (newval), "=&r" (tmp), "+Q" (*lock)
	: "Q" (lock->owner), "I" (1 << TICKET_SHIFT)
	: "memory");
}
```
 

SEV
发送事件（Send Event）指令。

这条指令将想系统中的所有CPU核发送事件。对应系统中的每个CPU核，设置事件寄存器（Event Register）相应的位。如果某个CPU核正在等待事件（WFE），那么该CPU核会被立即唤醒，并清除掉表示该CPU的事件寄存器相应的位。  

SEVL
发送本地事件（Send Event Locally）指令。

不同于前面说的SEV指令，这条指令只会向当前CPU核心发送。如果是多核CPU那也只向当前核心，不会向CPU内的其它核心发送。  
值得注意的是，这条指令只有在支持ARMv8指令集之后的处理器中才有效。

# 附录. 自旋锁调用层次

以下是内核中使用自旋锁的一段代码:  

```
		spin_lock(&pool->lock);

		if (tags->nr_free == pool->percpu_max_size) {
			move_tags(pool->freelist, &pool->nr_free,
				  tags->freelist, &tags->nr_free,
				  pool->percpu_batch_size);

			wake_up(&pool->wait);
		}
		spin_unlock(&pool->lock);
```

现在来看`spin_lock()`的调用原型：

> include/linux/spinlock.h

```
static inline void spin_lock(spinlock_t *lock)
{
	raw_spin_lock(&lock->rlock);
}
```

其中调用`raw_spin_lock()`函数实现，但是追踪代码后，发现其调用层次有点深， 最后才调用`arch_spin_lock()`,   
接下来记录下其调用层次：  

```
#define raw_spin_lock(lock)	_raw_spin_lock(lock)                        // include/linux/spinlock.h
    void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)                // kernel/locking/spinlock.c
        static inline void __raw_spin_lock(raw_spinlock_t *lock)        // include/linux/spinlock_api_smp.h
```

```
static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
	preempt_disable();
	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

只要看这段代码: 

```
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
```

这是一个宏定义，其原型如下：

```
#define LOCK_CONTENDED(_lock, try, lock)			\
do {								\
	if (!try(_lock)) {					\                           
		lock_contended(&(_lock)->dep_map, _RET_IP_);	\
		lock(_lock);					\
	}							\
	lock_acquired(&(_lock)->dep_map, _RET_IP_);			\
} while (0)
```

带入以后展开如下：


```
do {
	if (!do_raw_spin_trylock(_lock)) {                                  // 会先尝试获取锁，如果获取成功 则返回1, 获取失败则返回0
		lock_contended(&(_lock)->dep_map, _RET_IP_);
		do_raw_spin_lock(_lock);                                        // 获取锁， 获取不成功将自旋
	}
	lock_acquired(&(_lock)->dep_map, _RET_IP_);	
} while (0)
```

其中`do_raw_spin_trylock()`是通过调用`arch_spin_trylock()`实现（该函数详细解析看后面），`do_raw_spin_lock`则是通过调用`arch_spin_lock()`实现；

所以在这个宏中会去获取锁，最终的调用就在这停止了；  

## arch_spin_trylock

```
static inline int arch_spin_trylock(arch_spinlock_t *lock)
{
	unsigned long contended, res;
	u32 slock;

	prefetchw(&lock->slock);
	do {
		__asm__ __volatile__(
		"	ldrex	%0, [%3]\n"                // 讲lock->slock的值装载到临时变量slock中
		"	mov	%2, #0\n"                      // 将res清零
		"	subs	%1, %0, %0, ror #16\n"     // slock右循环16位， 即next在低16位，owenr在高16位， 然后与原来没有进行移位操作的slock想减， 其实就是next - owenr 或者说 owenr - next
		                                       // 会将想减结果放在res中，如果有进位会设置cpsr，下面两条指令是否执行会依赖该条指令执行结果
		"	addeq	%0, %0, %4\n"              // next == owenr 则给slock中的next加1
		"	strexeq	%2, %0, [%3]"              // next == owenr 则将slock的值存储回lock->slock， 操作结果存放到res中， 0表示操作成功
		: "=&r" (slock), "=&r" (contended), "=&r" (res)
		: "r" (&lock->slock), "I" (1 << TICKET_SHIFT)
		: "cc");
	} while (res);                             // res位零则退出， 或者是strexeq操作不成功则重新执行

	if (!contended) {                          // 根据前面代码分析，contended等于0 表示现在锁空闲， 1表示锁被占有
		smp_mb();
		return 1;
	} else {
		return 0;
	}
}
```



