---
layout: post
title:  "【从零开始写RTOS】汇编语言"
date:   2019-11-27
catalog:  true
author: Sourcelink
tags:
    - 从零开始写RTOS
    - 汇编

---

# 一. 环境

> 开发环境： linux
> 汇编：gcc


# 二. 指令

> .equ

类似#define , 常量符号

exp：

```
.equ NVA 0x10000
```


> .text

声明接下来的代码都是放在text段（可以去看下链接文件）


> .align 4

变量的对齐宽度


> .thumb

表示接下来都使用汇编为**thumb指令**


> .syntax unified

统一汇编语法

```
Cortex-m3为了兼容thumb指令和thumb2指令，使这两种指令可以使用统一的格式，引入了一种叫做"统一汇编语言UAL"的语法机制。简单说来就是我们不用关心我们使用的是thumb指令还是thumb2指令，而是统一使用32位thumb2指令的语法格式书写
```

> .type xxx, %function

声明xxx为一个函数

exp：

```
.type port_start, %function
port_start:
    ldr r0, r1
```

> cpsid i

屏蔽可配置的优先级中断， 等同primask = 1

> cpsie i

开启可配置的优先级中断， 等同primask = 0


> push

压栈

exp:

```
push {r4, r5}
```

> pop

出栈

exp：

```
pop {r4, r5}
```


> ldr

字数据加载指令

exp：

```
ldr r0, =50
```

> ldrb

字节数据加载指令

exp：

```
ldr r0, [r1]
```

将存放在r1地址中值存储到r0中


> str

字数据存储指令

exp：

```
str r5, [r4]
```

将r5寄存器中的值存放到r4寄存器地址中， *r4 = r5

缩写：store from a register into memory

> strb

字节数据存储指令

> add 

加法指令

```
add r1, r1, #1
```

等同于： r1 = r1 + 1


> adds 

等同于add指令， 区别会影响到标志寄存器xpsr


> sub

减法指令

```
sub r1, r1, #1
```

等同于： r1 = r1 - 1

> subs 

等同于sub指令， 区别会影响到标志寄存器xpsr

> mov

一般传送指令

exp：

```
mov r1, #0
```

等同于：r1 = 0

> msr

用于将操作数的内容传送到程序状态寄存器的特定域中， 操作数可以为通用寄存器或立即数

exp：

```
msr primask, #1
```

```
msr primask, r1
```

传送r1值的内容到primask


> mrs

用于将程序状态寄存器的内容传送到通用寄存器中

```
mrs r0, psp  #Process_Stack_Pointer
```

将psp指针中的内容存放到r0

> bx

bx 目标地址

指令跳转到指定的目标地址中， 目标地址处指令即可是ARM指令也可以是Thumb指令


```
bx lr
```

返回子程序


> cbz 

比较指令，如果为零就转移


参考链接：

```
https://blog.csdn.net/u010893262/article/details/52808639
```

> orr

用于两个操作数上的逻辑或运算

exp:

```
orr lr, lr, #0x04
```

lr中的值与立即数0x04相或，结果存储到lr中


> stm

类似与str但是它的对象为一组寄存器

exp：

```
stm r0, {r4-r11}
```

将r4-r11寄存器值保存到r0所指向的内存

缩写：store from registers into memory


> ldm

类似与ldr用于加载内存中的数据，但是它的对象为一块内存

exp：

```
ldm r0, {r4-r11}
```

将r0所指向的内存数据存储到r4-r11

缩写：load from  memory into  registers










