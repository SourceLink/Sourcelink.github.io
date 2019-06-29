---
layout: post
title:  "使用mdev自动挂载u盘和sd卡"
date:   2019-06-29 10 11:31:50
catalog:  true
author: Sourcelink
tags:
    - linux挂载U盘
    - mdev
    - linux
    - SD卡

---


# 一. 概述

在嵌入式linux设备中发现不能和电脑一样插上U盘就可以直接挂载盘符, 经过筛选和验证发现了mdev实现方式是个比较好的方法, 特此在此记录说明;

>宿主机 ： Ubuntu 16.04 LTS / X64  
目标板：讯为4412 全功能  
uboot： 2010.03  
kernel:  3.0.15  
文件系统: 讯为官方最小系统

**前提需要**: 内核支持usb Storage; 

```
Device Drivers --> 
     USB support -->
           <*> Support for Host-side USB
           <*> OHIC HCD support
           <*> UHCI HCD (most Intel and VIA) support
           <*> SL811HS HCD support
           <*> USB Mass Storage support
```

mdev官方的叫法是mini udev in busybox，mdev实际只是busybox的一个符号链接。执行mdev -s时会扫描/sys/class和/sys/block中的所有目录，如果目录中有名为dev的文件，就从中读取设备号，并利用设备号在/dev下面创建节点。

启动时需要设置热插拔处理程序为mdev，当有热插拔事件产生时，内核调用mdev。 mdev通过环境变量ACTION和DEVPATH确定热插拔事件和影响目录，接着查看目录下是否有dev文件，并利用其信息创建/dev节点。如果ACTION为add，就会创建设备节点，如果为remove则删除设备节点。

# 二. 配置文件系统支持mdev

mdev是busybox自带的一个简化版的udev, 如果使用`busybox`进行文件系统构建记得在配置的时候支持mdev:  

```
#make menuconfig
选择Linux System Utilities --->
         [*]mdev
         [*]     Support /etc/mdev.conf
         [*]          Support subdirs/symlinks
         [*]               Support regular expressions substitutions when renaming device
         [*]          Support command execution at device addition/removal
         [*]     Support loading of firmwares 
```

如果是使用buildroot则不需要进行配置, 它默认支持mdev;



# 三. 修改启动脚本

脚本路径如下:  

```
/etc/init.d/rcS
```

添加如下内容:  

```
/bin/mount -t tmpfs mdev /dev 
/bin/mount -t sysfs sysfs /sys
mkdir /dev/pts
mount -t devpts devpts /dev/pts
/bin/echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s 
```

# 四. 修改mdev.conf

修改mdev.conf文件, 实现u盘/sd卡自动挂载与卸载, `mdev.conf`文件在`/etc`目录下;

在文件最后添加如下内容:  

```
mmcblk[0-9]p[0-9] 0:0 666 @ /etc/sd_card_inserting
mmcblk[0-9] 0:0 666 $ /etc/sd_card_removing
sd[a-z][0-9] 0:0 666 @ /etc/hotplug/udisk_inserting
sd[a-z] 0:0 666 $ /etc/hotplug/udisk_removing
```

# 五. 添加对应脚本

上一节修改`mdev.conf`文件时, 对应有四个脚本需要编写, 分别是sd卡插入和拔出, U盘拔插脚本；

- sd卡拔插如下:

> /etc/sd_card_inserting

```
#!/bin/sh

mkdir -p /mnt/sd
mount /dev/mmcblk1p[0-9] /mnt/sd
```

**PS:** 因为板载emmc, 所以sd卡插入后，对应的是卡1, 即`mmcblk1`; 

> /etc/sd_card_removing


```
#!/bin/sh

umount -l /mnt/sd
rm /mnt/sd -rf
```

- u盘拔插如下:


> /etc/hotplug/udisk_inserting

```
#!/bin/sh

mkdir -p /mnt/sd
mount /dev/sd[a-z][0-9] /mnt/sd
```

> /etc/hotplug/udisk_removing


```
#!/bin/sh

umount -l /mnt/sd
rm /mnt/sd -rf
```

# 六. 实验

一般不出意外，按照上面的步骤是可以实现自动挂载的，重启下就可以进行实验了；

在进行实验的时候发现了拔插了并没有挂载，后来发现原因是因第五节添加的脚本未给予执行权限；

```
chmod +x /etc/sd_card_inserting
chmod +x /etc/sd_card_removing
chmod +x /etc/hotplug/udisk_inserting
chmod +x /etc/hotplug/udisk_removing
```

如果你想调试脚本的话, 不想重启, 你可以直接执行`mdev -s`命令；　

















