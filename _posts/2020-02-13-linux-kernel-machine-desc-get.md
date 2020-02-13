---
layout: post
title:  "linux内核之machine_desc获取"
date:   2020-02-13
catalog:  true
author: Sourcelink
tags:
    - 内核
    - Linux
---

# 一. 概述

> 板子： imx6ul
> 内核版本：linux4.1.15
> 宿主机：ubuntu16.04

machine_desc主要是对板子的一个特性，比如init_machine()和init_irq(), 这些都是它所具有的功能，不同厂家的板子这些初始化是不同的；


# 二. machine_desc的构造

```
DT_MACHINE_START(IMX6UL, "Freescale i.MX6 Ultralite (Device Tree)")
	.map_io		= imx6ul_map_io,
	.init_irq	= imx6ul_init_irq,
	.init_machine	= imx6ul_init_machine,
	.init_late	= imx6ul_init_late,
	.dt_compat	= imx6ul_dt_compat,
MACHINE_END
```

使用了宏`DT_MACHINE_START`对这个machine_desc进行定义；  



## 2.1 宏DT_MACHINE_START

```
#define DT_MACHINE_START(_name, _namestr)		\
static const struct machine_desc __mach_desc_##_name	\
 __used							\
 __attribute__((__section__(".arch.info.init"))) = {	\
	.nr		= ~0,				\
	.name		= _namestr,
```

使用了段属性`__section__(".arch.info.init")`将定义的struct machine_desc类型的变量存储在`.arch.info.init`段中；  


将上面imx6ul的例子展开：

```
static const struct machine_desc __mach_desc_IMX6UL __used __attribute__((__section__(".arch.info.init"))) = {
    	.nr		= ~0,				
	.name	= "Freescale i.MX6 Ultralite (Device Tree)",
    	.map_io		= imx6ul_map_io,
	.init_irq	= imx6ul_init_irq,
	.init_machine	= imx6ul_init_machine,
	.init_late	= imx6ul_init_late,
	.dt_compat	= imx6ul_dt_compat,
};
```


## 2.2 存储方式

> arch/arm/kernel/vmlinux.lds.S

```
	.init.arch.info : {
		__arch_info_begin = .;
		*(.arch.info.init)
		__arch_info_end = .;
	}
```

接下里可以通过__arch_info_begin来遍历machine_desc；  


```
extern const struct machine_desc __arch_info_begin[], __arch_info_end[];
#define for_each_machine_desc(p)			\
	for (p = __arch_info_begin; p < __arch_info_end; p++)
```


# 三. machine_desc的调用

```
start_kernel(void)
    setup_arch(&command_line);
        setup_machine_fdt();
```

## 3.1 setup_machine_fdt

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
}

```

调用 of_flat_dt_match_machine() 做匹配， 匹配成功会返回一个描述符；  

## 3.2 of_flat_dt_match_machine

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
    while ((data = get_next_compat(&compat))) {                            ①
		score = of_flat_dt_match(dt_root, compat);
		if (score > 0 && score < best_score) {                             ②
			best_data = data;
			best_score = score;
		}
	}
	if (!best_data) {
		const char *prop;
		int size;

		pr_err("\n unrecognized device tree list:\n[ ");

		prop = of_get_flat_dt_prop(dt_root, "compatible", &size);
		if (prop) {
			while (size > 0) {
				printk("'%s' ", prop);
				size -= strlen(prop) + 1;
				prop += strlen(prop) + 1;
			}
		}
		printk("]\n\n");
		return NULL;
	}

	pr_info("Machine model: %s\n", of_flat_dt_get_machine_name());

	return best_data;
}
```

> ①：通过函数get_next_compat指针来获取数据进行比较，他的比较对象是设备树根目录下compatible属性的内容

看下函数指针：

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

通过与dt_compat的值进行比较， 

imx6ul_dt_compa的值如下t：

```
static const char *imx6ul_dt_compat[] __initconst = {
	"fsl,imx6ul",
	"fsl,imx6ull",
	NULL,
};
```

> ②：如果是最佳得分, 即最佳匹配则将best_data(machine_desc)返回给调用函数

