---
layout: post
title:  "详解Linux内核之位操作"
date:   2020-08-04
catalog:  true
author: Sourcelink
tags:
    - kernel
    - bit

---

[TOC]


# 一. 概述

> 内核版本: linux4.1.15  
> 宿主机: imx6ul  
> 主机: ubuntu16.04  

在Linux内核中大量使用了位操作, 这样可以减少一定的内存, 一些关于位操作的宏如下:  

```
#define set_bit(nr,p)			ATOMIC_BITOP(set_bit,nr,p)
#define clear_bit(nr,p)			ATOMIC_BITOP(clear_bit,nr,p)
#define change_bit(nr,p)		ATOMIC_BITOP(change_bit,nr,p)
#define test_and_set_bit(nr,p)		ATOMIC_BITOP(test_and_set_bit,nr,p)
#define test_and_clear_bit(nr,p)	ATOMIC_BITOP(test_and_clear_bit,nr,p)
#define test_and_change_bit(nr,p)	ATOMIC_BITOP(test_and_change_bit,nr,p)
```


假如你想将一个变量的第2位置1, 你可以这样做:  

```
	set_bit(2, &test_bit);
```

现在看下`ATOMIC_BITOP`的定义:  

```
#ifndef CONFIG_SMP
#define ATOMIC_BITOP(name,nr,p)			\
	(__builtin_constant_p(nr) ? ____atomic_##name(nr, p) : _##name(nr,p))
#else
#define ATOMIC_BITOP(name,nr,p)		_##name(nr,p)
#endif
```

因为我的板子是单核的, 所以使用这条宏:  

```
#define ATOMIC_BITOP(name,nr,p)		_##name(nr,p)
```

接下来进行代码分析;

# 二. 非原子操作


`set_bit()`应该对应`_set_bit()`: 

## 2.1 __set_bit

> include/asm-generic/bitops/non-atomic.h

```
static inline void __set_bit(int nr, volatile unsigned long *addr)
{
	unsigned long mask = BIT_MASK(nr);                    // 将1向左移动nr位
	unsigned long *p = ((unsigned long *)addr) + BIT_WORD(nr);  // 从新计算地址, 如果设置的位已经超过了一个word的长度, 地址需要移位

	*p  |= mask;                                          // 再设置到变量上
}

```

显而易见这个函数是在没有任何保护的情况操作的, 是一个非原子操作;  

## 2.2 __clear_bit

```
static inline void __clear_bit(int nr, volatile unsigned long *addr)
{
	unsigned long mask = BIT_MASK(nr);
	unsigned long *p = ((unsigned long *)addr) + BIT_WORD(nr);

	*p &= ~mask;
}
```

和前面置位的操作相同, 只是与上了相反数进行了清除操作;  

## 2.3 test_bit

```
static inline int test_bit(int nr, const volatile unsigned long *addr)
{
	return 1UL & (addr[BIT_WORD(nr)] >> (nr & (BITS_PER_LONG-1)));
}
```

将对应变量向右移动nr位再与1相与, 如果为真表示该位已经被置1;  

其他的就不一一解释了, 接下来看下原子下的位操作


# 三. 原子操作

> arch/arm/include/asm/bitops.h

## 3.1 ____atomic_set_bit

```
static inline void ____atomic_set_bit(unsigned int bit, volatile unsigned long *p)
{
	unsigned long flags;
	unsigned long mask = 1UL << (bit & 31); // 类似去余数

	p += bit >> 5;                          // 计算要设置变量的地址, 相当于除32位

	raw_local_irq_save(flags);              // 保存当前irq状态, 并关闭irq
	*p |= mask;                             // 对相应的位置1
	raw_local_irq_restore(flags);
}
```

假设现在需要将一个变量的第33位置, 则mask计算出是2, p则需要进行地址加1操作;  

这里的原子操作是在关闭中断实现的;  

## 3.2 ____atomic_clear_bit

```
static inline void ____atomic_clear_bit(unsigned int bit, volatile unsigned long *p)
{
	unsigned long flags;
	unsigned long mask = 1UL << (bit & 31);

	p += bit >> 5;

	raw_local_irq_save(flags);
	*p &= ~mask;
	raw_local_irq_restore(flags);
}
```



# 附录. 汇编版本

> arch/arm/kernel/armksyms.c

```
EXPORT_SYMBOL(_set_bit);
EXPORT_SYMBOL(_test_and_set_bit);
EXPORT_SYMBOL(_clear_bit);
EXPORT_SYMBOL(_test_and_clear_bit);
EXPORT_SYMBOL(_change_bit);
EXPORT_SYMBOL(_test_and_change_bit);
EXPORT_SYMBOL(_find_first_zero_bit_le);
EXPORT_SYMBOL(_find_next_zero_bit_le);
EXPORT_SYMBOL(_find_first_bit_le);
EXPORT_SYMBOL(_find_next_bit_le);

```

> arch/arm/include/asm/sync_bitops.h

```
#define sync_set_bit(nr, p)		_set_bit(nr, p)
#define sync_clear_bit(nr, p)		_clear_bit(nr, p)
#define sync_change_bit(nr, p)		_change_bit(nr, p)
#define sync_test_and_set_bit(nr, p)	_test_and_set_bit(nr, p)
#define sync_test_and_clear_bit(nr, p)	_test_and_clear_bit(nr, p)
#define sync_test_and_change_bit(nr, p)	_test_and_change_bit(nr, p)
#define sync_test_bit(nr, addr)		test_bit(nr, addr)
#define sync_cmpxchg			cmpxchg
```

> arch/arm/lib/bitops.h

```
	.macro	bitop, name, instr
ENTRY(	\name		)
UNWIND(	.fnstart	)
	ands	ip, r1, #3
	strneb	r1, [ip]		@ assert word-aligned
	mov	r2, #1
	and	r3, r0, #31		@ Get bit offset
	mov	r0, r0, lsr #5
	add	r1, r1, r0, lsl #2	@ Get word offset
#if __LINUX_ARM_ARCH__ >= 7 && defined(CONFIG_SMP)
	.arch_extension	mp
	ALT_SMP(W(pldw)	[r1])
	ALT_UP(W(nop))
#endif
	mov	r3, r2, lsl r3
1:	ldrex	r2, [r1]
	\instr	r2, r2, r3
	strex	r0, r2, [r1]
	cmp	r0, #0
	bne	1b
	bx	lr
UNWIND(	.fnend		)
ENDPROC(\name		)
	.endm
```

`.macro`类似c语言的`#define`, 举个例子:  

```
bitop	_set_bit, orr
```

展开后:  

```
ENTRY(_set_bit)
UNWIND(.fnstart)
	ands	ip, r1, #3
	strneb	r1, [ip]		@ assert word-aligned
	mov	r2, #1
	and	r3, r0, #31		@ Get bit offset
	mov	r0, r0, lsr #5
	add	r1, r1, r0, lsl #2	@ Get word offset
#if __LINUX_ARM_ARCH__ >= 7 && defined(CONFIG_SMP)
	.arch_extension	mp
	ALT_SMP(W(pldw)	[r1])
	ALT_UP(W(nop))
#endif
	mov	r3, r2, lsl r3
1:	ldrex	r2, [r1]
	orr	r2, r2, r3
	strex	r0, r2, [r1]
	cmp	r0, #0
	bne	1b
	bx	lr
UNWIND(.fnend)
ENDPROC(_set_bit)
```

这个汇编函数和原子版位操作是一样的, 只是用汇编语言直接实现了; 


