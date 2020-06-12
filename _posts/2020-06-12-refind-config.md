---
layout: post
title:  "rEFInd引导linux+mac双系统"
date:   2020-06-12
catalog:  true
author: Sourcelink
tags:
    - rEFInd
    - 引导
    - macos
---

# 一. 概述


因为前段时间系统滚挂了以后， 重装了manjaro, 并且装了一个黑苹果，因为是先安装的manjaro, 后安装的黑苹果。  
一般情况下manjaro会扫描后面安装的系统引导，并显示在启动菜单中；但是这次并没有， 而且我是一个硬盘一个系统。y
试了很多种办法都没有在一个引导界面， 引导两个系统， 最后还是通过rEFInd完成了双系统引导；

# 二. 安装rEFInd

- 1.安装软件

```
yay  -S refind
```

或者

```
sudo pacman  -S refind
```

- 2. 讲refind安装到boot

执行如下命令讲refind安装到boot目录：

```
sudo refind-install
```

# 三. 安装主题

具体步骤如下：

```
cd /boot/EFI/refind/
mkdir -p themes
git clone https://github.com/EvanPurkhiser/rEFInd-minimal themes/rEFInd-minimal
echo "include themes/refind-minimal/theme.conf" >> refind.conf
```


# 四. 修改配置


refind的配置文件目录如下：

> /boot/efi/EFI/refind/refind.conf

先讲黑苹果引导盘中`CLOVER`整个拷贝到`/boot/efi/EFI`目录下， 完成后目录大概如下情况：

![](/images/refind/20200611225729189_1936265512.png)

接下修改配置文件：`/boot/efi/EFI/refind/refind.conf`

## 4.1 设置等待时间

```
#设置默认等待时间为20s
#timeout 0代表无限等待 timeout -1代表立即进入对应的系统
timeout 20
```


## 4.2 引导背景

讲背景设置成前面安装的主题：

```
banner themes/rEFInd-minimal/backgroud.png 
```

## 4.3 全屏显示

```
banner_scale fillscreen
```

## 4.4 分辨率设置

```
resolution 1920 1080 
```

## 4.5 屏蔽扫描目录


因为refind会自己自动扫描目录下的引导， 并自动加载进菜单， 但是这样界面会比较难看而且会员一些我们不用的引导出现， 所以后面我们自己配置；

```
dont_scan_dirs ESP:/EFI/boot,EFI/CLOVER,EFI/Manjaro
```

## 4.6 屏蔽扫描文件

如上，我们接着讲引导文件屏蔽， 后面自己配置引导；

```
dont_scan_files BOOTX64.efi,CLOVERX64.efi
```

这里分别屏蔽了原来manjaro和黑苹果的引导；


## 4.7 手动配置manjaro


```
menuentry "Manjaro" {
    loader /EFI/Manjaro/grubx64.efi
    icon /EFI/refind/themes/rEFInd-minimal/icons/os_manjaro.png
    enable 
}
```

## 4.8 手动配置macos

```
menuentry "Macos" {
    loader /EFI/CLOVER/CLOVERX64.efi
    icon /EFI/refind/themes/rEFInd-minimal/icons/os_mac.png
    enable 
}
```


# 五. 总结

这样我的电脑引导分部图如下：

![](/images/refind/20200611231605495_1454770022.png)




