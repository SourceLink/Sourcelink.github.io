---
layout: post
title:  "【从零开始写RTOS】内核双向链表"
date:   2019-12-16
catalog:  true
author: Sourcelink
tags:
    - 双向链表
    - rtos

---

# 一. 数据结构

## 1.1 成员
```
struct list_head {
	struct list_head *next, *prev;
};
```

示意图如下：
![](/images/rtos/20191216203821320_970381094.png)

## 1.2 初始化

- 静态初始化

```
#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
```


- 动态初始化

```
static __inline void list_head_init(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
```

初始化完以后的示意图如下：

![](/images/rtos/20191216204136561_352480110.png)

PS：node不具备成员含义，在这只是表示一个节点;


# 二. add

- 示意代码

```
static __inline void __list_add(struct list_head *new, struct list_head *prev, struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}

static __inline void list_add(struct list_head *head, struct list_head *new)
{
	__list_add(new, head, head->next);
}
```

ps: 以一个初始化完的节点情况下添加一个节点，`head->next == head == next->prev`;

- 执行`next->prev = new;`

![](/images/rtos/20191216204611201_54454629.png)

- 执行`new->next = next;`

![](/images/rtos/20191216205021282_59412531.png)


- 执行`new->prev = prev;`

![](/images/rtos/20191216205123328_1011615156.png)

- 执行`prev->next = new;`

![](/images/rtos/20191216205234644_44546143.png)

# 三. delete


- 示意代码

```
static __inline void __list_delete(struct list_head *prev, struct list_head *next)
{
	next->prev = prev;
	prev->next = next;
}

static __inline void list_delete(struct list_head *del)
{
	__list_delete(del->prev, del->next);
}
```

假设链表情况如下， 删除红框节点：

![](/images/rtos/20191216205513252_1372641593.png)

- 执行`next->prev = prev;`


![](/images/rtos/20191216205638420_321891165.png)


- 执行`prev->next = next;`


![](/images/rtos/20191216205744311_252545492.png)


但是可以看到被删除的节点，`prev`和`next`还是指向原节点，但是已经在链表中被删除;

使用下面代码可以删除的同时初始化节点

```
static __inline void list_delete_init(struct list_head *node)
{
	list_delete(node);
	list_head_init(node);
}
```


# 四. container


```
#define container_of(ptr, type, member) \
	((type *)(((unsigned char *)(ptr)) - ((unsigned int)(&((type *)0)->member))))
```

可以根据链表节点获取到结构体的地址（变量地址）

接设有个数据结构如下：

```
struct test_container {
    unsigned int val;
    struct list_head slot;
    unsigned int data;
    unsigned int time;
}
```

exp：

```
struct test_container _cont;

void fun(void) {

    struct test_container *tmp_container = NULL;
    struct list_head *curr = &_cont.slot;

    tmp_container = container_of(curr,  struct test_container, slot);
}
```

上面的函数可以计算出`tmp_container`指针指向全局变量`_cont`;

计算偏移值大小：

![](/images/rtos/20191216211006168_1506263960.png)


根据curr传进来的值：

![](/images/rtos/20191216211326001_1235595729.png)

根据偏移值与节点地址就可以计算出结构体变量的地址（0x20008800 - 0x04）。