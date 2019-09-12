---
layout: post
title:  "dtb信息转化为device_node结构"
date:   2019-09-10
catalog:  true
author: Sourcelink
tags:
    - dts
    - unflatten_device_tree

---



# 一. 概述

> 宿主机: ubuntu16.04  
> 开发板: tq-imx6ul  
> 内核版本: linux-4.1.15  

内核在启动的时候会读取dtb文件, 将dtb中的信息提取出来转换成对应的`device_node`, 接下来分析了下`device_node`等数据结构含义和实现过程;  


# 二. 结构体分析

## 2.1 struct device_node

Device Tree中的每一个node节点经过kernel处理都会生成一个struct device_node的结构体, struct device_node最终一般会被挂接到具体的struct device结构体。struct device_node结构体描述如下: 

```
struct device_node {
	const char *name;             /* name属性的值, 没有为<NULL> */
	const char *type;             /* device_type属性的值，没有为<NULL> */
	phandle phandle;              /* 可用于其他节点引用的标记 */
	const char *full_name;        /* 节点名称 */
	struct fwnode_handle fwnode;

	struct	property *properties; /*  指向该设备节点下的第一个属性，其他属性与该属性链表相接 */
	struct	property *deadprops;  /* removed properties */
	struct	device_node *parent;  /* 设备的父节点 */
	struct	device_node *child;   /* 设备子节点 */
	struct	device_node *sibling; /* 设备兄弟节点 */
	struct	kobject kobj;
	unsigned long _flags;
	void	*data;
};
```



## 2.2 struct property

kernel会根据Device Tree中所有的属性解析出数据填充`struct property`结构体。struct property结构体描述如下:   

```
struct property {
	char	*name;                    /* property full name */
	int	length;                   /* property value length */
	void	*value;                   /* property value */
	struct property *next;        /* next property under the same node */
	unsigned long _flags;
	unsigned int unique_id;
	struct bin_attribute attr;
};
```

kernel根据Device Tree的文件结构信息转换成struct property结构体, 并将同一个node节点下面的所有属性通过property.next指针进行链接, 形成一个单链表。



# 三. 调用流程分析


大致调用流程如下:  

![](/images/dts/unflatten_device_tree_call.png)


Device Tree的解析首先从unflatten_device_tree()开始，代码列出如下：

## 3.1 unflatten_device_tree

```
void __init unflatten_device_tree(void)
{
	__unflatten_device_tree(initial_boot_params, &of_root,
				early_init_dt_alloc_memory_arch);

	/* Get pointer to "/chosen" and "/aliases" nodes for use everywhere */
	of_alias_scan(early_init_dt_alloc_memory_arch);
}
```

参数`initial_boot_params`为dtb文件的起始地址, `of_root`用于保存device_node的根节点, 以后可以通过该变量找到所有的设备节点, `early_init_dt_alloc_memory_arch`该参数为回调函数用于分配内存给设备节点使用;  


## 3.2 __unflatten_device_tree

```
static void __unflatten_device_tree(void *blob,
			     struct device_node **mynodes,
			     void * (*dt_alloc)(u64 size, u64 align))
{
	unsigned long size;
	int start;
	void *mem;

	pr_debug(" -> unflatten_device_tree()\n");

	if (!blob) {
		pr_debug("No device tree pointer\n");
		return;
	}

	if (fdt_check_header(blob)) {
		pr_err("Invalid device tree blob header\n");
		return;
	}

	/* First pass, scan for size */
	start = 0;
	size = (unsigned long)unflatten_dt_node(blob, NULL, &start, NULL, NULL, 0, true);        ①
	size = ALIGN(size, 4);

	pr_debug("  size is %lx, allocating...\n", size);

	/* Allocate memory for the expanded device tree */
	mem = dt_alloc(size + 4, __alignof__(struct device_node));                               ②
	memset(mem, 0, size);

	*(__be32 *)(mem + size) = cpu_to_be32(0xdeadbeef);

	pr_debug("  unflattening %p...\n", mem);

	/* Second pass, do actual unflattening */
	start = 0;
	unflatten_dt_node(blob, mem, &start, NULL, mynodes, 0, false);                           ③
	if (be32_to_cpup(mem + size) != 0xdeadbeef)
		pr_warning("End of tree marker overwritten: %08x\n",
			   be32_to_cpup(mem + size));

	pr_debug(" <- unflatten_device_tree()\n");
}
```

> ①: 用于计算Device Tree转换成struct device_node和struct property结构体需要分配的内存大小;  
> ②:  调用回调函数根据unflatten_dt_node()计算出的size分配内存;  
> ③:  填充每一个struct device_node和struct property结构体;  

注意: 这个函数中调用了两次`unflatten_dt_node()`, 但是两次调用的含义是不同的;  

## 3.3 unflatten_dt_node

```
static void * unflatten_dt_node(void *blob,
				void *mem,
				int *poffset,
				struct device_node *dad,
				struct device_node **nodepp,
				unsigned long fpsize,
				bool dryrun)
{
	const __be32 *p;
	struct device_node *np;
	struct property *pp, **prev_pp = NULL;
	const char *pathp;
	unsigned int l, allocl;
	static int depth = 0;
	int old_depth;
	int offset;
	int has_name = 0;
	int new_format = 0;

	pathp = fdt_get_name(blob, *poffset, &l);                                ①
	if (!pathp)
		return mem;

	allocl = ++l;

	if ((*pathp) != '/') {                                                   ②
		new_format = 1;
		if (fpsize == 0) {

			fpsize = 1;
			allocl = 2;
			l = 1;
			pathp = "";
		} else {
			/* account for '/' and path size minus terminal 0
			 * already in 'l'
			 */
			fpsize += l;
			allocl = fpsize;
		}
	}

	np = unflatten_dt_alloc(&mem, sizeof(struct device_node) + allocl,        ③
				__alignof__(struct device_node));
	if (!dryrun) {
		char *fn;
		of_node_init(np);
		np->full_name = fn = ((char *)np) + sizeof(*np);                      ④
		if (new_format) {
			/* rebuild full path for new format */
			if (dad && dad->parent) {
				strcpy(fn, dad->full_name);
				fn += strlen(fn);
			}
			*(fn++) = '/';
		}
		memcpy(fn, pathp, l);                                                 ④

		prev_pp = &np->properties;                                            ⑤
		if (dad != NULL) {
			np->parent = dad;
			np->sibling = dad->child;
			dad->child = np;
		}
	}
	/* process properties */
	for (offset = fdt_first_property_offset(blob, *poffset);                  ⑥
	     (offset >= 0);
	     (offset = fdt_next_property_offset(blob, offset))) {
		const char *pname;
		u32 sz;

		if (!(p = fdt_getprop_by_offset(blob, offset, &pname, &sz))) {
			offset = -FDT_ERR_INTERNAL;
			break;
		}

		if (pname == NULL) {
			pr_info("Can't find property name in list !\n");
			break;
		}
		if (strcmp(pname, "name") == 0)
			has_name = 1;
		pp = unflatten_dt_alloc(&mem, sizeof(struct property),
					__alignof__(struct property));
		if (!dryrun) {
			if ((strcmp(pname, "phandle") == 0) ||                           ⑦
			    (strcmp(pname, "linux,phandle") == 0)) {
				if (np->phandle == 0)
					np->phandle = be32_to_cpup(p);
			}
			if (strcmp(pname, "ibm,phandle") == 0)
				np->phandle = be32_to_cpup(p);
			pp->name = (char *)pname;
			pp->length = sz;
			pp->value = (__be32 *)p;
			*prev_pp = pp;
			prev_pp = &pp->next;
		}
	}

	if (!has_name) {                                                         ⑧
		const char *p1 = pathp, *ps = pathp, *pa = NULL;
		int sz;

		while (*p1) {
			if ((*p1) == '@')
				pa = p1;
			if ((*p1) == '/')
				ps = p1 + 1;
			p1++;
		}
		if (pa < ps)
			pa = p1;
		sz = (pa - ps) + 1;
		pp = unflatten_dt_alloc(&mem, sizeof(struct property) + sz,
					__alignof__(struct property));
		if (!dryrun) {
			pp->name = "name";
			pp->length = sz;
			pp->value = pp + 1;
			*prev_pp = pp;
			prev_pp = &pp->next;
			memcpy(pp->value, ps, sz - 1);
			((char *)pp->value)[sz - 1] = 0;
			pr_debug("fixed up name for %s -> %s\n", pathp,
				(char *)pp->value);
		}
	}
	if (!dryrun) {                                                            
		*prev_pp = NULL;
		np->name = of_get_property(np, "name", NULL);
		np->type = of_get_property(np, "device_type", NULL);

		if (!np->name)
			np->name = "<NULL>";
		if (!np->type)
			np->type = "<NULL>";
	}

	old_depth = depth;
	*poffset = fdt_next_node(blob, *poffset, &depth);                         ⑨
	if (depth < 0)
		depth = 0;
	while (*poffset > 0 && depth > old_depth)
		mem = unflatten_dt_node(blob, mem, poffset, np, NULL,                 ⑩
					fpsize, dryrun);

	if (*poffset < 0 && *poffset != -FDT_ERR_NOTFOUND)
		pr_err("unflatten: error %d processing FDT\n", *poffset);

	if (!dryrun && np->child) {
		struct device_node *child = np->child;
		np->child = NULL;
		while (child) {
			struct device_node *next = child->sibling;
			child->sibling = np->child;
			np->child = child;
			child = next;
		}
	}

	if (nodepp)
		*nodepp = np;

	return mem;
}
```


> ①: 调用fdt_get_name()获取该节点名称, 该函数实现比较节点, 读者可以根据前面的设备树dtb文件分析笔记结合分析下;  
> ②: 特殊处理根节点将要分配的内存大小;  
> ③: 从mem中为该设备节点分配内存, size为sizeof(struct device_node) + allocl, allocl为节点名字长度;  
> ④: 成员full_name和变量 fn都指向该设备节点data成员, 接着将节点名字赋值给fn变量, 即如下图所示:   

![](/images/dts/dts_device_node.png)

做法猜想: 因为分配内存时, 成员是个full_name是个指针占四个字节, 但是节点名字长度不是固定的, 所以先计算了分配所有设备节点需要内存大小, 但是成员的地址已经固定, 所以只能将节点名字(字符串)放在这段内存的最后;  

以上只是个人猜想, 如果把成员full_name直接放最后不就没那么多事了吗?   

> ⑤: 记录下属性指针, 并将设备几点挂载到对应的父节点和兄弟节点;  
> ⑥: 处理该设备节点下的所有属性;  
> ⑦: 判断是否有无phandle属性, 有则保存下来;  
> ⑧: 如果该节点下没有name属性, 则为其添加一个name属性, 对应的值为设备节点名; 有两种情况:  

- example1: 

```
pmu {
	compatible = "arm,cortex-a7-pmu";
	interrupts = <0x0 0x5e 0x4>;
	status = "disabled";
};
```

则添加一个name属性为: `name = pmu;`

- example2:

```
dma-apbh@01804000 {
			compatible = "fsl,imx6ul-dma-apbh", "fsl,imx28-dma-apbh";
			reg = <0x1804000 0x2000>;
			interrupts = <0x0 0xd 0x4 0x0 0xd 0x4 0x0 0xd 0x4 0x0 0xd 0x4>;
			interrupt-names = "gpmi0", "gpmi1", "gpmi2", "gpmi3";
			#dma-cells = <0x1>;
			dma-channels = <0x4>;
			clocks = <0x1 0x80>;
			linux,phandle = <0x6>;
			phandle = <0x6>;
};
```

则添加一个name属性为: `name = dma-apbh;`


> ⑨: 获取下一个节点指针;  
> ⑩: 递归剩下的所有节点, 无论是该设备节点下的还是兄弟节点;  


# 四. 参考图

## 4.1 参考dts

```
/dts-v1/;

/memreserve/ 0x9ff00000 0x100000;

/ {
	#address-cells = <0x1>;
	#size-cells = <0x1>;
	model = "Freescale i.MX6 UltraLite 14x14 EVK Board";
	compatible = "fsl,imx6ul-14x14-evk", "fsl,imx6ul";

	
	chosen {
	 bootargs = "noinitrd root=/dev/mtdblock4 rw init=/linuxrc console=ttySAC0,115200";
	};

	memory {
		device_type = "memory";
		reg = <0x80000000 0x20000000>;
	};

	leds {
		compatible = "gpio-leds";
		pinctrl-names = "default";
		pinctrl-0 = <0x3f>;

		led1 {
			label = "LED1";
			gpios = <0x3b 0x7 0x1>;
		};

		led2 {
			label = "LED2";
			gpios = <0x3b 0x2 0x1>;
		};
	};
};
```


## 4.2 对应的设备节点图

![](/images/dts/dts_devices_node_struct.png)



