---
layout: post
title:  "详解Linux内核之原子计数"
date:   2020-08-03
catalog:  true
author: Sourcelink
tags:
    - kernel
    - atomic

---



# 一. 概述


> 宿主机: imx6ul  
> 内核版本: linux-4.1.15  


在多核同步的众多手段中，原子操作可以说是最基础的，但需要注意的是，单核系统(UP)同样需要原子操作，只不过多核系统(SMP)要比单核系统中的原子操作面临更多的问题。“原子(atom)”一词来自希腊语，意思是“不可分割(indivisible)”。当然，现代物理学中所说的“原子”并非是不可分割的。

```
static inline void atomic_add(int i, atomic_t *v)
{
	atomic_add_return(i, v);
}

static inline void atomic_sub(int i, atomic_t *v)
{
	atomic_sub_return(i, v);
}

static inline void atomic_inc(atomic_t *v)
{
	atomic_add_return(1, v);
}

static inline void atomic_dec(atomic_t *v)
{
	atomic_sub_return(1, v);
}
```

其中`atomic_t`的定义如下:  

```
typedef struct {
	int counter;
} atomic_t;
```


# 二. 內核实现

在内核是无法直接找到`atomic_add_return()`, 因为它是同一个宏定义进行声明；

> arch/arm/include/asm/atomic.h

```
#define ATOMIC_OPS(op, c_op, asm_op)                                           \
	ATOMIC_OP(op, c_op, asm_op)                                            \
	ATOMIC_OP_RETURN(op, c_op, asm_op)

ATOMIC_OPS(add, +=, add)
ATOMIC_OPS(sub, -=, sub)
```

```
#define ATOMIC_OP(op, c_op, asm_op)					\
static inline void atomic_##op(int i, atomic_t *v)			\
{									\
	unsigned long tmp;						\
	int result;							\
									\
	prefetchw(&v->counter);						\
	__asm__ __volatile__("@ atomic_" #op "\n"			\
"1:	ldrex	%0, [%3]\n"						\
"	" #asm_op "	%0, %0, %4\n"					\
"	strex	%1, %0, [%3]\n"						\
"	teq	%1, #0\n"						\
"	bne	1b"							\
	: "=&r" (result), "=&r" (tmp), "+Qo" (v->counter)		\
	: "r" (&v->counter), "Ir" (i)					\
	: "cc");							\
}									\
```


```
#define ATOMIC_OP_RETURN(op, c_op, asm_op)				\
static inline int atomic_##op##_return(int i, atomic_t *v)		\
{									\
	unsigned long tmp;						\
	int result;							\
									\
	smp_mb();							\
	prefetchw(&v->counter);						\
									\
	__asm__ __volatile__("@ atomic_" #op "_return\n"		\
"1:	ldrex	%0, [%3]\n"						\
"	" #asm_op "	%0, %0, %4\n"					\
"	strex	%1, %0, [%3]\n"						\
"	teq	%1, #0\n"						\
"	bne	1b"							\
	: "=&r" (result), "=&r" (tmp), "+Qo" (v->counter)		\
	: "r" (&v->counter), "Ir" (i)					\
	: "cc");							\
									\
	smp_mb();							\
									\
	return result;							\
}
```

## 2.1 atomic_add

将`ATOMIC_OPS(add, +=, add)`展开，如下：

```
static inline void atomic_add(int i, atomic_t *v)
{
	unsigned long tmp;
	int result;

	prefetchw(&v->counter);
	__asm__ __volatile__("@ atomic_add \n"
"1:	ldrex	%0, [%3]\n"                        // 先将v->counter中的计数装载到result中
"	add 	%0, %0, %4\n"                          // result加上数值i
"	strex	%1, %0, [%3]\n"                    // 将result的值存放到v->counter, 并将操作结果存放在tmp中
"	teq	%1, #0\n"                              // 判断tmp是否为0,如果为1表示未写入成功, 为0表示成功
"	bne	1b"                                    // 如果上面结果不为真, 即tmp == 1, 则跳转 1处继续执行
	: "=&r" (result), "=&r" (tmp), "+Qo" (v->counter)
	: "r" (&v->counter), "Ir" (i)	
	: "cc");
}
```

 两个约束条件的解释: `Q`约束符表示内存地址在一个寄存器中引用, `I`表示可以做为直接操作数;  

这里需要重点介绍下两个指令:  

- ldrex用来读取内存中的值，并标记对该段内存的独占访问

```
ldrex r0, [r1]
```

上面的指令意味着，读取寄存器R1指向的4字节内存值，将其保存到R0寄存器中，同时标记对R1指向内存区域的独占访问。
如果执行ldrex指令的时候发现已经被标记为独占访问了，并不会对指令的执行产生影响。



- strex在更新内存数值时，会检查该段内存是否已经被标记为独占访问

```
strex r0, R1, [R2]
```

如果执行这条指令的时候发现已经被标记为独占访问了，则将寄存器R1中的值更新到寄存器R2指向的内存，并将寄存器R0设置成0。指令执行成功后，会将独占访问标记位清除。  
而如果执行这条指令的时候发现没有设置独占标记，则不会更新内存，且将寄存器R0的值设置成1。


## 2.2 atomic_add_return

```
static inline int atomic_add_return(int i, atomic_t *v)
{
	unsigned long tmp;
	int result;

	smp_mb();
	prefetchw(&v->counter);

	__asm__ __volatile__("@ atomic_add_return\n"
"1:	ldrex	%0, [%3]\n"
"	add	%0, %0, %4\n"
"	strex	%1, %0, [%3]\n"
"	teq	%1, #0\n"
"	bne	1b"
	: "=&r" (result), "=&r" (tmp), "+Qo" (v->counter)
	: "r" (&v->counter), "Ir" (i)	
	: "cc");

	smp_mb();

	return result;
}
```

这个函数和`atomic_add()`基本相同, 除了显示的调用`smp_mb()`解决内存屏障和返回计数结果;  

这sub与add函数都是大同小异这里就不重复解释了;

## 2.3 atomic_read


```
#define atomic_read(v)	ACCESS_ONCE((v)->counter)
```

`ACCESS_ONCE`能够确保在生成汇编代码的时候, 编译器会强制生成从内存中读取变量值的代码, 而不会过度优化;  

## 2.4 atomic_set

```
#define atomic_set(v,i)	(((v)->counter) = (i))
```

该宏应当用于初始化阶段, 并由调用者使用自旋锁或者其他机制来防止并发一致性问题;  


## 2.5 atomic_cmpxchg

```
static inline int atomic_cmpxchg(atomic_t *ptr, int old, int new)
{
	int oldval;
	unsigned long res;

	smp_mb();
	prefetchw(&ptr->counter);

	do {
		__asm__ __volatile__("@ atomic_cmpxchg\n"
		"ldrex	%1, [%3]\n"                        // 将ptr->counter的值保存到oldval
		"mov	%0, #0\n"                              // 将res清零
		"teq	%1, %4\n"                              // 判断oldval与old值是否相同
		"strexeq %0, %5, [%3]\n"                    // 如果oldval == old , 则将new的值赋给ptr->counter
		    : "=&r" (res), "=&r" (oldval), "+Qo" (ptr->counter)
		    : "r" (&ptr->counter), "Ir" (old), "r" (new)
		    : "cc");
	} while (res);                                   // 如果strexeq写入失败, 则res会被置1, 将继续执行, 或者 strrexeq没有执行, 直接退出

	smp_mb();

	return oldval;
}

```

这个函数大概意思是判断`ptr->counter`的值与old值是否相同, 相同则赋值new为新值;  

比如:

```
 atomic_cmpxchg(&flag, 1, 0);
```

如果原来flag的值为1的话, 则设置新值0;


