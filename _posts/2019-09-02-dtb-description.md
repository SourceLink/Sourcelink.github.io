---
layout: post
title:  "设备树dtb文件分析"
date:   2019-09-02
catalog:  true
author: Sourcelink
tags:
    - dts
    - dtb

---


# 一. 概述

> 宿主机: ubuntu16.04  
> 开发板: tq-imx6ul  
> 内核版本: linux-4.1.15  

前面已经分析过dts文件的格式, 这篇笔记将记录下dtb文件的分析过程; dtb文件是一个二进制文件, 我们需要将他转换成16进制文件进行查看;  
笔者使用的是vscode进行查看, 安装了插件**hexdump**;  

为了方便讲解我自己写了一个简洁的dts方便接下来进行dtb分析, 内容如下:  


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

使用`dtc`命令将dts文件编译成dtb文件:  

```
dtc -I dts -O dtb -o imx6ul.dtb imx6ul.dts
```

详细命令请参考[手动编译dts](https://sourcelink.top/2019/08/22/dts-compare/)的笔记;  


# 二.  dtb数据结构


dtb文件的内容分布格式如下图:  

![](/images/dts/dtb_struct_des.png)


主要的四点有头部信息, 保留内存信息, 结构信息和字符串信息;  


|           块            |                                                 描述                                                 |
| ----------------------- | --------------------------------------------------------------------------------------------------- |
| struct ftd_header        | dtb文件头节点                                                                                         |
| memory reservation block | 保存的是一个保留内存映射列表                                                                              |
| structure block          | 设备树中的各节点的信息会保存在这                                                                           |
| Strings Block            | 由于某些属性(比如compatible)在大多数节点下都会存在,<br>为了减少dtb文件大小，就需要把这些属性字符串只指定一个存储位置即可 |


## 2.1 ftd_header


> scripts/dtc/libfdt/fdt.h


```
struct fdt_header {
    uint32_t magic;              // 固定头
    uint32_t totalsize;          // dtb文件大小
    uint32_t off_dt_struct;      // struct block偏移值
    uint32_t off_dt_strings;     // string block偏移值
    uint32_t off_mem_rsvmap;     // mem_rsvmap偏移值
    uint32_t version;            // 设备树版本
    uint32_t last_comp_version;  // 向下兼容版本号
    uint32_t boot_cpuid_phys;    // cpu的物理id
    uint32_t size_dt_strings;    // string block大小
    uint32_t size_dt_struct;     // struct block大小
};
```


struct fdt_header结构体包含了Device Tree的私有信息。例如: fdt_header.magic固定值为0xd00dfeed，fdt_header.totalsize是fdt文件的大小。  
使用二进制工具打开 imx6ul.dtb验证。 imx6ul.dtb二进制文件头信息如下图所示。从下图中可以得到Device Tree的文件是以**大端模式储存**。  

![](/images/dts/dtb_head.png)


查看下文件大小:  

```
➜ ls -al imx6ul.dtb
-rw-rw-r-- 1 sourcelink sourcelink 677 8月  30 15:16 imx6ul.dtb
```

十进制677对应十六进制0x02A5;  

如下图笔者使用框图进行数据标注:   

![](/images/dts/dtb_head_lable.png)

结合前面贴出的dtb文件的内容分布格式, 可以得到下图:  

![](/images/dts/dtb_file_struct.png)


## 2.2 memory reservation block

这个区域包括了若干的reserve memory描述符。每个reserve memory描述符是由address和size组成。其中address和size都是用U64来描述。

```
struct fdt_reserve_entry {
    uint64_t address;
    uint64_t size;
};
```

根据`off_mem_rsvmap`的值0x28, 查找对应的dtb文件区域发现其值为`0x9ff00000 0x100000`, 与前面dts设置的保留内存是相同的`/memreserve/ 0x9ff00000 0x100000;`


## 2.3 structure block

设备树结构块是一个线性化的结构体, 是设备树的主体, 以节点的形式保存了主板上的设备信息。

其中使用`FDT_BEGIN_NODE`表示节点的开始(其值为0x00000001), `FDT_END_NODE`表示节点的结束(其值为0x00000002), `FDT_PROP`表示一个属性(0x00000003),  `FDT_END`表示这个struct block结尾(0x00000009);  

笔者对前面的dts文件做了个标记, 如下:  

![](/images/dts/dtb_led_node_des.png)


在标记`FDT_BEGIN_NODE`开始后, 后面的值为该节点名字;  标记`FDT_PROP`后面按理应该是~~属性名在加属性值~~, 但我们前面说过所有的属性string都存储在string block中, 则有以下结构体进行描述:  

```
struct {
    uint32_t len;
    uint32_t nameoff;
}
```

其中`len`表示属性值的长度, `nameoff`表示该属性的名字存储在string block中的偏移值, 所以得出的数据结构如下:   

![](/images/dts/dtb_prop_struct.png)


**Example:**


以根目录下的`#address-cells`属性为例:

![](/images/dts/dtb_addresscells_struct_example.png)


在string block中的存储的值:  

![](/images/dts/dtb_addresscells_string_example.png)

其中还可以看FDT_END表示设备树结构结束了;  

![](/images/dts/dtb_struct_node.png)

笔者可以根据上述图然后结合dts和dtb文件进行对应节点分析;  

## 2.4 Strings Block

该块主要是用于存储字符串, 通过节点的定义知道节点都有若干属性, 而不同的节点的属性又有大量相同的属性名称, 因此将这些属性名称提取出一张表, 当节点需要应用某个属性名称时, 直接在属性名字段保存该属性名称在字符串块中的偏移量;  

`Strings Block`的偏移值为0x23c, 查看dtb文件可以发现第一个字符串为`#address-cells`, 在struct中的调用时`nameoff`值为0x00, 第二个字符串为`#size-cells`, 在struct中调用时`nameoff`的值为0x0F;  


# 三. 附件

## 3.1 dtb的十六进制图:  

![](/images/dts/dtb_hexdump.png)


## 3.2 参考

```
http://www.wowotech.net/device_model/dt-code-file-struct-parse.html
https://www.cnblogs.com/aaronLinux/p/5496559.html#t12
```