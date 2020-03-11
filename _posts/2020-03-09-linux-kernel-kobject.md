---
layout: post
title:  "详解Linux内核kobject模型"
date:   2020-03-09
catalog:  true
author: Sourcelink
tags:
    - kobject
    - kernel
---

# 一. 概述

在看linux的设备模型发现kobject在很多地方都有使用, 经过分析发现它非常经典的实现了面向对象;  

接下来分别先解释下kobject等结构体的概念;


## 1.1 kobject

kobject是一个结构体, 它的结构体声明如下:

```
struct kobject {
	const char		*name;           // 设备的名称
	struct list_head	entry;           // 用于挂接链表
	struct kobject		*parent;     // 父亲节点
	struct kset		*kset;           // 所属的kset
	struct kobj_type	*ktype;          // 对应ktype操作
	struct kernfs_node	*sd;
	struct kref		kref;            // 引用计数
	unsigned int state_initialized:1; // 是否初始化标志
	unsigned int state_in_sysfs:1;
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};
```

kobject只是一个**通用对象**的表示(类似c++中的基类), 它抽象出一些通用的属性（如引用计数）以及通用接口, 它通常是被其他数据结构(结构体)所包含(继承kobject),  
而每个外围数据结构各自完成这些接口的内部实现, 所以会表现出不同的属性, 因此我们更关心kobejct的外围数据结构;  


## 1.2 kset

kset是一组kobject的集合, 结构体声明如下:  

```
struct kset {
	struct list_head list;                       // 用于挂接kobject的链表
	spinlock_t list_lock;
	struct kobject kobj;                         // kset内嵌的object, 该obj可以用于表示该kset对象
	const struct kset_uevent_ops *uevent_ops;    // 当事件发生时候的可以调用的ops
};
```

当我们想统一管理某些有类似属性的kobjects时, 可以将它们加入到一个集合中, 方便进行管理,   
比如当热拔插事件发生时, 可以同时通知到该集合中的所有kobjects;  



## 1.3 ktype

ktype为其kobject提供一些方法,  比如release方法用于释放kobejct所占用的资源或包含该kobject模块的资源;  

```
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;        // 该kobject的sysfs文件系统接口
	struct attribute **default_attrs;
	const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
	const void *(*namespace)(struct kobject *kobj);
};
```

要注意每个kobject的**ktype是可以相同, 也可以不同的**, 主要取决于包含该kobject的数据结构, 在初始化kobejct的时候ktype注册了哪些回调函数;  
**因此每个外围结构体所在的模块都应该定义一个ktype，这个ktype就使kobject完成了由通用向特殊的过渡**;  


# 二. 功能分析


## 2.1 kobject查找

前面说过kobject的使用场景, 基本都是内嵌在其他数据结构中, 比如数据结构`struct driver_private`:

```
struct driver_private {
	struct kobject kobj;
	struct klist klist_devices;
	struct klist_node knode_bus;
	struct module_kobject *mkobj;
	struct device_driver *driver;
};
```

使用内嵌而不用指针也是为了方便通过kobject来反向定位其外围结构的实例, 通过我们熟知的container_of()宏便可以做到, 比如:  

```
#define to_driver(obj) container_of(obj, struct driver_private, kobj)
```

由于kobject是系统统一管理的, 因此先找到kobject对象进而跟踪到其代表的具体对象是很常见的做法.  



## 2.2 kset管理功能

前面有说过kset是一组kobject的集合, 但是它是如何将一组kobejct进行集合的呢?  

请看下图:  

![](/images/kobject/20200310104913978_1220199041.png)


其中通过成员list进行挂载, 可以看到是使用kobject结构体中的entry成员作为挂载节点;  


再来看下kset管理的好处, 以总线上的`driver_kset`为例, 按理`driver_kset`应该为driver kobject的合集,  即`driver_kset`的链表上挂载了很多driver的object;  

以`driver_find()`为例：

```
struct device_driver *driver_find(const char *name, struct bus_type *bus)
{
	struct kobject *k = kset_find_obj(bus->p->drivers_kset, name);
	struct driver_private *priv;

	if (k) {
		/* Drop reference added by kset_find_obj() */
		kobject_put(k);
		priv = to_driver(k);
		return priv->driver;
	}
	return NULL;
}
```

其中kset_find_obj()完成了从集合kset中查找obj的实例：

```
struct kobject *kset_find_obj(struct kset *kset, const char *name)
{
	struct kobject *k;
	struct kobject *ret = NULL;

	spin_lock(&kset->list_lock);

	list_for_each_entry(k, &kset->list, entry) {
		if (kobject_name(k) && !strcmp(kobject_name(k), name)) {
			ret = kobject_get_unless_zero(k);
			break;
		}
	}

	spin_unlock(&kset->list_lock);
	return ret;
}
```

宏`list_for_each_entry`最终通过`container_of()`获取到一个个kobject;  

这样就完成了一次从kset集合中查找一个内嵌了kobject的数据结构流程;  

**问题**:  再来说说kset中为何内嵌了kobject?

**答**:  虽然kset是一组kobject的集合且是通过list成员进行链接挂载, 但是作为一个对象本身它应该还有其他功能需要一定的表示, 所以通过内嵌了一个kobject来表示;  
比如kset被引用时,  因为它本身没有该功能, 则需要通过其内嵌成员kobject来进行引用操作, 即kset依赖于kobj维护引用计数;  

## 2.3 ktype销毁

从查看ktype的数据结构知道其中有个release方法, 前面说过它可以销毁kobject, 也可以销毁内嵌了该kobejct的模块, 它是如何做到的? 接下来看个例子:


```
static void bus_release(struct kobject *kobj)
{
	struct subsys_private *priv =
		container_of(kobj, typeof(*priv), subsys.kobj);
	struct bus_type *bus = priv->bus;

	kfree(priv);
	bus->p = NULL;
}
```

这是总线在初始化kobejct的时候注册的release函数,  可以看到当传进来一个kobejct变量, 内嵌在`structsubsys_private`中,  
先通过该kobejct使用`container_of()`方法找到子系统的私有数据, 接下来再free掉, 并且将总线上对应的指针清零;  


## 2.4 kobejct引用

前面讲的ktye的销毁功能都是自动执行的, 它是如何做到的?  

我们知道每个kobject有个引用计数, 当引用计数为0的时候就自动调用release函数进行销毁;  

但问题来了, **这个自动是什么时候, 是怎么实现的**?  

其实kobject在被引用的时候会调`kobect_get()`函数, 其中引用计数会增加,  
在去引用的时候会调用`kobect_put()`函数, 会对引用计数进行减操作, 而且会判断引用计数是否为0,  
为零则会调用`kobject_release()`函数, 其中会调用kobject对象的自身的release函数进行释放操作;  


# 三. kset与kobject的关系

这节贴几张内核中, kset与kobject的关系, 都是通过分析代码得到的

## 3.1 kset与kset

![](/images/kobject/20200309163519606_185342131.png)

原来kset集合之前也是有从属关系, 在分析bus的kset集合与驱动的kset集合得到;  

## 3.2 kobject与kobject

![](/images/kobject/20200309163826945_421097133.png)

可这个很简单, kobject2的父对象为kobject1;  比如block设备, char设备都属于设备;  

## 3.3 kset与kobject

![](/images/kobject/20200309164253072_95925556.png)

这个就不解释了;

## 3.4 kobject与kset

![](/images/kobject/20200309164516919_265961035.png)

该情况可以参考`/sys/`目录下注册的kernel对象(kobject对象)与kernerl目录下的slab对象(kset对象);


# 四. 示例分析

> sample/kobject/kobject-example.c

创建一个kobejct的主要调用流程如下, 会分析重要的函数:  

```
example_init
    	kobject_create_and_add("kobject_example", kernel_kobj);
    	    kobject_add(kobj, parent, "%s", name);
    	        kobject_add_varg(kobj, parent, fmt, args);
```


## 4.1 kobject_add_varg

```
static int kobject_add_varg(struct kobject *kobj, struct kobject *parent,
			    const char *fmt, va_list vargs)
{
	int retval;

	retval = kobject_set_name_vargs(kobj, fmt, vargs);        // 传进的格式化参数作为kobejct的名字
	if (retval) {
		printk(KERN_ERR "kobject: can not set name properly!\n");
		return retval;
	}
	kobj->parent = parent;
	return kobject_add_internal(kobj);                        // 下一节详细分析
}

```


## 4.2 kobject_add_internal

```
static int kobject_add_internal(struct kobject *kobj)
{
	int error = 0;
	struct kobject *parent;

	if (!kobj)
		return -ENOENT;

	if (!kobj->name || !kobj->name[0]) {
		WARN(1, "kobject: (%p): attempted to be registered with empty "
			 "name!\n", kobj);
		return -EINVAL;
	}

	parent = kobject_get(kobj->parent);                     // 获取该obj的父对象

	/* join kset if set, use it as parent if we do not already have one */
	if (kobj->kset) {                                       // 判断该obj的kset指针是否已经设置了对象
		if (!parent)
			parent = kobject_get(&kobj->kset->kobj);       // 如果当前object还没设置父对象, 则引用设置的kset对象为父对象
		kobj_kset_join(kobj);                              // 则将其加入到其设置的kset集合中(将其挂载到kset的链表上)
		kobj->parent = parent;                             // 设置父对象
	}

	error = create_dir(kobj);                              // 在系统创建目录 具体不清楚
	if (error) {
        kobj_kset_leave(kobj);                             // 如果创建目录失败则将kobject移除kset链表
		kobject_put(parent);                              // 去除引用
		kobj->parent = NULL;

		/* be noisy on error issues */
		if (error == -EEXIST)
			WARN(1, "%s failed for %s with "
			     "-EEXIST, don't try to register things with "
			     "the same name in the same directory.\n",
			     __func__, kobject_name(kobj));
		else
			WARN(1, "%s failed for %s (error: %d parent: %s)\n",
			     __func__, kobject_name(kobj), error,
			     parent ? kobject_name(parent) : "'none'");
	} else
		kobj->state_in_sysfs = 1;

	return error;
}
```


总结该函数主要是将传进来的形参kobject对象加入指定的kset集合中(加入kset的链表)  
并根据该kobject在子系统下的关系创建对应的目录;  


