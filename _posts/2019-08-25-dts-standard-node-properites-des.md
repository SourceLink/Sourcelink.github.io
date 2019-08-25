---
layout: post
title:  "设备树标准节点和属性"
date:   2019-08-24
catalog:  true
author: Sourcelink
tags:
    - dts

---

# 一. 概述

> 宿主机: ubuntu16.04
> 开发板: tq-imx6ull
> 内核版本: linux-4.1.15

这篇笔记会记录一些关于设备树常用的标准属性和节点的信息; 


# 二. 标准属性

## 2.1 compatible

该属性用于匹配板卡或驱动， 如下`compatible`是根节点下的属性，用于匹配板卡：

```
compatible = "fsl,imx6ul-14x14-evk", "fsl,imx6ul";
```

该属性值的设定格式一般为 `<制造商，设备名称>`， 上述可以看到它是个stringlist形式，即可以有多值，假如第一个值没有匹配上，则会继续匹配；  
匹配驱动的compatible属性值如下：

```
compatible = "fsl,imx6ul-dma-apbh", "fsl,imx28-dma-apbh";
```

内核将首先尝试查找支持的设备驱动程序`fsl,imx6ul-dma-apbh`， 如果没有则查找`fsl,imx28-dma-apbh`，如果都没有则不会注册驱动； 


## 2.2 phandle

该属性一般用户在编写dts文件是见不到也不会去使用的，因为一般都会使用label引用拉替代，而对应phandle交给编译器dtc去转换，示例详细看[设备树dts格式解析2.2节](https://sourcelink.top/2019/08/22/dts-grammer/)；


## 2.3 status

status属性表示设备的运行状态，表中列出并定义了有效值：

|     值      |               描述               |
| ---------- | ------------------------------- |
| "okay"     | 表示设备可以运行                   |
| "disabled" | 表示设备现不可运行                  |
| "reserved" | 表示设备应该保留,不使用              |
| "fail"     | 表示设备无法运行                   |
| "fail-sss" | 表示设备无法运行,sss表示遇到问题的原因 |


属性status的一种使用方式如下:  

> imx6ul.dtsi

```
fec1: ethernet@02188000 {
	compatible = "fsl,imx6ul-fec", "fsl,imx6q-fec";
    ....
	fsl,num-rx-queues=<1>;
	fsl,magic-packet;
	fsl,wakeup_irq = <0>;
	status = "disabled";
 };
```

> tq_imx6ul_net0_uart.dts

```
&fec1 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_enet1>;
	phy-mode = "rmii";
	phy-handle = <&ethphy0>;
	status = "okay";
    ....
};
```

## 2.4 #address-cells and #size-cells

-  #address-cells

#address-cells属性用于定义子节点的reg属性中的地址字段的<u32>类型单元的数量。

- #size-cells

#size-cells 属性定义用于对子节点的reg属性中的size字段的<u32>类型单元的数量。

**example:**

```
/ {
	#address-cells = <0x1>;
	#size-cells = <0x1>;
	model = "Freescale i.MX6 UltraLite 14x14 EVK Board";
	compatible = "fsl,imx6ul-14x14-evk", "fsl,imx6ul";

    interrupt-controller@00a01000 {
    		compatible = "arm,cortex-a7-gic";
    		#interrupt-cells = <0x3>;
    		interrupt-controller;
    		reg = <0xa01000 0x1000 0xa02000 0x100>;
    		linux,phandle = <0x25>;
    		phandle = <0x25>;
    	};
};
```

其中属性`#address-cells`值为1, 所以属性`reg`中的地址字段为一个u32长度值(0xa01000), 属性`#size-cells`值为1,  所以属性`reg`中的size字段为一个u32长度值(0x1000);且值`0xa02000`为地址值, `0x100`为size值;  
假如现在属性`#address-cells`值为2, 则`reg`中的地址字段需要两个u32类型的值类似: `reg = <0xa01000 0xa01000 0x1000 0xa01000 0xa02000 0x100>;`   

## 2.5 reg

reg属性一般描述设备对应寄存器地址和长度;

**example:**

```
fec1: ethernet@02188000 {
	compatible = "fsl,imx6ul-fec", "fsl,imx6q-fec";
	reg = <0x02188000 0x4000>;
     .....
	status = "disabled";
};
```

其中`0x02188000`表示该网卡设备的寄存器偏移地址, `0x4000`则表示偏移长度;  


## 2.6 ranges

该属性值可以为空或数组;  ranges属性提供了一种定义总线地址空间和总线节点父节点的地址空间之间的映射或转换的方法, range属性值的格式是(子总线地址, 父总线地址, 长度)的任意数量的三元组;  

如果使用<empty>值定义属性, 则它指定父地址空间和子地址空间相同, 并且不需要地址转换; 

**example:**

```
soc {
	#address-cells = <1>;
	#size-cells = <1>;
	compatible = "simple-bus";
	interrupt-parent = <&gpc>;
	ranges;

    ....
	
    	caam_sm: caam-sm@00100000 {
        	compatible = "fsl,imx7d-caam-sm", "fsl,imx6q-caam-sm";
        	reg = <0x00100000 0x3fff>;
    };
};
```

上述ranges属性值为空, 则表示子节点中`caam_sm`的`reg`属性的地址值0x00100000不需要进行映射或转换它就是一个实际的地址值;  


如果ranges属性值不为空:  

```
soc {
	compatible = "simple-bus";
	#address-cells = <1>;
	#size-cells = <1>;
	ranges = <0x0 0xe0000000 0x00100000>;
	serial {
		device_type = "serial";
		compatible = "ns16550";
		reg = <0x4600 0x100>;
		clock-frequency = <0>;
		interrupts = <0xA 0x8>;
		interrupt-parent = <&ipic>;
	};
};
```

在物理地址0x0处寻址的子节点映射到父节点物理地址0xe0000000处,  通过此映射后, 则可以使用地址0xe0004600处的加载或存储来寻址串行设备节点(0xe0000000+0x4600);   



# 三. 标准节点

## 3.1 /

```
/ {
	#address-cells = <1>;
	#size-cells = <1>;
	model = "xxxxx";
    compatible = "xxx,xxx";
};
```

设备树中根节点当然不能缺少, 一般根结点下有上述四个属性需要填写, `#address-cells`和`#size-cells`指定了根节点下的属性reg的地址类型大小和size类型大小, 如果子节点中未指定上述两个属性, 则与其父节点属性相同;  `model`属性主要描述制造商与板级名称; `compatible`属性主要用于内核匹配平台代码使用;  


## 3.1 /aliases

该节点顾名思义就是别名的意思, 看dts中的代码:  

```
aliases {
	can0 = &flexcan1;
	can1 = &flexcan2;
	ethernet0 = &fec1;
	ethernet1 = &fec2;
	gpio0 = &gpio1;
	gpio1 = &gpio2;
	gpio2 = &gpio3;
	gpio3 = &gpio4;
	gpio4 = &gpio5;
    .....
};
```

从dtb反汇编后代码如下:  

```
aliases {
	can0 = "/soc/aips-bus@02000000/can@02090000";
	can1 = "/soc/aips-bus@02000000/can@02094000";
	ethernet0 = "/soc/aips-bus@02000000/ethernet@02188000";
	ethernet1 = "/soc/aips-bus@02000000/ethernet@020b4000";
	gpio0 = "/soc/aips-bus@02000000/gpio@0209c000";
	gpio1 = "/soc/aips-bus@02000000/gpio@020a0000";
	gpio2 = "/soc/aips-bus@02000000/gpio@020a4000";
	gpio3 = "/soc/aips-bus@02000000/gpio@020a8000";
	gpio4 = "/soc/aips-bus@02000000/gpio@020ac000";
    ......
};
```

别名`can0`的值是`/soc/aips-bus@02000000/can@02090000`, 即根目录下的soc节点下的aips-bus@02000000节点下的`can@02090000`节点;  
可以看出别名的值是该节点的完整路径, 如果开发者在驱动开发中访问`can0`节点, 就可以通过在别名部分中搜索它而不是在整个DTS中搜索它;  

## 3.2 /memory


> tq-im6ul.dts

```
/ {
	model = "Freescale i.MX6 UltraLite 14x14 EVK Board";
	compatible = "fsl,imx6ul-14x14-evk", "fsl,imx6ul";

	chosen {
		stdout-path = &uart1;
	};

	memory {
	    	device_type = "memory";
		reg = <0x80000000 0x20000000>;
	};
	....
};
```

memory节点主要指定了内存的其实地址和长度而该节点有个很重要的属性`device_type`, 内核其实是靠它判断该节点是否用于指定内存地址和大小, 而不是memory节点本身;  


## 3.3 /chosen

**example1:**

> tq-im6ul.dts

```
chosen {
	stdout-path = &uart1;
};
```

上面的chosen节点中`stdout-path`属性用于指定输出流的路径, 其还有`stdin-path`属性指定输入流路径;  


**example2:**

```
chosen {
    bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0,115200";
};
```

还可以在chosen节点中, 使用`bootargs`属性进行根目录, 指定console设置;  


## 3.4 /cpus

所有设备都需要cpus节点, 它不代表系统中的真实设备, 而是充当容器用于代表系统CPU的子cpu节点;  

```
	cpus {
		#address-cells = <1>;
		#size-cells = <0>;
	    ....
	};
```

一般cpus节点下会有上述两个属性`#address-cells`和`#size-cells`同样是用来指定reg属性的值类型, 不过现在reg的值只有一个就是地址而没有长度;

一般如下:  

```
	cpus {
		#address-cells = <1>;
		#size-cells = <0>;
	    cpu0: cpu@0 {
			compatible = "arm,cortex-a7";
			device_type = "cpu";
			reg = <0>;
			...
		};
	    cpu1: cpu@1 {
			compatible = "arm,cortex-a7";
			device_type = "cpu";
			reg = <1>;
			...
		};
	};
```

因为有的cpu是多核, reg的值就表示第几个核;  也会有这样的:  

```
cpus {
		#address-cells = <1>;
		#size-cells = <0>;

		cpu0: cpu@A00 {
			device_type = "cpu";
			compatible = "arm,cortex-a9";
			reg = <0xA00>;
			clocks = <&clock CLK_ARM_CLK>;
			clock-names = "cpu";
			operating-points-v2 = <&cpu0_opp_table>;
		};

		cpu@A01 {
			device_type = "cpu";
			compatible = "arm,cortex-a9";
			reg = <0xA01>;
			operating-points-v2 = <&cpu0_opp_table>;
		};
};
```
