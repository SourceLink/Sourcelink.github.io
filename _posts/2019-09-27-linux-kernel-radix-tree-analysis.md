---
layout: post
title:  "详解Linux内核Radix树算法的实现"
date:   2019-09-26
catalog:  true
author: Sourcelink
tags:
    - linux 
    - kernel
    - radix
    - tree

---


# 一. 概述

> 宿主机: ubuntu16.04  
> 内核版本: linux-4.19  

 $\quad\quad$关于linux的Radix Tree我从网上看了许多文章, 但是都看的不明不白, 后面通过分析源码才慢慢明白, 在内核中利用 Radix Tree 可以根据一个长整型 (比如一个长 ID) 快速查找到其对应的对象指针, 可以说是一个key-value的方式插入和查找方式;  


## 1.1 radix_tree_node
 $\quad\quad$Linux 内核中, 使用 struct radix_tree_node 数据结构定义了一个 radix-tree 的节点, 节点包含了多个成员, 原型如下:  

```
struct radix_tree_node {
	unsigned char	shift;		/* Bits remaining in each slot */
	unsigned char	offset;		/* Slot offset in parent */
	unsigned char	count;		/* Total entry count */
	unsigned char	exceptional;	/* Exceptional entry count */
	struct radix_tree_node *parent;		/* Used when ascending tree */
	struct radix_tree_root *root;		/* The tree we belong to */
	union {
		struct list_head private_list;	/* For tree user */
		struct rcu_head	rcu_head;	/* Used when freeing node */
	};
	void __rcu	*slots[RADIX_TREE_MAP_SIZE];
	unsigned long	tags[RADIX_TREE_MAX_TAGS][RADIX_TREE_TAG_LONGS];
};
```


>  shift成员用于指向当前节点slots的单位;   
>  offset表示该节点在父节点的slot中偏移值;   
>  count 表示当前节点有多少个slot已经被使用;   
>  exceptional表示当前节点有多少个exceptional节点;   
>  parent 指向父节点; 参数 root 指向根节点;   
>  slots是个指针数组, 该数组既可以存储下一级的节点, 也可以用于存储即将插入的对象指针(接下来都用item代理说明);   
>  tags 用于标识当前节点包含了指定 tag 的节点数;   


 $\quad\quad$这里需要重点讲解下shift, 为什么说它是表示指向当前节点所占用的偏移呢?  以水表的表盘来讲解下, 表盘如下, 该表盘有四个小盘, 分别代表了不同的档位, 当`X0.0001`的盘转满一圈, 则表`X0.001`会转动一刻, 以此类推超出了对应的刻度则会向更上一级计量的表盘上**进位**; 
 
 ![](/images/radix-tree/20190927094113893_1883486795.png)

 $\quad\quad$那接下来说下这个shift是怎么回事, 首先要明白我们探讨的是一颗树, 树的节点上下是有层次, 就好比这个表盘, 树越上的节点则代表了更大的计量;  前面说了 radix_tree_node 节点维护了一个指针数组slots用于插入对应的item或节点, 假设现在每个slots的长度为16, 虽然每个节点的slots的长度刻度相同, 但是每个节点的层次关系有可能不同, 则导致节点slots的**单位**不同;  可以把它想象成上面的水表的表盘, 如果有的节点成员**shift等于0表示该节点slots的单位为$2^0$, 即等于1; 如果有的节点成员变量shift等于4则表示该节点slots的单位为$2^4$, 即等于16; 如果有的节点成员变量shift等于8, 则表示该节点slots的单位为$2^8$, 即为256;** 这样是不是和水表表盘的计量方式一样, 而刚举例的三个节点之间的关系一定是一级级的;

 $\quad\quad$因为内核是根据一个长整型ID进行对应的item插入, 现在分别插入两个ID为12和32的item, 当插入ID为12的item时, 创建一个新节点可以直接插入slots偏移为12的位置, 而当插入ID为32的item时, 发现这个节点已经无法插入了, 按照上面分析的思路, 我们应该创建一个新的节点, 而且单位更大那种进行接下来的插入; 这里只是简单的举个例子说明内核Radix树slots是如何根据ID进行对应的item插入, 接下会详细分析;  


## 1.2 radix_tree_root

```
struct radix_tree_root {
        spinlock_t              xa_lock;
        gfp_t                   gfp_mask;
        struct radix_tree_node  __rcu *rnode;
};
```

Linux 内核中, 使用 struct radix_tree_root 维护 radix-tree 的根节点; xa_lock 是一个自旋锁; 参数 gfp_mask 用于标识 radix-tree 的属性以及 radix-tree 节点申请 内核的标识; 参数 rnode 指向 radix-tree 的根节点; 


# 二. 详细分析


## 2.1 插入原理 


### 2.1.1 radix_tree_insert

```
static inline int radix_tree_insert(struct radix_tree_root *root,
			unsigned long index, void *entry)
{
	return __radix_tree_insert(root, index, 0, entry);
}
```


```
int __radix_tree_insert(struct radix_tree_root *root, unsigned long index,
			unsigned order, void *item)
{
	struct radix_tree_node *node;
	void **slot;
	int error;

	BUG_ON(radix_tree_is_internal_node(item));

	error = __radix_tree_create(root, index, order, &node, &slot);            ①
	if (error)
		return error;

	error = insert_entires(node, slot, item, order, false);                   ②
	if (error < 0)
		return error;

	if (node) {
		unsigned offset = get_slot_offset(node, slot);
		BUG_ON(tag_get(node, 0, offset));
		BUG_ON(tag_get(node, 1, offset));
		BUG_ON(tag_get(node, 2, offset));
	} else {
		BUG_ON(root_tags_get(root));
	}
	return 0;
}
```

> ①: 根据待插入item的ID值(index), 创建对应的树节点;  
> ②: 插入item到对应节点的slots中;  

example: 

```
static struct radix_tree_root _root;
static struct node node0 = { .name = "Sourcelink", .id = 32 };
INIT_RADIX_TREE(&_root, GFP_ATOMIC);
radix_tree_insert(&_root, node0.id, &node0);
```

如上是使用id值对应插入`node0`变量的地址;


### 2.1.2 __radix_tree_create


```
int __radix_tree_create(struct radix_tree_root *root, unsigned long index,
			unsigned order, struct radix_tree_node **nodep,
			void ***slotp)
{
	struct radix_tree_node *node = NULL, *child;
	void **slot = (void **)&root->rnode;
	unsigned long maxindex;
	unsigned int shift, offset = 0;
	unsigned long max = index | ((1UL << order) - 1);
	gfp_t gfp = root_gfp_mask(root);

	shift = radix_tree_load_root(root, &child, &maxindex);                  ①

	/* Make sure the tree is high enough. */
	if (order > 0 && max == ((1UL << order) - 1))
		max++;
	if (max > maxindex) {
		int error = radix_tree_extend(root, gfp, max, shift);            ②
		if (error < 0)
			return error;
		shift = error;
		child = root->rnode;
	}

	while (shift > order) {
		shift -= RADIX_TREE_MAP_SHIFT;
		if (child == NULL) {                                             ③
			/* Have to add a child node. */
			child = radix_tree_node_alloc(gfp, node, root, shift,
							offset, 0, 0);
			if (!child)
				return -ENOMEM;
			*slot = node_to_entry(child);
			if (node)
				node->count++;
		} else if (!radix_tree_is_internal_node(child))
			break;

		/* Go a level down */
		node = entry_to_node(child);
		offset = radix_tree_descend(node, &child, index);                ④
		slot = &node->slots[offset];
	}

	if (nodep)                                                               ⑤
		*nodep = node;
	if (slotp)
		*slotp = slot;
	return 0;
}
```

> ①: 该函数主要为了计算当前树的最大shift的为多少, 即目前可以计量的单位为多大(侧面反应了树的深度), maxindex变量保存当前树能存储ID值范围;  
> ②: 如果当前的ID值已经超过了当前树所能计量的范围, 则需要调用该函数拓展树的深度, 使其可以顺利插入item;  
> ③: 在shift>order的情况, 如果子节点为空则继续创建节点(新创建的节点会记录该节点shift值和在其父节点的offset值), 并将其挂slot下(*slot = node_to_entry(child););  
> ④: 根据index获取node节点slots对应位置的值给child, 并返回了该位置的offset;  
> ⑤: 形参记录下要插入的节点和slot地址;  


 $\quad\quad$前面看过node的结构体, 知道它的成员slots大小与宏`RADIX_TREE_MAP_SIZE`有关, 该宏定于如下:  

```
#define RADIX_TREE_MAP_SHIFT	(CONFIG_BASE_SMALL ? 4 : 6)
#define RADIX_TREE_MAP_SIZE	(1UL << RADIX_TREE_MAP_SHIFT)
```

$\quad\quad$可以说这个slots的大小取决了`RADIX_TREE_MAP_SHIFT`的大小, 假设没有定义`CONFIG_BASE_SMALL`, 则`RADIX_TREE_MAP_SHIFT`等于6, `RADIX_TREE_MAP_SIZE`则等于$2^6$; 即每个节点的slots可以插入64个节点或item;  因为刚开始插入`root->rnode`节点还为空 , 则无需通过`radix_tree_extend()`扩展深度, 直接通过第③步直接创建节点,  大概图解如下:  


![](/images/radix-tree/20190926221056449_299958990.png)


PS: slots是 radix_tree_node_1 节点的成员;  


 $\quad\quad$ 假设现在有个ID为32的item需要插入, 再经过第④步的计算, 将其插入`radix_tree_node_1`节点的slots的**offset为32**的位置, 大致图解如下:  

![](/images/radix-tree/20190927090057846_1023486693.png)
 

 $\quad\quad$如果现在有个ID为64的item需要插入, 则需要再**新增扩展树的深度**, 前面说过该树和水表计算方式一样,   新增节点的方式大致图解如下:  

![](/images/radix-tree/20190927090312198_1715305518.png)


 $\quad\quad$可以看到为了新插入的item需要合适的位置插入, 而新增了两个节点`radix_tree_node_2`和`radix_tree_node_3`, `radix_tree_node_3`节点则插入`radix_tree_node_2`节点slots的offset为1的位置, item则插入`radix_tree_node_3`节点slots的offset为0的位置;  `radix_tree_node_2`节点的成员shift值等于6, 相当于一个单位为64计算的节点, 只要ID的范围在$2^{12}$内都无需再扩展树的深度, 只需在对应的slot挂载节点后插入item;  可以说每个节点的单位和shift密切关系, 计算公式为: $2^{shift}$;  

再举个例子, 插入一个ID为1596的item, 图解如下(slots长度请无视, 注意文字表示):  

![](/images/radix-tree/20190927151222628_681916401.png)

目前该树的状态, 最大插入范围应该为$64*64 - 1$ == $2^{12}-1$, 即4095; 

### 2.1.3 insert_entires

```
static inline int insert_entires(struct radix_tree_node *node,
		void **slot, void *item, unsigned order, bool replace)
{
	if (*slot)
		return -EEXIST;
	*slot = item;
	if (node) {
		node->count++;
		if (radix_tree_exceptional_entry(item))
			node->exceptional++;
	}
	return 1;
}
```

这个函数功能比较简单, 主要是将item指针挂载到对应slot的上, 而且对当前节点中slot使用情况进行了计数;  


## 2.2 查找原理

### 2.2.1 radix_tree_lookup

```
void *radix_tree_lookup(const struct radix_tree_root *root, unsigned long index)
{
	return __radix_tree_lookup(root, index, NULL, NULL);
}
```

```
void *__radix_tree_lookup(const struct radix_tree_root *root,
			  unsigned long index, struct radix_tree_node **nodep,
			  void ***slotp)
{
	struct radix_tree_node *node, *parent;
	unsigned long maxindex;
	void **slot;

restart:
	parent = NULL;
	slot = (void **)&root->rnode;
	radix_tree_load_root(root, &node, &maxindex);
	if (index > maxindex)                                            ①
		return NULL;

	while (radix_tree_is_internal_node(node)) {
		unsigned offset;

		if (node == RADIX_TREE_RETRY)
			goto restart;
		parent = entry_to_node(node);
		offset = radix_tree_descend(parent, &node, index);       ②
		slot = parent->slots + offset;
	}
	if (nodep)
		*nodep = parent;
	if (slotp)
		*slotp = slot;
	return node;
}
```

> ①: 首先是根据ID进行对应item查找, 计算了当前树最大ID范围, 如果要查找的ID值超过该范围, 则说明该ID对应item不在该树中;  
> ②: 根据index从父节点开始获取存储了item的叶子节点node(通过while循环), 因为前面插入分析可以知道item都存储在叶子节点的slots中;  

当最后找到了对应item后, 经过判断发现已经不是树的节点则退出循环;  

### 2.2.2 radix_tree_descend


```
static unsigned int radix_tree_descend(const struct radix_tree_node *parent,
			struct radix_tree_node **nodep, unsigned long index)
{
	unsigned int offset = (index >> parent->shift) & RADIX_TREE_MAP_MASK;
	void **entry = parent->slots[offset];

	*nodep = (void *)entry;
	return offset;
}
```

这里重点讲解offset的计算, 对index进行了一个移位运算, 我们前面分析过shift相当于一个单位值, 与上`RADIX_TREE_MAP_MASK`是为了确保offset在范围内; 

![](/images/radix-tree/20190927151222628_681916401.png)

example: 

现查找index值是1596的item, 父节点的shift为6时, 则计算出的offset为24, 获取该父节点slots偏移为24的值, 发现还是一个节点, 再次计算偏移, 1596& 63后得到值为60,  获取该父节点slots偏移为60的值, 发现该值不是一个节点, 则退出;  


## 2.3 删除原理

### 2.3.1 radix_tree_delete

```
void *radix_tree_delete(struct radix_tree_root *root, unsigned long index)
{
	return radix_tree_delete_item(root, index, NULL);
}
```

```
void *radix_tree_delete_item(struct radix_tree_root *root,
				unsigned long index, void *item)
{
	struct radix_tree_node *node = NULL;
	void **slot = NULL;
	void *entry;

	entry = __radix_tree_lookup(root, index, &node, &slot);                ①
	if (!slot)
		return NULL;
	if (!entry && (!is_idr(root) || node_tag_get(root, node, IDR_FREE,     ②
						get_slot_offset(node, slot))))
		return NULL;

	if (item && entry != item)
		return NULL;

	__radix_tree_delete(root, node, slot);

	return entry;
}
```

> ①: 通过 index调用__radix_tree_lookup()查找到对应的item, 返回值即是;  
> ②: 如果entry存在则继续向下执行删除函数;  
> ③:  根据节点地址和对应的slot地址删除该item;  

### 2.3.2 __radix_tree_delete

```
static bool __radix_tree_delete(struct radix_tree_root *root,
				struct radix_tree_node *node, void **slot)
{
	void *old = *slot;
	int exceptional = radix_tree_exceptional_entry(old) ? -1 : 0;
	unsigned offset = get_slot_offset(node, slot);                         ①
	int tag;

	if (is_idr(root))
		node_tag_set(root, node, IDR_FREE, offset);
	else
		for (tag = 0; tag < RADIX_TREE_MAX_TAGS; tag++)
			node_tag_clear(root, node, tag, offset);

	replace_slot(slot, NULL, node, -1, exceptional);                       ②
	return node && delete_node(root, node, NULL);                          ③
}
```

> ①: 根据slot地址获取在slots中的偏移值;  
> ②: 更换slot, 该函数的第二个参数表示item, 赋值为NULL, 表示清空了该slot下记录的item;  并且对slot计数进行减一操作; 
> ③: 删除node, 如果节点slots已经没有挂载item则需要进行删除操作;  


### 2.3.3 delete_node

```
static bool delete_node(struct radix_tree_root *root,
			struct radix_tree_node *node,
			radix_tree_update_node_t update_node)
{
	bool deleted = false;

	do {
		struct radix_tree_node *parent;

		if (node->count) {                                                 ①
			if (node_to_entry(node) == root->rnode)
				deleted |= radix_tree_shrink(root,
								update_node);
			return deleted;
		}

		parent = node->parent;                                             ②
		if (parent) {
			parent->slots[node->offset] = NULL;
			parent->count--;
		} else {
			/*
			 * Should't the tags already have all been cleared
			 * by the caller?
			 */
			if (!is_idr(root))
				root_tag_clear_all(root);
			root->rnode = NULL;
		}

		radix_tree_node_free(node);                                        ③
		deleted = true;

		node = parent;                                                     ④
	} while (node);

	return deleted;
}
```

> ①: 如果该节点的slots上还有item则, 判断该节点是否为root节点, 不是则立即返回, 否则将基数缩小到对小高度;  
> ②: 根据该节点的信息, 对其父节点slots上对应的位置进行清零并进行计数减一操作;  
> ③: 释放该节点内存;  
> ④: 接下来对删除节点其父节点进行操作(因为它父节点slots也没有挂有其他节点也需要删除);  

### 2.3.4 radix_tree_shrink

```
static inline bool radix_tree_shrink(struct radix_tree_root *root,
				radix_tree_update_node_t update_node)
{
	bool shrunk = false;

	for (;;) {
		struct radix_tree_node *node = root->rnode;
		struct radix_tree_node *child;

		if (!radix_tree_is_internal_node(node))
			break;
		node = entry_to_node(node);

		if (node->count != 1)                                            ①
			break;
		child = node->slots[0];                                          ②
		if (!child)
			break;
		if (!radix_tree_is_internal_node(child) && node->shift)
			break;

		if (radix_tree_is_internal_node(child))
			entry_to_node(child)->parent = NULL;

		root->rnode = (void *)child;
		if (is_idr(root) && !tag_get(node, IDR_FREE, 0))
			root_tag_clear(root, IDR_FREE);

		node->count = 0;
		if (!radix_tree_is_internal_node(child)) {                      ③
			node->slots[0] = (void *)RADIX_TREE_RETRY;
			if (update_node)
				update_node(node);
		}

		radix_tree_node_free(node);
		shrunk = true;
	}

	return shrunk;
}
```

该函数主要功能是为了缩减树的高度, 去除一些没有使用的节点; 打个比方现在插入一个ID值很大的item, 比如0xf2e32223, 则树深度绝对就不止两层了, 当把该item删除, 则需要把树的深度将下来;  


> ①: 如果该节点只有一个slot被使用, 有可能是挂载了节点, 继续执行;  
> ②: 如果该节点slots[0]的值不是挂载子节点的, 则退出;  
> ③: 对节点进行清零操作;  

像这种情况是需要进行树深度缩减的:  

![](/images/radix-tree/20190927172946606_1607208940.png)

