---
layout: post
title:  "详解Linux内核红黑树算法的实现"
date:   2019-09-19
catalog:  true
author: Sourcelink
tags:
    - 红黑树
    - kernel

---



# 一. 概述

> 宿主机: ubuntu16.04  
> 内核版本: linux-3.0.8  


红黑树是每个节点都带有颜色属性的二叉查找树，颜色为红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的额外要求：

> 1.节点是红色或黑色。  
> 2. 根是黑色。  
> 3. 所有叶子都是黑色（叶子是NIL节点）。  
> 4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点）  
> 5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。  


如下是一颗符合上述性质的红黑树:  

![](/images/rbtree/20190919094722372_231363163.png)


内核中红黑树源码位于:  

> include/linux/rbtree.h  
> lib/rbtree.c

通过搜索内核源码, 截取了一段红黑树的应用代码(rtc驱动相关)如下:  

```
static void alarm_enqueue_locked(struct alarm *alarm)
{
	struct alarm_queue *base = &alarms[alarm->type];
	struct rb_node **link = &base->alarms.rb_node;
	struct rb_node *parent = NULL;
	struct alarm *entry;
	int leftmost = 1;
	bool was_first = false;

	pr_alarm(FLOW, "added alarm, type %d, func %pF at %lld\n",
		alarm->type, alarm->function, ktime_to_ns(alarm->expires));

	if (base->first == &alarm->node) {
		base->first = rb_next(&alarm->node);
		was_first = true;
	}
	if (!RB_EMPTY_NODE(&alarm->node)) {
		rb_erase(&alarm->node, &base->alarms);
		RB_CLEAR_NODE(&alarm->node);
	}

	while (*link) {
		parent = *link;
		entry = rb_entry(parent, struct alarm, node);
		/*
		* We dont care about collisions. Nodes with
		* the same expiry time stay together.
		*/
		if (alarm->expires.tv64 < entry->expires.tv64) {        ①
			link = &(*link)->rb_left;
		} else {
			link = &(*link)->rb_right;
			leftmost = 0;
		}
	}
	if (leftmost)
		base->first = &alarm->node;
	if (leftmost || was_first)
		update_timer_locked(base, was_first);

	rb_link_node(&alarm->node, parent, link);                   ②
	rb_insert_color(&alarm->node, &base->alarms);               ③
}
```

> ①: 根据`tv64`的值, 选择将要插入哪个节点的左或右节点;  
> ②: 通过link指针将新节点连接到对应的节点下;  
> ③: 为新节点上色;  

其中与红黑树最重要的三个函数有:`rb_link_node()`、`rb_insert_color()`、 `rb_erase()`;  


# 二. 详解

## 2.1 struct rb_node

```
struct rb_node
{
	unsigned long  rb_parent_color;
#define	RB_RED		0
#define	RB_BLACK	    1
	struct rb_node *rb_right;
	struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
```

一开始看到这个结构体脑海中的思路大概如下: `rb_parent_color`成员存储该父节点颜色, `rb_right`和`rb_left`分别存储该节点的右节点和左节点;  但是后面发现父节点好像没有存储, 直到后面看到宏`rb_parent`才恍然大悟, `rb_parent_color`成员其实是存储了父节点的地址和节点本身的颜色;  

```
#define rb_parent(r)   ((struct rb_node *)((r)->rb_parent_color & ~3))
```

实现一个变量存储两种信息的关键是: `__attribute__((aligned(sizeof(long))))`, 指定该结构体对齐长度为`sizeof(long)`, 在32位机中,长度为4字节, 在64位机中长度为8字节;  

假设现在在一台32位机上运行, 使用`struct rb_node`定义变量的时候就会发现, **变量的地址是4的倍数**; 即变量地址的最后一个字节为`0x?4`, 对应的二进制为`b?100`, 可以看到最低的两位是0, 可以看出这两位是未使用的, 内核用最低一位用于存储节点颜色; 这也就明白为什么宏`rb_parent()`的含义, 用于清除最低两位的颜色, 则对应的就是父节点的地址;  


总结下`rb_parent_color`成员保存了两个信息:  

> 父节点的地址  
> 本节点的颜色  

## 2.2 相关宏讲解


```
#define rb_parent(r)   ((struct rb_node *)((r)->rb_parent_color & ~3))    // 获取父节点地址
#define rb_color(r)   ((r)->rb_parent_color & 1)                          // 获取该节点的颜色信息
#define rb_is_red(r)   (!rb_color(r))                                     // 判断该节点是否为红色
#define rb_is_black(r) rb_color(r)                                        // 判断该节点是否为黑色
#define rb_set_red(r)  do { (r)->rb_parent_color &= ~1; } while (0)       // 设置节点颜色为红色
#define rb_set_black(r)  do { (r)->rb_parent_color |= 1; } while (0)      // 设置节点颜色为黑色

static inline void rb_set_parent(struct rb_node *rb, struct rb_node *p)   // 设置父节点
{
	rb->rb_parent_color = (rb->rb_parent_color & 3) | (unsigned long)p;   // 先清空父节点地址,再或上新的父节点地址
}
static inline void rb_set_color(struct rb_node *rb, int color)            // 设置节点颜色
{
	rb->rb_parent_color = (rb->rb_parent_color & ~1) | color;             // 先清空颜色信息, 再或上新颜色
}
```

## 2.3 rb_link_node

```
static inline void rb_link_node(struct rb_node * node, struct rb_node * parent,
				struct rb_node ** rb_link)
{
    node->rb_parent_color = (unsigned long )parent;   // 保存父节点地址
	node->rb_left = node->rb_right = NULL;            // 将该节点的左右节点置空

	*rb_link = node;                                  // 父节点连接上该节点
}
```

连接节点大概是上述三个步骤, 新插入的节点的颜色默认为红色, 最后通过传进的参数`rb_link`再连接进父节点的左或右子节点;  


## 2.4 rb_insert_color

```
void rb_insert_color(struct rb_node *node, struct rb_root *root)
{
	struct rb_node *parent, *gparent;

	while ((parent = rb_parent(node)) && rb_is_red(parent))        // 如果父节点存在并且为红色
	{
		gparent = rb_parent(parent);                               // 获取祖父节点

		if (parent == gparent->rb_left)                            // 如果父节点是祖父节点的左子节点
		{
			{
				register struct rb_node *uncle = gparent->rb_right;    // 则右子节点为叔父节点
				if (uncle && rb_is_red(uncle))                    // 如果叔父节点存在且为红色节点
				{
					rb_set_black(uncle);
					rb_set_black(parent);
					rb_set_red(gparent);
					node = gparent;
					continue;
				}
			}

			if (parent->rb_right == node)                           // 如果该节点插入是父节点的右子节点
			{
				register struct rb_node *tmp;
				__rb_rotate_left(parent, root);                     // 绕父节点左旋
				tmp = parent;
				parent = node;                                      // 更新parent的值, 因为在左旋中父节点与其子节点关系已经交换
				node = tmp;
			}

			rb_set_black(parent);
			rb_set_red(gparent);
			__rb_rotate_right(gparent, root);                       // 绕祖父节点右旋
		} else {                                                    // 父节点是祖父节点的右子节点
			{
				register struct rb_node *uncle = gparent->rb_left;  // 则左子节点为叔父节点
				if (uncle && rb_is_red(uncle))
				{
					rb_set_black(uncle);
					rb_set_black(parent);
					rb_set_red(gparent);
					node = gparent;
					continue;
				}
			}

			if (parent->rb_left == node)                            // 如果新插入的节点是父节点的左子节点
			{
				register struct rb_node *tmp;
				__rb_rotate_right(parent, root);                    // 绕父节点进行右旋
				tmp = parent;
				parent = node;
				node = tmp;
			}

			rb_set_black(parent);
			rb_set_red(gparent);
			__rb_rotate_left(gparent, root);                        // 绕祖父节点左旋
		}
	}

	rb_set_black(root->rb_node);                                    // 将root节点设置为黑色
}
```

这里用以下几种情况讲解下节点插入的过程: 

- case1: 

父节点是祖父节点的左子节点, 没有叔父节点且插入的是父节点的左节点, 情况如下, 02是新插入的节点:  

![](/images/rbtree/20190919150427287_408480116.png)

按照上述流程, 需要将父节点置黑, 祖父节点置红, 再绕祖父节点做右旋转, 转换流程图如下:  

![](/images/rbtree/20190919151342744_34951411.png)

绕祖父节点右旋, 大致的功能是使节点4变成节点的父节点, 而节点5变成节点4的右子几点;  

- case2: 

父节点是祖父节点的左子节点, 没有叔父节点且插入的是父节点的右节点, 情况如下, 06是新插入的节点: 

![](/images/rbtree/20190919151949944_460536537.png)

相比case1, 多增加一步绕父节点左旋转, 流程如下:  

![](/images/rbtree/20190919153410897_1901182973.png)


- case3: 

父节点是祖父节点的左子节点, 叔父节点且为红色, 插入的是父节点的左节点

![](/images/rbtree/20190919153601393_1385818923.png)

进行了颜色转换后, 结束了这次循环, while的检测条件未通过, 则进行最后一步将根节点设置成黑色, 最后这颗树符合红黑树性质; 

![](/images/rbtree/20190919154254089_1814519848.png)


以上三种case都是讲解父节点为祖父节点的左子节点情况, 但做为祖父节点的右子节点情况是差不多的;  


## 2.5 rb_erase

```
void rb_erase(struct rb_node *node, struct rb_root *root)
{
	struct rb_node *child, *parent;
	int color;

	if (!node->rb_left)                        // 如果没有左子节点
		child = node->rb_right;                // 孩子为节点的右子节点
	else if (!node->rb_right)                  // case1 如果没有右子节点
		child = node->rb_left;                 // case1 孩子为节点的左子节点
	else
	{
		struct rb_node *old = node, *left;     // case2 将要删除的节点保存一份

		node = node->rb_right;                 // case2 接下来处理node的右子节点
		while ((left = node->rb_left) != NULL)
			node = left;

		if (rb_parent(old)) {
			if (rb_parent(old)->rb_left == old)
				rb_parent(old)->rb_left = node;
			else
				rb_parent(old)->rb_right = node;    // case2 待删除节点的父节点的右子节点指向待删除节点的右子节点(10号节点的右子节点变为20号节点)
		} else
			root->rb_node = node;

		child = node->rb_right;                     // case2 保存待删除节点的右子节点的右子节点为孩子节点(child=null)
		parent = rb_parent(node);                   // case2 获取node节点的父节点(15号节点)
		color = rb_color(node);                     // case2 获取节点颜色(20号节点的颜色) 

		if (parent == old) {
			parent = node;                           // case2 更新parent的值, 该值为20号节点的值
		} else {
			if (child)
				rb_set_parent(child, parent);
			parent->rb_left = child;

			node->rb_right = old->rb_right;
			rb_set_parent(old->rb_right, node);
		}

		node->rb_parent_color = old->rb_parent_color;    // case2 待删除节点的右子节点的父亲节点指向待删除节点的父亲节点, 并且替换为待删除节点的颜色
		node->rb_left = old->rb_left;                    // case2 待删除节点的右子节点的左子节点指向待删除节点的左子节点
		rb_set_parent(old->rb_left, node);               // case2 设置待删除节点的左子节点的父节点为删除节点的右子节点(即20号节点为13号节点的父节点)

		goto color;                                      // case2 调到去重新设置颜色
	}

	parent = rb_parent(node);                    // case1 获取该节点的父节点
	color = rb_color(node);                      // case1 获取该节点的颜色

	if (child)
		rb_set_parent(child, parent);            // case1 设置孩子的父节点为node节点的父节点
	if (parent)
	{
		if (parent->rb_left == node)             // case1 如果该节点是父节点的左子节点, 则父节点的左子节点更改为孩子节点
			parent->rb_left = child;
		else
			parent->rb_right = child;
	}
	else
		root->rb_node = child;

 color:
	if (color == RB_BLACK)                        // 如果删除的节点是黑色, 则需要重新着色
		__rb_erase_color(child, parent, root);
}
```


- case1: 


![](/images/rbtree/20190919171434225_418722980.png)

上述情况删除节点4, 即删除的节点有左子节点而没有右子节点的情况, 删除过程如下:  

![](/images/rbtree/20190919171602887_1313521257.png)



- case2:


![](/images/rbtree/20190919172616341_1770479770.png)

上述情况删除节点15, 即删除的节点有左右子节点的情况, 删除过程如下:


![](/images/rbtree/20190919180624862_885069581.png)

相应的注释与转换过程都在代码中有标注, 这里就不在过多解析了;  


# 三. 在线演示红黑树

```
https://www.cs.usfca.edu/~galles/visualization/RedBlack.html
```






