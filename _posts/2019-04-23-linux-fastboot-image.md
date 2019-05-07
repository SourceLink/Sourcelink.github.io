---
layout: post
title:  "linux环境fastboot烧写镜像"
date:   2019-04-23 22:36:56
catalog:  true
author: Sourcelink
tags:
    - 烧写
    - 4412

---

# 一. 概述

**fastboot** 烧写方式, 该方式可以烧写 Android 系统和 Qt 系统;  
Windows下和linux下都可以使用该烧写方式, 推荐使用linux下的烧写方式, 接下来讲解linux的环境烧写步骤;  

**环境**:

- 运行环境: ubuntu16.04
- 开发平台: 讯为4412
- 烧写: 开发包中QT镜像

# 二. 烧写步骤

## 2.1 fastboot 安装

```
sudo apt-get install fastboot
```

可以使用 `fastboot -h` 查看使用详情;  

主要的格式如下:  

```
fastboot flash <partition> [ <filename> ] 
```

## 2.2 格式化分区

假如原先已经分过区, 或者已经烧写过镜像则不需要进行一下步骤;

给开发板上电进入uboot模式, 输入分区命令: 

```
fdisk -c 0
```

- 格式化分区1

```
fatformat mmc 0:1
```

- 格式化分区2

```
ext3format mmc 0:2
```

- 格式化分区3

```
ext3format mmc 0:3
```

- 格式化分区4

```
ext3format mmc 0:4
```


## 2.3 准备分区

- 进入fastboot模式

在开发板的uboot模式下, 输入:

```
fastboot
```

以上命令可以使开发板那边进入fastboot模式, 这样电脑端才能发现该设备;  

- 验证设备连接

将开发板的otg连接到电脑端, 在电脑端输入:  

```
sudo fastboot devices
```

如果连接成功,执行以上命令则会显示连接的设备名;  


- 烧写uboot

```
sudo fastboot flash bootloader u-boot-iTOP-4412.bin
```

- 烧写kernel

```
sudo fastboot flash kernel zImage
```

- 烧写ramdisk

```
sudo fastboot flash ramdisk ramdisk-uboot.img
```

- 烧写文件系统

```
sudo fastboot flash system system.img
```



# 三. 重启开发板

- 重启开发板
```
sudo fastboot reboot
```

- 擦除缓存数据

```
sudo fastboot -w
```


**备注**: 以上加`sudo`, 即`root`权限的原因是需要读取usb设备的权限;  















