---
layout: post
title:  "linux内核之initcall机制"
date:   2020-02-14
catalog:  true
author: Sourcelink
tags:
    -linux
    -kernel
---


# 一. 概述

引入问题：

initcall机制是做什么的？它有哪些功能？


回顾昨天的 machie_desc, 其中`init_machine`是在哪被调用的?

查看发现是在如下函数被调用:

```

static int __init customize_machine(void)
{
	/*
	 * customizes platform devices, or adds new ones
	 * On DT based machines, we fall back to populating the
	 * machine from the device tree, if no callback is provided,
	 * otherwise we would always need an init_machine callback.
	 */
	of_iommu_init();
	
	if (machine_desc->init_machine)
		machine_desc->init_machine();
#ifdef CONFIG_OF
	else
		of_platform_populate(NULL, of_default_bus_match_table,
					NULL, NULL);
#endif
	return 0;
}
```


搜索发现只是被一个宏进行调用, 但是没有其他地方再调用了

```
arch_initcall(customize_machine);
```


# 二. initcall


很多的调用宏在`include/linux/init.h`:


```
#define pure_initcall(fn)		__define_initcall(fn, 0)
#define core_initcall(fn)		__define_initcall(fn, 1)
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall(fn)		__define_initcall(fn, 7)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
```


以`arch_initcall(customize_machine);`为例暂开讲解;  

## 2.1 __define_initcall


```
#define __define_initcall(fn, id) \
	static initcall_t __initcall_##fn##id __used \
	__attribute__((__section__(".initcall" #id ".init"))) = fn; \
	LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```


将`arch_initcall(customize_machine)`展开:


```
static initcall_t __initcall_customize_machine3 __used __attribute__((__section__(".initcall3.init"))) = customize_machine; 
```


其中的`LTO_REFERENCE_INITCALL`, 经验证CONFIG_LTO没有定义:   

```
#ifdef CONFIG_LTO
/* Work around a LTO gcc problem: when there is no reference to a variable
 * in a module it will be moved to the end of the program. This causes
 * reordering of initcalls which the kernel does not like.
 * Add a dummy reference function to avoid this. The function is
 * deleted by the linker.
 */
#define LTO_REFERENCE_INITCALL(x) \
	; /* yes this is needed */			\
	static __used __exit void *reference_##x(void)	\
	{						\
		return &x;				\
	}
#else
#define LTO_REFERENCE_INITCALL(x)
#endif
```



## 2.2 段属性INIT_CALLS

> include/asm-generic/vmlinux.lds.h

```
#define INIT_CALLS_LEVEL(level)						\
		VMLINUX_SYMBOL(__initcall##level##_start) = .;		\ 
		*(.initcall##level##.init)				\
		*(.initcall##level##s.init)				\

#define INIT_CALLS							\
		VMLINUX_SYMBOL(__initcall_start) = .;			\
		*(.initcallearly.init)					\
		INIT_CALLS_LEVEL(0)					\
		INIT_CALLS_LEVEL(1)					\
		INIT_CALLS_LEVEL(2)					\
		INIT_CALLS_LEVEL(3)					\
		INIT_CALLS_LEVEL(4)					\
		INIT_CALLS_LEVEL(5)					\
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					\
		INIT_CALLS_LEVEL(7)					\
		VMLINUX_SYMBOL(__initcall_end) = .;
```

看下是谁从这些存储位置取出函数指针来运行?


```
extern initcall_t __initcall_start[];
extern initcall_t __initcall0_start[];
extern initcall_t __initcall1_start[];
extern initcall_t __initcall2_start[];
extern initcall_t __initcall3_start[];
extern initcall_t __initcall4_start[];
extern initcall_t __initcall5_start[];
extern initcall_t __initcall6_start[];
extern initcall_t __initcall7_start[];
extern initcall_t __initcall_end[];

static initcall_t *initcall_levels[] __initdata = {
	__initcall0_start,
	__initcall1_start,
	__initcall2_start,
	__initcall3_start,
	__initcall4_start,
	__initcall5_start,
	__initcall6_start,
	__initcall7_start,
	__initcall_end,
};
```


可以将宏`INIT_CALLS_LEVEL`展开, 可以看到每个宏都有一个__initcallx_start对应(x表示0, 1, 2 等数字);    


## 2.3 do_initcalls

```
static void __init do_initcalls(void)
{
	int level;

	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
		do_initcall_level(level);
}
```

处理对应等级下的函数指针；　　

## 2.4 do_initcall_level

```
static void __init do_initcall_level(int level)
{
	initcall_t *fn;

	strcpy(initcall_command_line, saved_command_line);
	parse_args(initcall_level_names[level],
		   initcall_command_line, __start___param,
		   __stop___param - __start___param,
		   level, level,
		   &repair_env_string);
	for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
		do_one_initcall(*fn);
}
```


假设现在处理level为３的情况，通过for循环，调用３到４之前的全部的函数指针；它提取出来的fn函数指针传给了`do_one_initcall`;


## 2.5 do_one_initcall

```
int __init_or_module do_one_initcall(initcall_t fn)
{
	int count = preempt_count();
	int ret;
	char msgbuf[64];

	if (initcall_blacklisted(fn))
		return -EPERM;

	if (initcall_debug)
		ret = do_one_initcall_debug(fn);
	else
		ret = fn();                            ①　
    .....
}
```

> ①：如果没有开启debug模式函数最终会在这被调用；　


# 三. initcall 调用栈

```
start_kernel
    rest_init
        kernel_init
            kernel_init_freeable
                do_basic_setup
                    do_initcalls
```

可以看到最后是在start_kernel下的rest_init函数中的一个内核线程kernel_init中被调用；　　


# 四. 回顾

回顾do_initcalls可以看到它将存储的函数指针都执行了一遍；

再看下下面的宏：

```
#define __initcall(fn) device_initcall(fn)
#define module_init(x)	__initcall(x);
```

可以看到很多驱动init函数都是通过`module_init`进行存储；

- 情况１

如果把这些驱动编译进内核的话，在启动的时候这些驱动就会通过initcall被加载；　

- 情况２

是将驱动编译成模块的形式，使用命令insmod进行加载的，这里面肯定和这个命令；　


