---
layout: post
title:  "根据dts匹配对应单板"
date:   2019-09-05
catalog:  true
author: Sourcelink
tags:
    - dts
    - linux

---


# 一. 概述

> 宿主机: ubuntu16.04  
> 开发板: tq-imx6ul  
> 内核版本: linux-4.1.15  

以前最早学习linux驱动时, 使用的是无设备树版本的内核, 内核是通过uboot传来的machine id选择对应的machine_desc, 而现在换成了设备树版本它是如何根据设备树进行对应的machine_desc选择的呢? 前面讲[设备树常用节点和属性](https://sourcelink.top/2019/08/24/dts-standard-node-properites-des/)时候, 说过内核会根据设备树根目录下的compatile去进行machine_desc匹配;   


但有个问题: 设备树中的compatile去和谁匹配, 匹配对象从哪来?   


# 二. 匹配platform


## 2.1 思路

以imx6ul为例:  

> mach-imx6ul.c

```
static const char *imx6ul_dt_compat[] __initconst = {
	"fsl,imx6ul",
	"fsl,imx6ull",
	NULL,
};

DT_MACHINE_START(IMX6UL, "Freescale i.MX6 Ultralite (Device Tree)")    ①
	.map_io		= imx6ul_map_io,
	.init_irq	= imx6ul_init_irq,
	.init_machine	= imx6ul_init_machine,
	.init_late	= imx6ul_init_late,
	.dt_compat	= imx6ul_dt_compat,                                   ②
MACHINE_END
```

> ①: DT_MACHINE_START和MACHINE_END宏用于定义一个machine描述符;  其原型如下:  

```
#define DT_MACHINE_START(_name, _namestr)		\
static const struct machine_desc __mach_desc_##_name	\
 __used							\
 __attribute__((__section__(".arch.info.init"))) = {	\
	.nr		= ~0,				\
	.name		= _namestr,

#define MACHINE_END				\
};
```

编译的时候, 编辑器会把这些 machine descriptor放到一个特殊的段中(**.arch.info.init**), 形成machine描述符的列表;  

> ②: 内核在匹配machine_desc时, 会从字段`.arch.info.init`中取出每个machine_desc中的`.dt_compat`成员与设备树根目录下的compatile属性进行比较; 



## 2.2 匹配流程


从`start_kernel()`开始分析, 涉及到文件如下:  

> init/main.c  
> arch/arm/kernel/setup.c
> arch/arm/kernel/devtree.c
> drivers/of/fdt.c


调用流程大概如下:  

```
start_kernel();
    setup_arch(&command_line);
        setup_machine_fdt(__atags_pointer);
            of_flat_dt_match_machine(mdesc_best, arch_get_next_mach);
                of_flat_dt_match(dt_root, compat);
                    of_fdt_match();
                        of_fdt_is_compatible();
```

![](/images/dts/dts_platform_math.png)


### 2.2.1 setup_arch()


```
void __init setup_arch(char **cmdline_p)
{
	const struct machine_desc *mdesc;

	setup_processor();
	mdesc = setup_machine_fdt(__atags_pointer);
	if (!mdesc)
		mdesc = setup_machine_tags(__atags_pointer, __machine_arch_type);
	machine_desc = mdesc;
	machine_name = mdesc->name;
	dump_stack_set_arch_desc("%s", mdesc->name);
    .....
}
```

该函数会先调用`setup_machine_fdt()`进行设备树匹配, 如果没有匹配成功则会`setup_machine_tags()`进行匹配; 即默认`__atags_pointer`变量指向设备树dtb;  


### 2.2.2 setup_machine_fdt()

```
const struct machine_desc * __init setup_machine_fdt(unsigned int dt_phys)
{
	const struct machine_desc *mdesc, *mdesc_best = NULL;

#ifdef CONFIG_ARCH_MULTIPLATFORM
	DT_MACHINE_START(GENERIC_DT, "Generic DT based system")
	MACHINE_END

	mdesc_best = &__mach_desc_GENERIC_DT;
#endif

	if (!dt_phys || !early_init_dt_verify(phys_to_virt(dt_phys)))
		return NULL;

	mdesc = of_flat_dt_match_machine(mdesc_best, arch_get_next_mach);

	.....

	return mdesc;
}
```

主要调用`of_flat_dt_match_machine()`进行比较, 返回值为匹配成功的`machine_desc`指针;  在调用的时候注册一个回调函数`arch_get_next_mach()`


```
static const void * __init arch_get_next_mach(const char *const **match)
{
	static const struct machine_desc *mdesc = __arch_info_begin;
	const struct machine_desc *m = mdesc;

	if (m >= __arch_info_end)
		return NULL;

	mdesc++;
	*match = m->dt_compat;
	return m;
}
```

该函数主要功能是从`.arch.info.init`字段中提取出`machine_desc`, 并且将其中成员`dt_compat`值传给了形参`match`用于接下来的平台比较, 指针`__arch_info_begin`指向`.arch.info.init`字段的头, 指针`__arch_info_end`指向`.arch.info.init`字段的尾部;  


### 2.2.3 of_flat_dt_match_machine



```
const void * __init of_flat_dt_match_machine(const void *default_match,
		const void * (*get_next_compat)(const char * const**))
{
	const void *data = NULL;
	const void *best_data = default_match;
	const char *const *compat;
	unsigned long dt_root;
	unsigned int best_score = ~1, score = 0;

	dt_root = of_get_flat_dt_root();
	while ((data = get_next_compat(&compat))) {                        ①
		score = of_flat_dt_match(dt_root, compat);                     ②
		if (score > 0 && score < best_score) {                         ③
			best_data = data;
			best_score = score;
		}
	}
	
    ...
	pr_info("Machine model: %s\n", of_flat_dt_get_machine_name());

	return best_data;
}
```

> ①: 一直调用回调函数`arch_get_next_mach()`从`.arch.info.init`字段获取`machine_desc`;    
> ②: 从`.arch.info.init`字段获取`machine_desc`与dtb中的根目录下`compare`属性进行比较;    
> ③: 如果是最佳得分, 即最佳匹配则将`best_data`(machine_desc)返回给调用函数;    


### 2.2.4 of_flat_dt_match


```
int __init of_flat_dt_match(unsigned long node, const char *const *compat)
{
	return of_fdt_match(initial_boot_params, node, compat);
}
```

其中参数`initial_boot_params`存放了dtb文件的起始地址; 

### 2.2.5 of_fdt_match


```
int of_fdt_match(const void *blob, unsigned long node,
                 const char *const *compat)
{
	unsigned int tmp, score = 0;

	if (!compat)
		return 0;

	while (*compat) {
		tmp = of_fdt_is_compatible(blob, node, *compat);
		if (tmp && (score == 0 || (tmp < score)))
			score = tmp;
		compat++;
	}

	return score;
}
```


### 2.2.6 of_fdt_is_compatible


```
int of_fdt_is_compatible(const void *blob,
		      unsigned long node, const char *compat)
{
	const char *cp;
	int cplen;
	unsigned long l, score = 0;

	cp = fdt_getprop(blob, node, "compatible", &cplen);            ①
	if (cp == NULL)
		return 0;
	while (cplen > 0) {
		score++;
		if (of_compat_cmp(cp, compat, strlen(compat)) == 0)        ②
			return score;
		l = strlen(cp) + 1;                                        ③
		cp += l;
		cplen -= l;
	}

	return 0;
}
```

> ①: 获取属性`compatible`中的值;   
> ②: 将获取到属性`compatible`的值与变量`compat`比较, 该值是从`.arch.info.init`字段获取到的`machine_desc`中成员`dt_compat`;   
> 如果比较成功变量`score`则立即返回, 否则进行下一轮比较, `score`的值越大说明匹配度越低;    
> ③: 因为属性`compatible`一般为字符串列表, 所以用strlen是可以计算出字符串长度, 加1是为了跳过逗号;  