---
layout: post
title:  "详解Linux内核之syscall"
date:   2020-07-31
catalog:  true
author: Sourcelink
tags:
    - kernel
    - syscall

---


# 一. 概述

> 内核版本: linux4.1.15 
> 宿主机: imx6ul 
> 主机: ubuntu16.04  


在Linux中，用户空间是不能直接访问内核空间，系统调用是用户空间访问内核的唯一方式， 一般情况下，在应用层调用的`open()`, `read()`等函数就是一个系统调用；

当我们需要打开一个设备的时候， 通常会使用`open(/dev/xxx)`方式进行，通过这样一个系统调用就可以间接的访问驱动层的open();  



# 二. 添加一个系统调用


## 2.1 添加系统调用服务函数

> kernel/sys.c

```
/* add by Sourcelink */
asmlinkage void sys_hello(const char __user *buf, size_t count)
{
	char kernel_buf[100];

	if (buf) {
		copy_from_user(kernel_buf, buf, (count < 100) ? count : 100);
		kernel_buf[99] = '\0';
		printk("sys_hello: %s\r\n", kernel_buf);
	}
}
```

我是在kernel/sys.c中添加，实际在内核调用相关的文件中添加都可以；

## 2.2 添加服务函数头文件

> include/linux/syscalls.h

```
/* add by Sourcelink */
asmlinkage void sys_hello(const char __user *buf, size_t count);
```

## 2.3 添加系统调用定义

> arch/arm/linux/kernel/calls.S

```
/* add by sourcelink 对应388 */ CALL(sys_hello)
```

## 2.4 重新编译内核

在重新编译内核的时候遇到了下面问题：

```
arch/arm/kernel/entry-common.S: Assembler messages:
arch/arm/kernel/entry-common.S:108: Error: __NR_syscalls is not equal to the size of the syscall table
make[1]: *** [scripts/Makefile.build:294: arch/arm/kernel/entry-common.o] Error 1
make: *** [Makefile:947: arch/arm/kernel] Error 2
make: *** Waiting for unfinished jobs....
```


查看`entry-common.S`文件对应位置提示如下：

```
.ifne NR_syscalls - __NR_syscalls
.error "__NR_syscalls is not equal to the size of the syscall table"
.endif
```

意思是__NR_syscalls与系统调用表大小不一致，经过查看发现`__NR_syscalls`是一个宏定义，所以需要修改这个大小；

> arch/arm/include/asm/unistd.h

原来:

```
#define __NR_syscalls  (388)
```

修改后:

```
/* modified by Sourcelink */
#define __NR_syscalls  (392)
```

为什么修改成这个值，后面再详细解析；

## 2.5 编写应用层测试程序

```
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

#define __NR_hello 388


int main(char **arcv, int argc)
{
	syscall(__NR_hello, "sourcelink", 10);
	return 0;
}
```

在上一小节更新了kernel的基础上，将该应用程序放上去运行，不出意外会有如下打印：

```
sys_hello: sourcelink
```

其中`__NR_hello`是我们前面添加系统调用对应的编号，即系统调用号，而`sourcelink`与`10`是系统调用`sys_hello()`对应的形参；

# 三. 内核调用分析

根据前面应用程序，我们先来看看系统调用号，再来看看如何根据系统调用号查找到对应的系统调用函数；

## 3.1 编译错误详解

先看下这段代码：

> arch/arm/linux/kernel/calls.S

```
	.equ NR_syscalls,0
#define CALL(x) .equ NR_syscalls,NR_syscalls+1
#include "calls.S"

/*
 * Ensure that the system call table is equal to __NR_syscalls,
 * which is the value the rest of the system sees
 */
.ifne NR_syscalls - __NR_syscalls
.error "__NR_syscalls is not equal to the size of the syscall table"
.endif
```

这段汇编代码主要是用于**编译阶段进行一个校检作用**，为什么这么说我们详细来分析下:

```
.equ NR_syscalls,0  // 是个赋值语句， 相当于NR_syscalls=0
#define CALL(x) .equ NR_syscalls,NR_syscalls+1        // 则相当于 NR_syscalls += 1
```

再来看看`#include "calls.S"`:


```
/* 0 */		CALL(sys_restart_syscall)
		CALL(sys_exit)
		CALL(sys_fork)
		CALL(sys_read)
		CALL(sys_write)
        ....
```

这样做就是为了给`NR_syscalls`做加法操作，则可以计算出添加系统调用的长度；

而`sys_restart_syscall`对应的系统调用号为0，以此类推则`sys_write`对应的系统调用号为4；

再来看下`#include "calls.S`最后一段代码：

```
#ifndef syscalls_counted
.equ syscalls_padding, ((NR_syscalls + 3) & ~3) - NR_syscalls
#define syscalls_counted
#endif
.rept syscalls_padding
		CALL(sys_ni_syscall)
.endr
```


```
.equ syscalls_padding, ((NR_syscalls + 3) & ~3) - NR_syscalls
```

这段代码是为了计算出`NR_syscalls`与4对齐的补数，即当`NR_syscalls`等于2时，则`syscalls_padding`等于2；

```
.rept syscalls_padding
		CALL(sys_ni_syscall)
.endr
```

它会重复调用`syscalls_padding`次`CALL(sys_ni_syscall)`， 所以最后`NR_syscalls`的大小会与4对齐，  
所以前面编译的时候为什么在只添加了一个系统调用的基础上而`__NR_syscalls`需要修改为392；  
其中系统调用`sys_ni_syscall`，它除了返回`-ENOSYS`外不做任何其他工作；  

## 3.2 系统调用查找

系统调用的实现，是产生一个软中断，使cpu进行内核态，再调用根据应用层传入的系统调用号，找到对应的调用函数，再进行调用；

那么如何根据系统调用号进行相应的函数查找？

现在看看软中断的服务函数：

```
#undef CALL
#define CALL(x) .long x
```

```
	.align	5
ENTRY(vector_swi)

	sub	sp, sp, #S_FRAME_SIZE                                             ①
	stmia	sp, {r0 - r12}			@ Calling r0 - r12
 ARM(	add	r8, sp, #S_PC		)
 ARM(	stmdb	r8, {sp, lr}^		)	@ Calling sp, lr
 THUMB(	mov	r8, sp			)
 THUMB(	store_user_sp_lr r8, r10, S_SP	)	@ calling sp, lr
	mrs	r8, spsr			@ called from non-FIQ mode, so ok.
	str	lr, [sp, #S_PC]			@ Save calling PC
	str	r8, [sp, #S_PSR]		@ Save CPSR
	str	r0, [sp, #S_OLD_R0]		@ Save OLD_R0

	zero_fp
	alignment_trap r10, ip, __cr_alignment
	enable_irq
	ct_user_exit
	get_thread_info tsk

#if defined(CONFIG_OABI_COMPAT)                                            ②

#ifdef CONFIG_ARM_THUMB
	tst	r8, #PSR_T_BIT
	movne	r10, #0				@ no thumb OABI emulation
 USER(	ldreq	r10, [lr, #-4]		)	@ get SWI instruction
#else
 USER(	ldr	r10, [lr, #-4]		)	@ get SWI instruction
#endif
 ARM_BE8(rev	r10, r10)			@ little endian instruction

#elif defined(CONFIG_AEABI)


#elif defined(CONFIG_ARM_THUMB)
	/* Legacy ABI only, possibly thumb mode. */
	tst	r8, #PSR_T_BIT			@ this is SPSR from save_user_regs
	addne	scno, r7, #__NR_SYSCALL_BASE	@ put OS number in
 USER(	ldreq	scno, [lr, #-4]		)

#else
	/* Legacy ABI only. */
 USER(	ldr	scno, [lr, #-4]		)	@ get SWI instruction
#endif

	adr	tbl, sys_call_table		@ load syscall table pointer               ③

    ...
    
local_restart:
	ldr	r10, [tsk, #TI_FLAGS]		@ check for syscall tracing
	stmdb	sp!, {r4, r5}			@ push fifth and sixth args

	tst	r10, #_TIF_SYSCALL_WORK		@ are we tracing syscalls?
	bne	__sys_trace

	cmp	scno, #NR_syscalls		@ check upper syscall limit
	adr	lr, BSYM(ret_fast_syscall)	@ return address                       ⑤
	ldrcc	pc, [tbl, scno, lsl #2]		@ call sys_* routine               ④

	add	r1, sp, #S_OFF
2:	cmp	scno, #(__ARM_NR_BASE - __NR_SYSCALL_BASE)
	eor	r0, scno, #__NR_SYSCALL_BASE	@ put OS number back
	bcs	arm_syscall
	mov	why, #0				@ no longer a real syscall
	b	sys_ni_syscall			@ not private func

#if defined(CONFIG_OABI_COMPAT) || !defined(CONFIG_AEABI)
	/*
	 * We failed to handle a fault trying to access the page
	 * containing the swi instruction, but we're not really in a
	 * position to return -EFAULT. Instead, return back to the
	 * instruction and re-enter the user fault handling path trying
	 * to page it in. This will likely result in sending SEGV to the
	 * current task.
	 */
9001:
	sub	lr, lr, #4
	str	lr, [sp, #S_PC]
	b	ret_fast_syscall
#endif
ENDPROC(vector_swi)

```

> ①: 一些保护现场与初始操作  

> ②: 获取系统调用编号，系统调用有两种规范，一种是老的OABI（系统调用号来自swi指令中），另外一种是ARM ABI，也就是EABI（系统调用号来自r7）。笔者在编译内核时候有配置：

```
CONFIG_AEABI=y
```

所以前面应用层的代码可以改成下面这种情况：

```
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

#define __NR_hello 388

void hello(char *buf, int count)
{
	asm("mov r0, %0\n" /* save the argment in r0 */
	    "mov r1, %1\n" /* save the argment in r0 */
	    "mov r7, %2\n" /* sys call number */
	    "swi 0\n"	   /* do the system call */
	    :
	    : "r"(buf), "r"(count), "i"(__NR_SYSCALL_BASE + __NR_hello)
	    : "r0", "r1");
}

int main(char **arcv, int argc)
{
	hello("sourcelink", 10);
    return 0;
}
```

下面是asm系统调用getpid的方法：

```
int s_getpid(void)
{
	int pid = 0;

	__asm__ volatile (
		"mov r7, %1\n"
		"swi 0\n"
		"mov %0, r0\n"
		:"=r"(pid)
		:"i"(__NR_getpid)
		:"cc"
	);

	return pid;
}
```

> ③: 将系统调用表地址存放在tbl  

我们现在看下sys_call_table的定义及初始化过程:

```
	.type	sys_call_table, #object
ENTRY(sys_call_table)
#include "calls.S"
```

这里由包含了`#include "calls.S"`， 但这次`CALL`宏定义已经改变， 在swi服务函数前面：

```
#undef CALL
#define CALL(x) .long x
```

所以此时展开的效果：

```
    .long sys_restart_syscall
    .long sys_exit
    .long sys_fork

    ....

```

所以sys_call_table中就是这些函数地址了，偏移从0开始；

> ④: 在`sys_call_table`中根据系统调用号，找到对应的服务函数地址并传递给pc 

> ⑤: 设置系统调用后，需要执行的函数`ret_fast_syscall`  


## 3.3 系统调用函数的定义

这就是一次系统调用的流程，但是在查找一些系统系统调用函数的定义时候， 发现无法直接找到其原型，比如`sys_write`

```
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
    ....
	return ret;
}
```

其中`SYSCALL_DEFINE3`对应的宏如下:

```
....
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
....

#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```

这里主要看下`__SYSCALL_DEFINEx`:

```
#define __SYSCALL_DEFINEx(x, name, ...)					\
	asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))	\
		__attribute__((alias(__stringify(SyS##name))));		\
	static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));	\
	asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
	asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
	{								\
		long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));	\
		__MAP(x,__SC_TEST,__VA_ARGS__);				\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
		return ret;						\
	}								\
	static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```


以 `write`为例展开的话： 

```
	asmlinkage long sys_write(unsigned int fd, const char __user *buf, size_t count) __attribute__((alias(__stringify(SyS_write))));		\
	static inline long SYSC_write(unsigned int fd, const char __user *buf, size_t count);	\
	asmlinkage long SyS_write(unsigned int fd, const char __user *buf, size_t count);	\
	asmlinkage long SyS_write(unsigned int fd, const char __user *buf, size_t count)	\
	{								\
		long ret = SYSC_write((unsigned int)fd, (const char __user *)buf, (szie_t)count);	\
		BUILF_BUG_ON_ZERO(!__TYPE_IS_LL(unsigned int) && sizeof(unsigned int) > sizeof(long))         \
         BUILF_BUG_ON_ZERO(!__TYPE_IS_LL(const char __user*) && sizeof(const char __user*) > sizeof(long))    \
         BUILF_BUG_ON_ZERO(!__TYPE_IS_LL(szie_t) && sizeof(size_t) > sizeof(long))                \
		__PROTECT(3, ret, fd, buf, count);	\
		return ret;						\
	}								\
	static inline long SYSC_write(unsigned int fd, const char __user *buf, size_t count)
```

这里会定义多个函数: `sys_write`, `SYSC_write`, `SyS_write`;

可以看到其实write的最后实际是`static inline long SYSC_write(unsigned int fd, const char __user *buf, size_t count)`, 
在`SyS_write`中会调用`SYSC_write`，而前面在定义声明`sys_write`的时候，使用了别名机制， `sys_write`是`SyS_write`的别名， 即调用  
`sys_write`等同于`调用SyS_write`;

**PS:** 可以看出不同等级的宏是和调用函数的参数有关， 如果是带有4个参数的系统调用函数，则需要使用`SYSCALL_DEFINE4`去定义；



# 附录. 系统调用号

> arch/arm/include/uapi/asm/unistd.h

```
#define __NR_restart_syscall		(__NR_SYSCALL_BASE+  0)
#define __NR_exit			(__NR_SYSCALL_BASE+  1)
#define __NR_fork			(__NR_SYSCALL_BASE+  2)
#define __NR_read			(__NR_SYSCALL_BASE+  3)
#define __NR_write			(__NR_SYSCALL_BASE+  4)
#define __NR_open			(__NR_SYSCALL_BASE+  5)
#define __NR_close			(__NR_SYSCALL_BASE+  6)

.....

```

