---
layout: post
title: " linux驱动中mmc分区流程"
date:   2019-06-25  17:59:56
catalog:  true
author: Sourcelink
version: v1.0
tags:
    - linux分区

---


# 一. 概述

>宿主机 ： Ubuntu 16.04 LTS / X64  
目标板：讯为4412 全功能  
uboot： 2010.03  
kernel:  3.0.15

上篇在修改了uboot分区后发现进入linux系统后分区的属性并未改变, 然后查看打印信息只有如下几点与mmc相关的信息:

```
[    4.816923] mmc0: EXT_CSD revision 8
[    4.825743] mmc0:mshci_set_clock, 50000000@ration:4,11[mjdbg] cmu_set_clock: 50000000
[    4.843483] mmc0: new high speed DDR MMC card at address 0001
[    4.848122] mmcblk0: mmc0:0001 AJTD4R 14.5 GiB 
[    4.857234]  mmcblk0: p1 p2 p3 p4
[    4.890078] *******mmc2: inserted!!!!!******
[    4.945667] *******mmc2: inserted!!!!!******
[    4.995091] *******mmc2: inserted!!!!!******
[    5.040133] *******mmc2: inserted!!!!!******
```

根据这些打印日志在内核源码中搜索了下也没有找到思路, 后来网上查了下mmc对应的驱动在`drivers/mmc/card/block.c`中, 看下代码:  

```
static struct mmc_driver mmc_driver = {
	.drv		= {
		.name	= "mmcblk",
	},
	.probe		= mmc_blk_probe,
	.remove		= mmc_blk_remove,
	.suspend	= mmc_blk_suspend,
	.resume		= mmc_blk_resume,
};
```

只能得出是个probe总线驱动, 大写的难受.....


# 二. 流程分析

probe驱动按理应该有个driver和device两个驱动, 因为对mmc驱动不了解所以半天没找到, 最后觉得还是根据日志来搜索;  
搜索关键字`card at address`, 最后发现在`mmc_add_card()`中发现了对应的打印log, 按着这个思路看下去就清晰很多了;  

## 2.1 突破口

> drivers/mmc/core/bus.c

```
int mmc_add_card(struct mmc_card *card)
{
    .....
    
	if (mmc_host_is_spi(card->host)) {
		printk(KERN_INFO "%s: new %s%s%s card on SPI\n",
			mmc_hostname(card->host),
			mmc_card_highspeed(card) ? "high speed " : "",
			mmc_card_ddr_mode(card) ? "DDR " : "",
			type);
	} else {
		printk(KERN_INFO "%s: new %s%s%s card at address %04x\n",
			mmc_hostname(card->host),
			mmc_sd_card_uhs(card) ? "ultra high speed " :
			(mmc_card_highspeed(card) ? "high speed " : ""),
			mmc_card_ddr_mode(card) ? "DDR " : "",
			type, card->rca);
	}

#ifdef CONFIG_DEBUG_FS
	mmc_add_card_debugfs(card);
#endif

	ret = device_add(&card->dev);
	if (ret)
		return ret;

	mmc_card_set_present(card);

	return 0;
}
```

可以看到打印日志就是从这个地方出来的, 接下来看下它是被谁调用了; 


## 2.2 谁调用了我?

> drivers/mmc/core/core.c

```
mmc_rescan
    mmc_rescan_try_freq
        mmc_attach_mmc
            mmc_add_card
```

以上是我搜索代码的时候找到的调用关系, 但是到这里还没完, 继续搜索`mmc_rescan`: 

> drivers/mmc/core/host.c

```
struct mmc_host *mmc_alloc_host(int extra, struct device *dev)
{
	int err;
	struct mmc_host *host;

    ....

	dev_set_name(&host->class_dev, "mmc%d", host->index);

	host->parent = dev;
	host->class_dev.parent = dev;
	host->class_dev.class = &mmc_host_class;
	device_initialize(&host->class_dev);

	mmc_host_clk_init(host);

	spin_lock_init(&host->lock);
	init_waitqueue_head(&host->wq);
	wake_lock_init(&host->detect_wake_lock, WAKE_LOCK_SUSPEND,
		kasprintf(GFP_KERNEL, "%s_detect", mmc_hostname(host)));
	INIT_DELAYED_WORK(&host->detect, mmc_rescan);
	INIT_DELAYED_WORK_DEFERRABLE(&host->disable, mmc_host_deeper_disable);
    .....
	return host;

free:
	kfree(host);
	return NULL;
}
```

**INIT_DELAYED_WORK(&host->detect, mmc_rescan)**, 可以看到这里将**mmc_rescan**加入了内核的工作线程去运行了;

在进一步的进行`mmc_alloc_host`搜索时发现, 调用关系有点多, 无法确定是谁调用了该函数, 所以我们加入打印:

```
	/* add by Sourcelink */
	printk("Sourcelink mmc_alloc_host\n");
	#if 1
	dump_stack();
	#endif
```

在`mmc_alloc_host()`中加入上面的函数, 使用`dump_stack`打印调用堆栈信息, 重新编译内核后再烧录重启信息如下:  

```
[    4.493903] Sourcelink mmc_alloc_host
[    4.497499] Backtrace: 
[    4.499932] [<c004ce88>] (dump_backtrace+0x0/0x11c) from [<c0631d54>] (dump_stack+0x18/0x1c)
[    4.508350]  r6:00000000 r5:c09bab18 r4:d6329800 r3:d6040000
[    4.513987] [<c0631d3c>] (dump_stack+0x0/0x1c) from [<c0424224>] (mmc_alloc_host+0x98/0x1f8)
[    4.522438] [<c042418c>] (mmc_alloc_host+0x0/0x1f8) from [<c04337e8>] (mshci_alloc_host+0x24/0x54)
[    4.531352]  r7:c09bab18 r6:c09baad8 r5:00000024 r4:c09bab18
[    4.536991] [<c04337c4>] (mshci_alloc_host+0x0/0x54) from [<c062ccd0>] (mshci_s3c_probe+0xa8/0x4b0)
[    4.546020]  r5:c0a678d0 r4:c09bab18
[    4.549577] [<c062cc28>] (mshci_s3c_probe+0x0/0x4b0) from [<c02c6094>] (platform_drv_probe+0x20/0x24)
[    4.558788] [<c02c6074>] (platform_drv_probe+0x0/0x24) from [<c02c4984>] (driver_probe_device+0xa4/0x1b8)
[    4.568334] [<c02c48e0>] (driver_probe_device+0x0/0x1b8) from [<c02c4b2c>] (__driver_attach+0x94/0x98)
[    4.577615]  r8:00000000 r7:00000000 r6:c09bab4c r5:c09e04a4 r4:c09bab18
[    4.584113] r3:c02c6310
[    4.586724] [<c02c4a98>] (__driver_attach+0x0/0x98) from [<c02c3dd8>] (bus_for_each_dev+0x64/0xa0)
[    4.595669]  r6:c02c4a98 r5:d6041ed0 r4:c09e04a4 r3:c09bab18
[    4.601306] [<c02c3d74>] (bus_for_each_dev+0x0/0xa0) from [<c02c47d0>] (driver_attach+0x24/0x28)
[    4.610077]  r7:00000000 r6:d60f23c0 r5:c09c9340 r4:c09e04a4
[    4.615715] [<c02c47ac>] (driver_attach+0x0/0x28) from [<c02c382c>] (bus_add_driver+0x1e4/0x2b8)
[    4.624497] [<c02c3648>] (bus_add_driver+0x0/0x2b8) from [<c02c5198>] (driver_register+0x80/0x190)
[    4.633435] [<c02c5118>] (driver_register+0x0/0x190) from [<c02c6594>] (platform_driver_register+0x64/0x6c)
[    4.643149]  r8:00000000 r7:00000013 r6:c0088c94 r5:c00394f8 r4:c0a38000
[    4.649647] r3:00000000
[    4.652261] [<c02c6530>] (platform_driver_register+0x0/0x6c) from [<c00265ec>] (mshci_s3c_init+0x18/0x1c)
[    4.661818] [<c00265d4>] (mshci_s3c_init+0x0/0x1c) from [<c0041684>] (do_one_initcall+0x3c/0x190)
[    4.670674] [<c0041648>] (do_one_initcall+0x0/0x190) from [<c000848c>] (kernel_init+0xb0/0x140)
[    4.679354] [<c00083dc>] (kernel_init+0x0/0x140) from [<c0088c94>] (do_exit+0x0/0x7b4)
[    4.687247]  r5:c00083dc r4:00000000
```

yes, 很清晰的看到了调用关系:  


```
mshci_s3c_probe
    mshci_alloc_host
        mmc_alloc_host
```

又是一个platform驱动, 驱动注册完毕后, 在检测和匹配到设备后就会调用`mshci_s3c_probe()`; 

>  drivers/mmc/core/mshci-s3c.c

```
static int __devinit mshci_s3c_probe(struct platform_device *pdev)
{
    .....
    
	host = mshci_alloc_host(dev, sizeof(struct mshci_s3c));
	if (IS_ERR(host)) {
		dev_err(dev, "mshci_alloc_host() failed\n");
		return PTR_ERR(host);
	}
	sc = mshci_priv(host);
    
    .....
```


## 2.3 我的驱动在哪里?

上一节我们看了`mmc_add_card()`被谁调用了, 现在看下它向下调用了谁?


```
mmc_add_card
    device_add
        bus_probe_device
            device_attach
                bus_for_each_drv
                    __device_attach
                        driver_match_device
                        driver_probe_device
                            really_probe
```

添加card相当于添加设备, 现在设备有了就需要为该设备匹配驱动了;

> drivers/base/dd.c

```
static int really_probe(struct device *dev, struct device_driver *drv)
{
	int ret = 0;

	......
	
	if (dev->bus->probe) {
		ret = dev->bus->probe(dev);
		if (ret)
			goto probe_failed;
	} else if (drv->probe) {
		ret = drv->probe(dev);
		if (ret)
			goto probe_failed;
	}

    ......
    
	return ret;
}
```

匹配到后需要优先执行总线上的probe函数, 其次执行驱动上的probe函数;  

因为在同一个文件中发现了有注册mmc总线:  

```
int mmc_register_bus(void)
{
	return bus_register(&mmc_bus_type);
}
```

该函数在`mmc_init()`中被调用;  


好了接下来看下mmc总线设备的probe函数中做了什么事情:  

```
static int mmc_bus_probe(struct device *dev)
{
	struct mmc_driver *drv = to_mmc_driver(dev->driver);
	struct mmc_card *card = mmc_dev_to_card(dev);

	return drv->probe(card);
}
```

执行驱动上的probe函数, 现在问题来了驱动在哪?  

在`device_add()`添加如下打印信息, 看看注册的设备名:  

```
	/* modified by Sourcelink */
	printk("Sourcelink device: '%s': %s\n", dev_name(dev), __func__);
```


重新烧录后发现了关键:   

```
Sourcelink device: 'mmc0:0001': device_add
Sourcelink device: '179:0': device_add
Sourcelink device: 'mmcblk0': device_add
```

再次添加堆栈log:

```
	if (strncmp(dev_name(dev), "mmcblk0", sizeof("mmcblk0")) == 0) {
		/* add by Sourcelink */
		printk("Sourcelink device_add\n");
		#if 1
		dump_stack();
		#endif
	}
```

重新烧录后的关键信息如下:  

```
[    6.509687] Sourcelink device: '179:0': device_add
[    6.509859] Sourcelink device: 'mmcblk0': device_add
[    6.509865] Sourcelink device_add
[    6.509869] Backtrace: 
[    6.509883] [<c004ce88>] (dump_backtrace+0x0/0x11c) from [<c0631cb4>] (dump_stack+0x18/0x1c)
[    6.509890]  r6:00000000 r5:d62d4460 r4:d62d4460 r3:d6054000


[    6.509908] [<c0631c9c>] (dump_stack+0x0/0x1c) from [<c02c2708>] (device_add+0x564/0x5c8)


[    6.509923] [<c02c21a4>] (device_add+0x0/0x5c8) from [<c026374c>] (register_disk+0x40/0x16c)
[    6.509936] [<c026370c>] (register_disk+0x0/0x16c) from [<c026392c>] (add_disk+0xb4/0x2bc)
[    6.509943]  r7:d6055d9c r6:d6364000 r5:c0a64f98 r4:d62d4400
[    6.509962] [<c0263878>] (add_disk+0x0/0x2bc) from [<c042fbd4>] (mmc_add_disk+0x18/0x6c)
[    6.509969]  r8:00000000 r7:d6055d9c r6:00000000 r5:d62d4000 r4:d609f680
[    6.509989] [<c042fbbc>] (mmc_add_disk+0x0/0x6c) from [<c04300b4>] (mmc_blk_probe+0x408/0x45c)
[    6.509995]  r5:d62d4000 r4:d609f680
[    6.510008] [<c042fcac>] (mmc_blk_probe+0x0/0x45c) from [<c04238fc>] (mmc_bus_probe+0x20/0x24)
[    6.510038] [<c04238dc>] (mmc_bus_probe+0x0/0x24) from [<c02c48e0>] (driver_probe_device+0xa4/0x1b8)
[    6.510052] [<c02c483c>] (driver_probe_device+0x0/0x1b8) from [<c02c4ad0>] (__device_attach+0x44/0x48)
[    6.510059]  r8:00000000 r7:00000000 r6:c02c4a8c r5:d62d4008 r4:c09e03ec
[    6.510071] r3:c04238c8
[    6.510082] [<c02c4a8c>] (__device_attach+0x0/0x48) from [<c02c3bcc>] (bus_for_each_drv+0x64/0xa0)
[    6.510089]  r5:d6055e28 r4:d62d4008
[    6.510101] [<c02c3b68>] (bus_for_each_drv+0x0/0xa0) from [<c02c4ba0>] (device_attach+0xa0/0xc0)
[    6.510108]  r7:d6229808 r6:00000000 r5:d62d403c r4:d62d4008
[    6.510124] [<c02c4b00>] (device_attach+0x0/0xc0) from [<c02c3964>] (bus_probe_device+0x2c/0x48)
[    6.510131]  r6:00000000 r5:d62d4010 r4:d62d4008 r3:00000001


[    6.510146] [<c02c3938>] (bus_probe_device+0x0/0x48) from [<c02c25d8>] (device_add+0x434/0x5c8)


[    6.510158] [<c02c21a4>] (device_add+0x0/0x5c8) from [<c0423b14>] (mmc_add_card+0xdc/0x1ec)
[    6.510169] [<c0423a38>] (mmc_add_card+0x0/0x1ec) from [<c04257b8>] (mmc_attach_mmc+0xbc/0x1d0)
[    6.510176]  r5:00000000 r4:d6229800
[    6.510188] [<c04256fc>] (mmc_attach_mmc+0x0/0x1d0) from [<c04231d4>] (mmc_rescan+0x2f8/0x528)
[    6.510194]  r6:00000000 r5:00061a80 r4:d6229800
[    6.510212] [<c0422edc>] (mmc_rescan+0x0/0x528) from [<c009d6ac>] (process_one_work+0x134/0x52c)
[    6.510219]  r6:d63d4a00 r5:c0a5c520 r4:d60032c0 r3:c0422edc
[    6.510237] [<c009d578>] (process_one_work+0x0/0x52c) from [<c009de40>] (worker_thread+0x18c/0x4ac)
[    6.510252] [<c009dcb4>] (worker_thread+0x0/0x4ac) from [<c00a4d18>] (kthread+0x94/0x98)
[    6.510266] [<c00a4c84>] (kthread+0x0/0x98) from [<c0088c94>] (do_exit+0x0/0x7b4)
```

看到堆栈信息中有调用两次`device_add()`, 第一次调用是注册了设备号`179:0`, 被`mmc_add_card()`调用, 第二次是注册了设备`mmcblk0`, 被`register_disk()`调用;  

现在我们知道和设备`mmcblk0`有关的驱动是`mmc_blk_probe()`函数, 也就是第一章中`drivers/mmc/card/block.c`下的驱动代码, 分析到这思路更加清晰了驱动也找到了;  

## 2.4 驱动probe做了什么?

```
mmc_blk_probe
    mmc_add_disk
        add_disk
            register_disk
                blkdev_get
                    __blkdev_get
                        rescan_partitions
                            check_partition
                                check_part[i++](state);   // msdos_partition
```

为了验证流程是否正确可以在`msdos_partition()`中添加打印信息;  

看到这函数中一条关键的信息, 看到这里恍然大悟: 

```
p = (struct partition *) (data + 0x1be);
```

内核读取mbr信息是从0x1be开始的, 我们在uboot中已经修改了mbr信息, 所以这里也得修改成0x1ae, for循环的大小也需要修改为5, 毕竟现在有5个分区;  


# 三. 总结


该版本内核的分区信息来自于mbr, mbr的分区信息可以在uboot中修改, 在内核启动后加载mmc设备后会读取mbr中的信息, 根据读取到的mbr信息进行分区的创建;  
前面的分析流程仅仅只是抛砖引玉, 大家可以重点看下第二章的第四小结;




