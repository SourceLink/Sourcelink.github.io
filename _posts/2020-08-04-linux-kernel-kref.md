---
layout: post
title:  "详解Linux内核之引用"
date:   2020-08-04
catalog:  true
author: Sourcelink
tags:
    - kernel
    - kref

---



# 一. 概述

引用计数在linux内核中大量存在, 主要用于存在性保证, 比如当线程初次打开一个文件时, 会在文件系统的散列表中创建文件节点描述符, 并将引用计数设置为1.   
随后的调用者在打开同一个文件时, 不会再次创建这个文件的节点描述符, 而是简单的将文件描述符引用计数加1;  

在每个线程关闭文件对象时, 会将文件节点描述符的引用计数减1, 当最后一个应用文件节点描述符的调用者在关闭文件时, 需要释放响应的文件节点描述符;  


# 二. 内核相关代码解析

引用的数据结构定义如下:

```
struct kref {
	atomic_t refcount;
};
```


## 2.1 kref_init


其初始化程序如下:  

```
static inline void kref_init(struct kref *kref)
{
	atomic_set(&kref->refcount, 1);
}
```

既然初始化了, 那表明有人将要引用包含了引用数据结构的变量, 直接将引用计数设置为1;

举个例子:  


```
struct resv_map *resv_map_alloc(void)
{
	struct resv_map *resv_map = kmalloc(sizeof(*resv_map), GFP_KERNEL);
	if (!resv_map)
		return NULL;

	kref_init(&resv_map->refs);
	spin_lock_init(&resv_map->lock);
	INIT_LIST_HEAD(&resv_map->regions);

	return resv_map;
}
```

这里内核中的一段代码, kref是内嵌在`struct resv_map `的数据结构, 当分配了一块`struct resv_map`类型的内存, 
则增加一个引用, 表示现在这块内存是有人使用的了; 一般来说当后面引用计数为0的时候, 则会释放这块内存;  


## 2.2 kref_get

```
static inline void kref_get(struct kref *kref)
{
	WARN_ON_ONCE(atomic_inc_return(&kref->refcount) < 2);
}
```

该函数执行会增加一次引用，`atomic_inc_return()`会对引用计数进行加１操作;  


## 2.3 kref_sub


```
static inline int kref_sub(struct kref *kref, unsigned int count,
	     void (*release)(struct kref *kref))
{
	WARN_ON(release == NULL);

	if (atomic_sub_and_test((int) count, &kref->refcount)) {        // refcount减去数值count,　并判断其值释放为0
		release(kref);
		return 1;
	}
	return 0;
}
```


如果引用计数`kref->refcount`减了`count`后, 值为0的话, 则会执行`release`回调函数;  


## 2.4 kref_put

```
static inline int kref_put(struct kref *kref, void (*release)(struct kref *kref))
{
	return kref_sub(kref, 1, release);
}
```

该函数是在`kref_sub`基础上的一个封装, 只是每次调用引用计数只会减1, 当减为0, 则执行`release()`回调;  


