---
layout: post
title:  "【从零开始写RTOS】VSCODE开发STM32"
date:   2019-12-8
catalog:  true
author: Sourcelink
tags:
    - vscode
    - stm32

---

#  一. 构建

- 先生成makefile

```
cmake .. -DBOARD_FAMILY=NUCLEO_STM32F746ZG
```

其中`-DBOARD_FAMILY=NUCLEO_STM32F746ZG`是为了指定对应的board；

- 编译makefile

```
make
```


输出日志：

```
Scanning dependencies of target arch
[ 10%] Building C object arch/arm/arm-v7m/CMakeFiles/arch.dir/corterxm-common/gcc/port_c.c.obj
[ 20%] Building ASM object arch/arm/arm-v7m/CMakeFiles/arch.dir/corterxm-common/gcc/port_s.s.obj
[ 30%] Linking C static library ../../../lib/libarch.a
[ 30%] Built target arch
Scanning dependencies of target board
[ 40%] Building C object board/NUCLEO_STM32F746ZG/CMakeFiles/board.dir/board.c.obj
[ 50%] Linking C static library ../../lib/libboard.a
[ 50%] Built target board
Scanning dependencies of target slos_kernel
[ 60%] Building C object kernel/CMakeFiles/slos_kernel.dir/cpu/kos_cpu.c.obj
[ 70%] Linking C static library ../lib/libslos_kernel.a
[ 70%] Built target slos_kernel
Scanning dependencies of target demo_proc_manager.elf
[ 80%] Building ASM object demo/proc_manager/CMakeFiles/demo_proc_manager.elf.dir/__/__/platform/st/CMSIS/Device/ST/STM32F7xx/Source/Templates/gcc/startup_stm32f746xx.s.obj
[ 90%] Building C object demo/proc_manager/CMakeFiles/demo_proc_manager.elf.dir/proc_manager.c.obj
[100%] Linking C executable ../../bin/demo_proc_manager.elf
[100%] Built target demo_proc_manager.elf
Scanning dependencies of target size
   text    data     bss     dec     hex filename
   6332     108    2412    8852    2294 /home/sourcelink/work/code/rtos/Sl-Rtos/build/bin/demo_proc_manager.elf
[100%] Built target size
Scanning dependencies of target demo_proc_manager.bin
[100%] Built target demo_proc_manager.bin
Scanning dependencies of target demo_proc_manager.hex
[100%] Built target demo_proc_manager.hex
```


从日志中可以看出，分别编译三个静态库，最后再构建一个执行文件进行动态库连接；


目前整个项目的目录结构如下：

```
.
├── arch
│   └── arm
│       └── arm-v7m
│           ├── CMakeLists.txt
│           ├── common
│           │   └── include
│           └── corterxm-common
│               ├── armcc
│               │   ├── sl_port_c.c
│               │   ├── sl_port.h
│               │   └── sl_port_s.s
│               └── gcc
│                   ├── port_c.c
│                   ├── port.h
│                   └── port_s.s
├── board
│   └── NUCLEO_STM32F746ZG
│       ├── board.c
│       ├── CMakeLists.txt
│       ├── include
│       │   └── board.h
│       └── ldscripts
│           └── STM32F746ZGTx_FLASH.ld
├── build
├── cmake
│   └── gcc_stm32f7.cmake
├── CMakeLists.txt
├── demo
│   └── proc_manager
│       ├── CMakeLists.txt
│       └── proc_manager.c
├── doc
├── platform
├── kernel
│   ├── CMakeLists.txt
│   ├── cpu
│   │   ├── include
│   │   │   └── kos_cpu.h
│   │   └── kos_cpu.c
│   ├── fallaway
│   │   ├── sl_ready_queue.c
│   │   └── sl_ready_queue.h
│   └── include
├── README.md
└── test

```

arch目录下有个CMakeLists.txt，board目录下的板子NUCLEO_STM32F746ZG下也有个CMakeLists.txt， kernel目录下也有个CMakeLists.txt， 这三个CMakeLists.txt文件的目的是将其分别编译成对应的静态库；

这样设计的原因如下：

> 1. 内核的设计不应该与使用的mcu有关系，所以需要提供一个arch（对于内核来说差异只在上下文调度和临界区处理）  
> 2. 分离arch和board是因为有的板子虽然使用的是同一个架构但是一些接口设计有所不同
> 3. 可以更好的复用代码，编写demo测试程序


# 二. download

- 方式1

```
make flash
```


- 方式2

```
st-flash write xxx.bin 0x80000000
```

