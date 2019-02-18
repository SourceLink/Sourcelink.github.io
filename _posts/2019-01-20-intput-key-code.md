---
layout: post
title:  "Input系统之键值映射"
date:   2019-01-20  17:16:54
catalog:  true
tags:
    - android
    - input
    - keyboard
---

# 一. 概述

android系统的输入事件来源在linux内核提供的`/dev/input`的设备节点下, 当该设备下及诶点有数据刻度时,将数据独处并进行一系列的翻译和加工,然后在所有的窗口中寻找合适的接受者,并派发给它;

输入系统总体流程如下(引之深入理解android卷3 ): 

![输入系统总体流程](/images/input/keycode/input_system_process.png)


## 1.1 开发环境

- 系统: ubuntu 16.04
- 运行环境: firefly-rk3288
- android版本: arm-5.1.0


# 二. 准备工作

## 2.1 模拟输入事件

为了接下来讲解原理方便, 在这里模拟一个输入设备, 方法-写一个驱动; 

- 驱动源码  

```
/* 参考drivers\input\keyboard\gpio_keys.c */

#include <linux/module.h>
#include <linux/version.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/input.h>

static struct input_dev *input_emulator_dev;

static int input_emulator_init(void)
{
	int i;
	int ret;

	/* 1. 分配一个input_dev结构体 */
	input_emulator_dev = input_allocate_device();

	/* 2. 设置 */
	/* 2.1 能产生哪类事件 */
	set_bit(EV_KEY, input_emulator_dev->evbit);
	set_bit(EV_REP, input_emulator_dev->evbit);
	
	/* 2.2 能产生所有的按键 */
	for (i = 0; i < BITS_TO_LONGS(KEY_CNT); i++)
		input_emulator_dev->keybit[i] = ~0UL;

	/* 2.3 为android构造一些设备信息 */
	input_emulator_dev->name = "smart remote";
	input_emulator_dev->id.bustype = 1;
	input_emulator_dev->id.vendor  = 0x0001;
	input_emulator_dev->id.product = 0x0010;
	input_emulator_dev->id.version = 1;

	/* 3. 注册 */
	ret = input_register_device(input_emulator_dev);
	if (ret < 0) {
		return -1;
	}
	
	return 0;
}

static void input_emulator_exit(void)
{
	input_unregister_device(input_emulator_dev);
	input_free_device(input_emulator_dev);	
}

module_init(input_emulator_init);
module_exit(input_emulator_exit);
MODULE_LICENSE("GPL");
```

- makefile  

```
KERN_DIR = /media/sourcelink/Backups/5.CompileCode/firefly3288/kernel

all:
	make -C $(KERN_DIR) M=`pwd` modules 

clean:
	make -C $(KERN_DIR) M=`pwd` modules clean
	rm -rf modules.order

obj-m	+= smart_remote.o
```

编译完后,放到板子上加载,使用命令`cat /proc/bus/input/devices`查看设备加载情况如下:  

```
root@firefly:/data/nfs # ins
insmod    installd  
root@firefly:/data/nfs # insmod smart_remote.ko
root@firefly:/data/nfs # cat /proc/bus/input/devices

....
I: Bus=0001 Vendor=0001 Product=0010 Version=0001
N: Name="smart remote"
P: Phys=
S: Sysfs=/devices/virtual/input/input3
U: Uniq=
H: Handlers=sysrq event3 ddr_freq keychord 
B: PROP=0
B: EV=100003
B: KEY=ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff fffffffe

```

如上是我板子加载完驱动的情况,原默认的信息我已经删除, 也可以看下在<font color=red>/dev/input</font>, 节点下多了个<font color=red>event3</font>;  

```
root@firefly:/data/nfs # ls /dev/input/
event0
event1
event2
event3
```

这样就完成一个输入设备的模拟,接下来介绍两个工具模拟事件的产生;  


## 2.2 工具使用

android系统提供了<font color=red>getevent</font> 与<font color=red>sendevent</font>两个工具供开发从设备节点中读取和写入事件;  

- getevent  

语法:  

> getevent [-opera]  [节点路径]

可以使用`getevent -help` 查看一些具体的operation;  


如果不带指定设备节点, 这样会监控所有的设备节点:  

```
root@firefly:/data/nfs # getevent
add device 1: /dev/input/event3
  name:     "smart remote"
add device 2: /dev/input/event2
  name:     "RK_ES8323 Headphone Jack"
add device 3: /dev/input/event1
  name:     "rk29-keypad"
add device 4: /dev/input/event0
  name:     "ff680000.pwm"
```

现在我将自己的键盘插到开发板上了,使用`getevent`命令来监控事件;  

当我按下并松开键盘上的数字`1`键获取到的数据如下:  
```
root@firefly:/data/nfs # getevent /dev/input/event4 
0004 0004 0007001e
0001 0002 00000001
0000 0000 00000000
0004 0004 0007001e
0001 0002 00000000
0000 0000 00000000
```

数据意义依次是: 事件类型, 事件代码,  事件值  

事件值的1表示按下, 0表示抬起, 观察数据可以发现每次按下或抬起都会获取到`0000 0000 00000000`的数据, 表示同步事件,通知此次事件已经结束可以进行处理了;
这里的事件代码是linux端发来的原始数据, 在android端还会进行一次键值布局,下面笔者会讲解这个映射关系;  

- setevent  

语法:  

> setevent [节点路径] [事件类型] [事件代码] [事件值]

现在操作下往设备几点写入一个事件, 打开我开发板的浏览器:

![rk3288浏览器](/images/input/keycode/rk3288_browser.png)


依次输入如下指令:  

```
sendevent /dev/input/event3 1 2 1
sendevent /dev/input/event3 1 2 0
sendevent /dev/input/event3 0 0 0
```

效果如下:  

![按键1输入效果图](/images/input/keycode/input_1_code.png)

屏幕上出现了一个数字1, 最后发送的`0 0 0`表示同步事件,通知此次事件已经结束可以进行处理了.


# 三. 按键布局和键值映射

## 3.1 Key Layout


按键事件来源于linux内核, 但是在android端对按键的值有重新做一个布局,即将linux key code转换为android key code, 这个布局依赖于个`.kl`文件, 全称: Key Layout Files;

该文件搜索路径如下:  

> /odm/usr/keylayout/Vendor_XXXX_Product_XXXX_Version_XXXX.kl  
/vendor/usr/keylayout/Vendor_XXXX_Product_XXXX_Version_XXXX.kl  
/system/usr/keylayout/Vendor_XXXX_Product_XXXX_Version_XXXX.kl  
/data/system/devices/keylayout/Vendor_XXXX_Product_XXXX_Version_XXXX.kl  
/odm/usr/keylayout/Vendor_XXXX_Product_XXXX.kl  
/vendor/usr/keylayout/Vendor_XXXX_Product_XXXX.kl  
/system/usr/keylayout/Vendor_XXXX_Product_XXXX.kl  
/data/system/devices/keylayout/Vendor_XXXX_Product_XXXX.kl  
/odm/usr/keylayout/DEVICE_NAME.kl  
/vendor/usr/keylayout/DEVICE_NAME.kl  
/system/usr/keylayout/DEVICE_NAME.kl  
/data/system/devices/keylayout/DEVICE_NAME.kl  
/odm/usr/keylayout/Generic.kl  
/vendor/usr/keylayout/Generic.kl  
/system/usr/keylayout/Generic.kl  
/data/system/devices/keylayout/Generic.kl  
    

从上面路径信息可以看出Key Layout Files文件的命名和厂家信息有关, 具体为供应商id, 产品id和设备名有关;  
比如我们的模拟输入设备的需要的`.kl`文件可以命名为`Vendor_0001_Product_0010.kl`或`smart remote.kl`;  

输入系统在检测到有新设备接入时, 在上述路径查找对应符合规则的`.kl`文件并加载它,如果没有找到则加载`Generic.kl`文件;  

- 修改测试   

拷贝一个`Generic.kl`和我们模拟按键设备名字一样, 并加上权限,如果你的板子上没有该目录的话则创建它;  

```
cp /system/usr/keylayout/Generic.kl /data/system/devices/keylayout/smart_remote.kl
chmod 777 /data/system/devices/keylayout/smart_remote.kl
```

我们打开这个文件看下里面的内容:  

```
key 1     ESCAPE
key 2     1
key 3     2
key 4     3
key 5     4
key 6     5
key 7     6
key 8     7
key 9     8
key 10    9
....
```

看到这就明白了为什么我们前面输入的键值2,最后在浏览器上看到了1; 我们修改下这个文件:  

```
key 1     ESCAPE
key 2     3
...
```

卸载驱动, 再重新加载驱动后,再依次输入如下指令:  

```
sendevent /dev/input/event3 1 2 1
sendevent /dev/input/event3 1 2 0
sendevent /dev/input/event3 0 0 0
```

效果如下:  

![更改kl文件](/images/input/keycode/after_modified_kl.png)

这样就达到了输入同样按键却得到不同之的效果了;  


## 3.2 Key Character Map

负责将android key code与修饰符的组合映射到Unicode字符。


> /odm/usr/keychars/Vendor_XXXX_Product_XXXX_Version_XXXX.kcm  
/vendor/usr/keychars/Vendor_XXXX_Product_XXXX_Version_XXXX.kcm  
/system/usr/keychars/Vendor_XXXX_Product_XXXX_Version_XXXX.kcm  
/data/system/devices/keychars/Vendor_XXXX_Product_XXXX_Version_XXXX.kcm  
/odm/usr/keychars/Vendor_XXXX_Product_XXXX.kcm  
/vendor/usr/keychars/Vendor_XXXX_Product_XXXX.kcm  
/system/usr/keychars/Vendor_XXXX_Product_XXXX.kcm  
/data/system/devices/keychars/Vendor_XXXX_Product_XXXX.kcm  
/odm/usr/keychars/DEVICE_NAME.kcm  
/vendor/usr/keychars/DEVICE_NAME.kcm  
/system/usr/keychars/DEVICE_NAME.kcm  
/data/system/devices/keychars/DEVICE_NAME.kcm  
/odm/usr/keychars/Generic.kcm  
/vendor/usr/keychars/Generic.kcm  
/system/usr/keychars/Generic.kcm  
/data/system/devices/keychars/Generic.kcm  
/odm/usr/keychars/Virtual.kcm  
/vendor/usr/keychars/Virtual.kcm  
/system/usr/keychars/Virtual.kcm  
/data/system/devices/keychars/Virtual.kcm  



输入系统在创建设备时会在这些路径下查找对应的`.kl`和`.kcm`文件进行加载, 如果没有找到对应的文件将会加载`Generic.kl`和`Generic.kcm`文件  

- 修改测试   

我们现在根据我们的模拟设备来更改下kcm文件, 操作如下:  

```
mkdir /data/system/devices/keychars
cp /system/usr/keychars/Generic.kcm /data/system/devices/keychars/smart_remote.kcm
```

打开该文件查看下里面内容:  

```
### Basic QWERTY keys ###

key A {
    label:                              'A'
    base:                               'a'
    shift, capslock:                    'A'
}

key B {
    label:                              'B'
    base:                               'b'
    shift, capslock:                    'B'
}

key C {
    label:                              'C'
    base:                               'c'
    shift, capslock:                    'C'
    alt:                                '\u00e7'
    shift+alt:                          '\u00c7'
}
....
```

以按键<font color=red>A</font>为例, 查看`.kl`文件当输入事件代码为`30`时, 对应到android的`key A`, 
根据`.kcm`文件知道此时会映射成字符`a`, 当按下`shift`键时再按下`A`会显示字符`A`;  

输入如下指令看下效果:  

```
sendevent /dev/input/event3 1 30 1
sendevent /dev/input/event3 1 30 0
sendevent /dev/input/event3 0 0 0
```

效果如下:  
![按下按键a的效果](/images/input/keycode/press_a_button.png)


为了方便大家查看都使用了文件前面的内容进行修改并演示;  


如果想当按下a键时显示字符b, 按下shift+a时显示字符2修改下kcm文件,如下:  

```
key A {
    label:                              'A'
    base:                               'b'
    shift, capslock:                    '2'
}
...
```

重新卸载驱动并加载驱动, 再次执行:  

```
sendevent /dev/input/event3 1 30 1
sendevent /dev/input/event3 1 30 0
sendevent /dev/input/event3 0 0 0
```

效果如下: 

![修改kcm](/images/input/keycode/after_modified_kcm.png)


现在试下同时按下shift键的效果, 依次输入如下:  

```
sendevent /dev/input/event3 1 42 1
sendevent /dev/input/event3 1 30 1
sendevent /dev/input/event3 1 30 0
sendevent /dev/input/event3 0 0 0
```

效果如下:  

![shift+a 修改kcm](/images/input/keycode/after_modified_kcm_shift.png)


果然和我们修改的kcm文件映射的字符保持了一致;  


## 3.3 总结

按键事件的转化流程大致如下图:   

![转化流程](/images/input/keycode/keycode_process.png)


如果想个性化定制输入的按键的键值和字符显示只需要修改kl和kcm文件;  











