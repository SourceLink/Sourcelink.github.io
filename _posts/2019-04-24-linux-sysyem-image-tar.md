---
layout: post
title:  "linux环境system.img解打包"
date:   2019-04-24 22:15:56
catalog:  true
author: Sourcelink
tags:
    - 解包
    - 文件系统

---

# 一. 概述

- 运行环境: ubuntu16.04
- 开发平台: 讯为4412
- 镜像: 开发包中QT的system.img镜像

对于system.img而言, 一般可以使用`mount`命令进行挂载, 在修改过以后再`umout`即可, 这样一次的文件系统镜像的更新则完成了;  

但是对于讯为提供的`system.img`镜像包则有所不同, 读者继续往下看;  

# 二. 解打包

## 2.1 解包 

先使用file命令查看这个system.img镜像属性, 结果如下:  

```
file system.img
system.img: Android sparse image, version: 1.0, Total of 76800 4096-byte output blocks in 1124 input chunks
```

可以看出讯为提供的镜像并非一个标准的linux下的`ext4`格式的镜像而是`Android sparse image`, 所以这里需要先将镜像转换;  

这里使用到了一个命令`simg2img`, 这个执行文件可以在编译的android源码中搜索得到, 将该执行拷贝到`/usr/loacl/bin/`下, 再执行`sudo ldconfig`命令;  

使用方式如下:  

```
simg2img system.img system.img.ext4
```

现在再使用file命令查看转换后的文件镜像:  

```
file system.img.ext4 
system.img.ext4: Linux rev 1.0 ext4 filesystem data, UUID=20fc1eee-2485-a159-a346-5deada1421b2, volume name "linux" (extents) (large files)
```

可以看到文件格式已经转换成标准的`linux ext4`格式;  

接着将转换后的文件镜像挂载:  

```
mkdir mnt_dir
sudo mount system.img.ext4 mnt_dir
```

先创建了挂载目录后, 再执行挂载;  


## 2.2 打包

在上面解包并挂载的基础上对文件系统进行裁剪或者增添后,需要重新打包;  

打包使用了`make_ext4fs`, 该命令必须使用讯为提供的, 因为不同的`make_ext4fs`的打包分区情况不同, 所以打包出的文件系统很可能无法正常使用, 无法正常使用的log一般如下:  

```
[    6.900456] EXT4-fs (mmcblk0p2): mounted filesystem with ordered data mode. Opts: (null)
[    6.907273] VFS: Mounted root (ext4 filesystem) on device 179:2.
[    6.913391] Freeing init memory: 228K
[    6.920537] Failed to execute /linuxrc.  Attempting defaults...
[    6.934588] ***********enter  Panic*********
[    6.937478] panic_dump_info_set: 0x 0
[    6.941106] first 0xe4120001
[    6.943925] info is : 0xe4120001 
[    6.947268] Kernel panic - not syncing: No init found.  Try passing init= option to kernel. See Linux Documentation/init.txt for guidance.
[    6.959687] Backtrace: 
```


将该命令与`simg2img`一样拷贝到`/usr/loacl/bin/`下;  

然后执行命令:  

```
sudo make_ext4fs -s -l 314572800 -a root -L linux system.img mnt_dir
```

这样一个新文件系统镜像就ok了, 可以用 **fastboot** 刷到板子上进行测试;  



