---
layout: post
title:  "手动编译dts"
date:   2019-08-22
catalog:  true
author: Sourcelink
tags:
    - 手动编译dts
    - dts

---

# 一. 概述

> 宿主机: ubuntu16.04
> 开发板: tq-imx6ull
> 内核版本: linux-4.1.15

想手动编译dts时, 遇到如下问题:  

```
Error: tq-imx6ul_net0_uart.dts:2.1-2 syntax error
FATAL ERROR: Unable to parse input tree
```

我使用的命令如下:

```
./dtc -O dtb -I dts -o tq-imx6ul_net0_uart.dtb.bak tq-imx6ul_net0_uart.dts
```


**dtc**命令在内核源码的`scripts/dtc/`目录下, 我将其拷贝了出来放到了dts目录下;  


# 二. 预编译

根据提示错误查看dts文件发现是因为`#include "tq-imx6ul.dts"`该处问题, 但是使用make dtb进行编译的是可以的, 查找资料发现需要修改为`/include/ "tq-imx6ul.dts"`, 按照该方式修改还是为解决问题, 因为包含的一些`.h`文件中有c的语法在内, 比如一些#define, 后查找发现需要预编译, 解决办法如下:  

```
cpp -I ../../../include -E -P -x assembler-with-cpp tq-imx6ul_net0_uart.dts tq-imx6ul_net0_uart.dts.pre
```

> -I ../../../include(我将dts目录下的文件都拷贝出来了所以没有指定路径有所改变)内核设备树源主要是`#include <dt-bindings/input/input.h>` 这是一个相对于`arch/arm/boot/dts/include`的路径.  
> -E 表示仅预处理,不确定在使用cpp命令时是否需要   
> -P 禁用源行号注释,这会混淆设备树编译器   
> -x assembler-with-cpp强制预处理器以某种语言模式运行, 我想这有助于它不会被与预处理器指令在同一文件中的设备树语法混淆.我使用它是因为它在内核Makefile中.  
> tq-imx6ul_net0_uart.dts是需要进行预编译的dts文件  
> tq-imx6ul_net0_uart.dts.pre为预编译完成的dts文件  


> tq-imx6ul_net0_uart.dts.pre

部分内容如下:  

```
 aliases { };
 memory { device_type = "memory"; reg = <0 0>; };
};
/ {
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
  i2c0 = &i2c1;
  i2c1 = &i2c2;
  i2c2 = &i2c3;
  i2c3 = &i2c4;
  mmc0 = &usdhc1;
  mmc1 = &usdhc2;
  serial0 = &uart1;
  serial1 = &uart2;
  serial2 = &uart3;
  serial3 = &uart4;
  ....
```

可以看到预编译已经把多个文件合成, 而且都如何dts语法;  

# 三. 编译  

接下来将使用**dtc**命令进行手动编译:  

```
./dtc -O dtb -I dts -o tq-imx6ul_net0_uart.dtb.bak tq-imx6ul_net0_uart.dts.pre
```

> -I dts指定输入格式是文本设备树源  
> -O dtb指定创建设备树二进制blob  
> -o tq-imx6ul_net0_uart.dtb.bak定义输出的文件名  
> tq-imx6ul_net0_uart.dts.pre 将要编译成dtb的文件  


# 四. 反汇编

```
./dtc -O dts -I dtb -o tq-imx6ul_net0_uart.dts.bak tq-imx6ul_net0_uart.dtb.bak
```

> -I dtb 指定输入设备树二进制文件  
> -O dts指定创建设备树文本格式  
> -o tq-imx6ul_net0_uart.dts.bak定义输出的文件名  
>  tq-imx6ul_net0_uart.dtb.bak将要反汇编的dtb文件  


截取反汇编的部分内容如下:

> tq-imx6ul_net0_uart.dts.bak  

```
/dts-v1/;

/ {
	#address-cells = <0x1>;
	#size-cells = <0x1>;
	model = "Freescale i.MX6 UltraLite 14x14 EVK Board";
	compatible = "fsl,imx6ul-14x14-evk", "fsl,imx6ul";

	chosen {
		stdout-path = "/soc/aips-bus@02000000/spba-bus@02000000/serial@02020000";
	};

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
		i2c0 = "/soc/aips-bus@02100000/i2c@021a0000";
		i2c1 = "/soc/aips-bus@02100000/i2c@021a4000";
		i2c2 = "/soc/aips-bus@02100000/i2c@021a8000";
		i2c3 = "/soc/aips-bus@02100000/i2c@021f8000";
		mmc0 = "/soc/aips-bus@02100000/usdhc@02190000";
		mmc1 = "/soc/aips-bus@02100000/usdhc@02194000";
		serial0 = "/soc/aips-bus@02000000/spba-bus@02000000/serial@02020000";
		serial1 = "/soc/aips-bus@02100000/serial@021e8000";
		serial2 = "/soc/aips-bus@02100000/serial@021ec000";
		serial3 = "/soc/aips-bus@02100000/serial@021f0000";
		serial4 = "/soc/aips-bus@02100000/serial@021f4000";
		serial5 = "/soc/aips-bus@02100000/serial@021fc000";
		serial6 = "/soc/aips-bus@02000000/spba-bus@02000000/serial@02018000";
		serial7 = "/soc/aips-bus@02000000/spba-bus@02000000/serial@02024000";
		spi0 = "/soc/aips-bus@02000000/spba-bus@02000000/ecspi@02008000";
		spi1 = "/soc/aips-bus@02000000/spba-bus@02000000/ecspi@0200c000";
		spi2 = "/soc/aips-bus@02000000/spba-bus@02000000/ecspi@02010000";
		spi3 = "/soc/aips-bus@02000000/spba-bus@02000000/ecspi@02014000";
		usbphy0 = "/soc/aips-bus@02000000/usbphy@020c9000";
		usbphy1 = "/soc/aips-bus@02000000/usbphy@020ca000";
	};
```

# 五. 预编译然后并解析成dtb

预编译然后并解析成dtb

```
cpp -I ../../../arch/arm/boot/dts/include -E -P -x assembler-with-cpp tq-imx6ul_net0_uart.dts | ../dtc -O dtb -I dts -o tq-imx6ul_net0_uart.dtb.bak -
```
