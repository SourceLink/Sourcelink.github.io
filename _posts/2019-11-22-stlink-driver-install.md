---
layout: post
title:  "【从零开始写RTOS】stlink环境搭建"
date:   2019-11-22
catalog:  true
author: Sourcelink
tags:
    - stlink

---

# 一. 源码安装

## 1.1 源码下载

```
git clone https://github.com/texane/stlink
```


## 1.2 编译

- 1. make

直接在`stlink`目录下直接`make`

- 2. cmake

```
mkdir build && cd build && cmake .. && make && sudo make
```

- 3. make && cmake

先执行以下命令进行编译: 

```
make release 
```

再进入`build/Release`, 执行`sudo make install`命令进行安装;

按照日志: 

```
[ 20%] Built target stlink
[ 29%] Built target st-flash
[ 50%] Built target stlink-static
[ 55%] Built target st-info
[ 67%] Built target st-util
[ 73%] Built target stlink-gui
[ 79%] Built target stlink-gui-local
[ 88%] Built target flash
[ 94%] Built target usb
[100%] Built target sg
Install the project...
-- Install configuration: "Debug"
-- Installing: /usr/local/lib/libstlink.so.1.5.1
-- Installing: /usr/local/lib/libstlink.so.1
-- Installing: /usr/local/lib/libstlink.so
-- Installing: /usr/local/lib/libstlink.a
-- Installing: /usr/local/bin/st-flash
-- Set runtime path of "/usr/local/bin/st-flash" to ""
-- Installing: /usr/local/bin/st-info
-- Set runtime path of "/usr/local/bin/st-info" to ""
-- Installing: /etc/modprobe.d/stlink_v1.conf
-- Installing: /etc/udev/rules.d/49-stlinkv1.rules
-- Installing: /etc/udev/rules.d/49-stlinkv2-1.rules
-- Installing: /etc/udev/rules.d/49-stlinkv2.rules
-- Installing: /etc/udev/rules.d/49-stlinkv3.rules
-- Installing: /usr/local/bin/st-util
-- Set runtime path of "/usr/local/bin/st-util" to ""
-- Installing: /usr/local/bin/stlink-gui
-- Installing: /usr/local/share/stlink/stlink-gui.ui
-- Installing: /usr/local/share/applications/stlink-gui.desktop
-- Installing: /usr/local/share/icons/hicolor/scalable/apps/stlink-gui.svg
-- Installing: /usr/local/lib/pkgconfig/stlink.pc
-- Installing: /usr/local/include/stlink.h
-- Installing: /usr/local/include/stlink/version.h
-- Installing: /usr/local/include/stlink/backend.h
-- Installing: /usr/local/include/stlink/chipid.h
-- Installing: /usr/local/include/stlink/commands.h
-- Installing: /usr/local/include/stlink/flash_loader.h
-- Installing: /usr/local/include/stlink/logging.h
-- Installing: /usr/local/include/stlink/mmap.h
-- Installing: /usr/local/include/stlink/reg.h
-- Installing: /usr/local/include/stlink/sg.h
-- Installing: /usr/local/include/stlink/usb.h
-- Installing: /usr/local/share/man/man1/st-util.1
-- Installing: /usr/local/share/man/man1/st-flash.1
-- Installing: /usr/local/share/man/man1/st-info.1
```

在执行以下命令使动态库生效：

```
sudo ldconfig
```

## 1.3 查看版本信息


```
st-info --version
```

输出

```
v1.5.1-50-g3690de9
```


# 二. 命令安装


```
yay -S stlink
```


