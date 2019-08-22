---
layout: post
title:  "设备树DTS格式解析"
date:   2019-08-22
catalog:  true
author: Sourcelink
tags:
    - dts
    - 设备树语法

---




# 一. 概述


> 宿主机: ubuntu16.04
> 开发板: tq-imx6ull
> 内核版本: linux-4.1.15

用实例讲解下设备树dts语法, `dts`文件都在内核的`arch/arm/boot/dts`目录下; 在查看`dts`文件时候发现这有些c语言的语法包含在其中, 比如`#inlcude <xxx.h>`, `#include "xxx.dtsi"`; 接下来分析天嵌官方提供的设备树文件; 



# 二. 语法分析


> arch/arm/boot/dts

tq官方提供了两种设备树配置, 一种是单网口和8串口配置, 一种是双网口和6串口配置;  笔者将以后一种配置的设备树文件进行分析, 开头有说过, dts文件支持类似C语言的语法, 下面列出了包含情况:  

```
tq-imx6ul_net0_uart.dts
    #include "tq-imx6ul.dts"
        #include <dt-bindings/input/input.h>
        #include "imx6ul.dtsi"
            #include "skeleton.dtsi"
            #include <dt-bindings/clock/imx6ul-clock.h>
            #include <dt-bindings/gpio/gpio.h>
            #include <dt-bindings/interrupt-controller/arm-gic.h>
            #include "imx6ul-pinfunc.h"
```

先从最前面的dts文件分析:  

> tq-imx6ul_net0_uart.dts

截取部分内容如下:  

```
#include "tq-imx6ul.dts"

.....

&fec1 {                                                    ①
	pinctrl-names = "default";                             
	pinctrl-0 = <&pinctrl_enet1>;                          ②
	phy-mode = "rmii";
	phy-handle = <&ethphy0>;
	status = "okay";

	mdio {                                                 
		#address-cells = <1>;
		#size-cells = <0>;

		ethphy0: ethernet-phy@2 {
			compatible = "ethernet-phy-ieee802.3-c22";
			reg = <2>;
		};
	};

};

.....

```

## 2.1 dts修改方式

> ①: `&fec1`表示引用标签`fec1`, 它来自`imx6ul.dtsi`, 原型如下: 


```
fec1: ethernet@02188000 {
	compatible = "fsl,imx6ul-fec", "fsl,imx6q-fec";
	reg = <0x02188000 0x4000>;
	interrupts = <GIC_SPI 118 IRQ_TYPE_LEVEL_HIGH>,
		     <GIC_SPI 119 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&clks IMX6UL_CLK_ENET>,
		 <&clks IMX6UL_CLK_ENET_AHB>,
		 <&clks IMX6UL_CLK_ENET_PTP>,
		 <&clks IMX6UL_CLK_ENET_REF>,
		 <&clks IMX6UL_CLK_ENET_REF>;
	clock-names = "ipg", "ahb", "ptp",
		      "enet_clk_ref", "enet_out";
	stop-mode = <&gpr 0x10 3>;
	fsl,num-tx-queues=<1>;
	fsl,num-rx-queues=<1>;
	fsl,magic-packet;
	fsl,wakeup_irq = <0>;
	status = "disabled";
 };
```

仔细观察它比在`tq-imx6ul_net0_uart.dts`时的属性多的多, 其实这是dts的一个特点, 可以进行**引用修改**, 即在原有的基础上进行修改或增加, 以属性`status`为例, 原本是`"disable"`状态, 后引用修改为`"okay"`状态, 而不需要重新全部的写一边节点; 因为在编译成dtb的时候, 编译器会将他们合并; 这个可以通过dtb文件反汇编成dts文件查看[反汇编dtb](https://sourcelink.top/2019/08/22/dts-compare/), 上述`ethernet@02188000`节点反汇编后相关的数据如下:  


```
ethernet@02188000 {
	compatible = "fsl,imx6ul-fec", "fsl,imx6q-fec";
	reg = <0x2188000 0x4000>;
	interrupts = <0x0 0x76 0x4 0x0 0x77 0x4>;
	clocks = <0x1 0x90 0x1 0x91 0x1 0x30 0x1 0x2c 0x1 0x2c>;
	clock-names = "ipg", "ahb", "ptp", "enet_clk_ref", "enet_out";
	stop-mode = <0x16 0x10 0x3>;
	fsl,num-tx-queues = <0x1>;
	fsl,num-rx-queues = <0x1>;
	fsl,magic-packet;
	fsl,wakeup_irq = <0x0>;
	status = "okay";
	pinctrl-names = "default";
	pinctrl-0 = <0x1a>;
	phy-mode = "rmii";
	phy-handle = <0x1b>;

	mdio {
		#address-cells = <0x1>;
		#size-cells = <0x0>;

		ethernet-phy@2 {
			compatible = "ethernet-phy-ieee802.3-c22";
			reg = <0x2>;
			linux,phandle = <0x1b>;
			phandle = <0x1b>;
		};
	};
};
```

可以看到他们已经合并在一起了, 其实还有修改一种方式如下:  

```
/ {
    ethernet@02188000 {
	    ......
	    status = "okay";
    };
};
```

该方式称为**重写**; 注意, 这样书写的方式必须写在root节点内; 讲到这里觉得有比较说下dts的基本语法知识, 如下:  

```
[label:] node-name[@unit-address] {
    [properties definitions]
    [child nodes]
};
```

在设备树中最开始的符号:  

```
/ {

};
```

表示根节点, 也叫做root节点, 而`fec1`表示一个label, `ethernet`表示节点名字, 则`@02188000`表示节点对应的寄存器地址, 加上这个只是为了区分不同的节点, 比如这款soc就支持双网口;  
`compatible`和`status`表示一个属性, `mdio { };`则表示一个子节点;  

从lable的引用可以看出很多公共的节点都会先在`.dtsi`中书写或`.h`中书写, 有差异的时候需要修改则在`.dts`中引用修改;


## 2.2 lable转换phandle

> ②: 属性`pinctrl-0`通过 `pinctrl_enet1`标签引用;

```
pinctrl_enet1: enet1grp {
			fsl,pins = <
				MX6UL_PAD_GPIO1_IO07__ENET1_MDC		0x1b0b0
				MX6UL_PAD_GPIO1_IO06__ENET1_MDIO	0x1b0b0
                 .....
			>;
		};
```

其实在编译以后, lable会转换成属性`phandle`在该节点中, 并且会为其赋上u32类型的值; 反汇编后`enet1grp`节点的dts如下:


```
enet1grp {
    ....
	linux,phandle = <0x1a>;
	phandle = <0x1a>;
};
```

我们在看看前面反汇编出的`ethernet@02188000`节点, 属性`pinctrl-0`引用的值变为了**0x1a**

```
ethernet@02188000 {
	compatible = "fsl,imx6ul-fec", "fsl,imx6q-fec";
    ....
	pinctrl-0 = <0x1a>;
	phy-mode = "rmii";
	phy-handle = <0x1b>;
    ....
};
```



## 2.3 属性取值方式


- prop-encoded-array

比如: `local-mac-address = [ 00 00 12 34 56 78];`

每个字节的值用两个16进制的数表示, 也可以写成`local-mac-address = [ 000012345678];`

- string

比如: `compatible = "fsl,imx6ul-lcdif";`

- string list

比如: `compatible = "fsl,imx6ul-lcdif", "fsl,imx28-lcdif";`

- u32

比如: `address-bits = <48>;`

- phandle

比如: `phy-handle = <&PHY0>;`

该方式其实是引用lable, 最后编译其实是个u32类型的整数;  

