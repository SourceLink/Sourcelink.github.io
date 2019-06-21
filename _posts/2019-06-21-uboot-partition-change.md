---
layout: post
title:  "uboot分区修改"
date:   2019-06-21 16:30:56
catalog:  true
author: Sourcelink
version: v1.0
tags:
    - uboot
    - 分区

---


# 一. 概述

>宿主机 ： Ubuntu 16.04 LTS / X64
目标板：讯为4412 全功能
uboot： 2010.03

因为讯为提供的uboot实在是太老了，新增或删减分区还必须得修改代码才能实现，研究了一早上代码， 自己也修改测试过，现在记录下修改的过程和分区原理；

- 查看分区

```
fdisk -p 0
```

0表示存储设备0；

- 分区

```
fdisk -c 0 1024 300 300 
```

1024， 300 ，300表示前三个分区的大小

# 二. 分区

> cmd__mmc_fdisk.c

## 2.1 create_mmc_fdisk


```
int create_mmc_fdisk(int argc, char *argv[])
{
	int		rv;
	int		total_block_count;
	unsigned char	mbr[512];

	memset(mbr, 0x00, 512);
	
	total_block_count = get_mmc_block_count(argv[2]);                     ①
	if (total_block_count < 0)
		return -1;

	/* modified by Sourcelink */
	#if 0
	make_mmc_partition(total_block_count, mbr, (argc==6?1:0), argv);
	#else 
	make_mmc_partition(total_block_count, mbr, 1, argv);                  ②
	#endif

	/* save partition info to mbr */
	rv = put_mmc_mbr(mbr, argv[2]);                                       ③
	if (rv != 0)
		return -1;
		
	printf("fdisk is completed\n");

	argv[1][1] = 'p';
	print_mmc_part_info(argc, argv);                                      ④
	return 0;
}
```

> ①: 根据输入的设备号获取存储设备的块数， 每个块的大小为512字节
> ②: 根据输入参数创建分区，创建的分区信息存放在mbr中
> ③: 将mbr存放进存储设备中
> ④: 打印分区信息

分区的流程基本就是上述几步，接下来重点讲解下如何分区；


## 2.2 make_mmc_partition

```
int make_mmc_partition(int total_block_count, unsigned char *mbr, int flag, char *argv[])
{
	unsigned int		block_start = 0, block_offset;

	SDInfo		sdInfo;
	/* modified by Sourcelink */
	PartitionInfo	partInfo[5];


	memset((unsigned char *)&sdInfo, 0x00, sizeof(SDInfo));


	get_SDInfo(total_block_count, &sdInfo);                                                ①


	block_start	= calc_unit(DISK_START, sdInfo);

	if (flag)
		block_offset = calc_unit((unsigned long long)simple_strtoul(argv[3], NULL, 0)*1024*1024, sdInfo);
	else
		block_offset = calc_unit(SYSTEM_PART_SIZE, sdInfo);

	partInfo[0].bootable	= 0x00;
	partInfo[0].partitionId	= 0x83;

	make_partitionInfo(block_start, block_offset, sdInfo, &partInfo[0]);

	block_start += block_offset;

    ......

	/* add by Sourcelink */
	block_start += block_offset;
	block_offset = calc_unit(CACHE_PART_SIZE, sdInfo);                                    ②

	partInfo[3].bootable	= 0x00;
	partInfo[3].partitionId	= 0x83;                                                       ③

	make_partitionInfo(block_start, block_offset, sdInfo, &partInfo[3]);                  ④
	/*********************/

	block_start += block_offset;
	block_offset = BLOCK_END;

	partInfo[4].bootable	= 0x00;
	partInfo[4].partitionId	= 0x0C;

	make_partitionInfo(block_start, block_offset, sdInfo, &partInfo[4]);


	memset(mbr, 0x00, sizeof(*mbr)*512);// liang, clean the mem again
	mbr[510] = 0x55; mbr[511] = 0xAA;                                                     ⑤

	encode_partitionInfo(partInfo[0], &mbr[0x1CE]);                                       ⑥
	encode_partitionInfo(partInfo[1], &mbr[0x1DE]);
	encode_partitionInfo(partInfo[2], &mbr[0x1EE]);
	encode_partitionInfo(partInfo[3], &mbr[0x1AE]);
	/* modified by Sourcelink */
	encode_partitionInfo(partInfo[4], &mbr[0x1BE]);

	return 0;
}
```

上面贴出来的代码有删减；

> ①: 获取mmc的寻址方式（LBA or CHS）
> ②: 根据要设置的分区大小计算块偏移值
> ③: 设置分区信息（这个id比较重要）
> ④: 设置分区信息，如块起始地址，偏移值等
> ⑤: 设置mbr校检信息，这个相当于校检头后面一些读取分区信息会使用到
> ⑥: 将partInfo里的信息保存到mbr中，可以看出mbr的保存方式是个置顶向下的栈方式

可以看到我这里新增了一个分区存储在`mbr[0x1AE]`；


## 2.3 encode_partitionInfo

```
void encode_partitionInfo(PartitionInfo partInfo, unsigned char *result)
{
	*result++ = partInfo.bootable;

	encode_chs(partInfo.C_start, partInfo.H_start, partInfo.S_start, result);
	result +=3;
	*result++ = partInfo.partitionId;

	encode_chs(partInfo.C_end, partInfo.H_end, partInfo.S_end, result);
	result += 3;

	memcpy(result, (unsigned char *)&(partInfo.block_start), 4);
	result += 4;	
	
	memcpy(result, (unsigned char *)&(partInfo.block_count), 4);
}
```

这个函数将前面保存在`partInfo`的信息编码存储在`result`中，存储宽度为16个字节；

## 2.4 get_mmc_part_info

```
int get_mmc_part_info(char *device_name, int part_num, int *block_start, int *block_count, unsigned char *part_Id)
{
	int		rv;
	PartitionInfo	partInfo;
	unsigned char	mbr[512];
	
	rv = get_mmc_mbr(device_name, mbr);
	if(rv !=0)
		return -1;
				
	switch(part_num)
	{
		case 1:
			decode_partitionInfo(&mbr[0x1BE], &partInfo);
			*block_start	= partInfo.block_start;	
			*block_count	= partInfo.block_count;	
			*part_Id 	= partInfo.partitionId;	
			break;
		case 2:
			decode_partitionInfo(&mbr[0x1CE], &partInfo);
			*block_start	= partInfo.block_start;	
			*block_count	= partInfo.block_count;	
			*part_Id 	= partInfo.partitionId;	
			break;
		
		case 3:
			decode_partitionInfo(&mbr[0x1DE], &partInfo);
			*block_start	= partInfo.block_start;	
			*block_count	= partInfo.block_count;	
			*part_Id 	= partInfo.partitionId;	
			break;
		case 4:
			decode_partitionInfo(&mbr[0x1EE], &partInfo);
			*block_start	= partInfo.block_start;	
			*block_count	= partInfo.block_count;	
			*part_Id 	= partInfo.partitionId;	
			break;
		case 5:
			/* add by Sourcelink */
			decode_partitionInfo(&mbr[0x1AE], &partInfo);
			*block_start	= partInfo.block_start;	
			*block_count	= partInfo.block_count;	
			*part_Id 	= partInfo.partitionId;	
			break;
		default:
			return -1;
	}	

	return 0;
}
```

通过该函数获取分区信息可以看到分区1对应的是`mbr[0x1BE]`中的存储信息， 我们新增的分区5存储的是`mbr[0x1AE]`的信息；


# 三. 注意事项

## 3.1 fastboot

为什么说那个id信息很重要，在一开始我新增的分区时将`mbr[0x1BE]`中的id修改成了`0x83`,  `mbr[0x1AE]`中的id修改成了`0x0c`, 在修改了上述东西后，发现fastboot不能用了，提示信息如下：

> Error: No MBR is found at SD/MMC.
> Hint: use fdisk command to make partitions.

后面查看代码发现了问题，在函数：

> cmd_fastboot.c

```
static int set_partition_table_sdmmc()
{
	int start, count;
	unsigned char pid;

    ....

	/* System */
	get_mmc_part_info((dev_number_write == 0)?"0":"1", 2, &start, &count, &pid);
	if (pid != 0x83)
		goto part_type_error;
	strcpy(ptable[pcount].name, "system");
	ptable[pcount].start = start * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].length = count * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].flags = FASTBOOT_PTENTRY_FLAGS_USE_MMC_CMD;
	pcount++;

	/* Data */
	get_mmc_part_info((dev_number_write == 0)?"0":"1", 3, &start, &count, &pid);
	if (pid != 0x83)
		goto part_type_error;
	strcpy(ptable[pcount].name, "userdata");
	ptable[pcount].start = start * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].length = count * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].flags = FASTBOOT_PTENTRY_FLAGS_USE_MMC_CMD;
	pcount++;

	/* Cache */
	get_mmc_part_info((dev_number_write == 0)?"0":"1", 4, &start, &count, &pid);
	if (pid != 0x83)
		goto part_type_error;
	strcpy(ptable[pcount].name, "cache");
	ptable[pcount].start = start * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].length = count * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].flags = FASTBOOT_PTENTRY_FLAGS_USE_MMC_CMD;
	pcount++;

	/* fat */
	get_mmc_part_info((dev_number_write == 0)?"0":"1", 1, &start, &count, &pid);
	if (pid != 0xc)
		goto part_type_error;
	strcpy(ptable[pcount].name, "fat");
	ptable[pcount].start = start * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].length = count * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].flags = FASTBOOT_PTENTRY_FLAGS_USE_MMC_CMD;
	pcount++;

    ......

part_type_error:
	printf("Error: No MBR is found at SD/MMC.\n");
	printf("Hint: use fdisk command to make partitions.\n");

	return -1;
}
```

可以看出这里做了`id`的判断， 因为我们修改分区信息导致id判断失败从而使fastboot不能使用；

所以在后面增加分区的时候为了尽可能小的改动， 所以在`mbr[0x1AE]`中将id修改为0x83，设置成新增分区， 原`mbr[0x1BE]`的分区信息保留不变；


## 3.2 ext2format

在新增分区后，使用`fdisk -p 0`时可以看到分区信息：

```
partion #    size(MB)     block start #    block count    partition_Id 
   1         12960          3973120        26542080          0x0C 
   2          1024            32768         2097152          0x83 
   3           300          2129920          614400          0x83 
   4           300          2744320          614400          0x83 
   5           300          3358720          614400          0x83
```

一个新增的大小为300MByte的分区；

但是在使用`ext2format mmc 0:5`命令在格式化第五个分区的时候出现了问题， 在查看代码后发现问题所在：

涉及到的源文件如下：

> ext2fs.c
> part.c
> part_dos.c

函数调用关系如下：

```
do_ext2_format
    ext_format
        ext2fs_format
            get_partition_info
                get_partition_info_dos
                    get_partition_info_extended
```


关键函数如下：

```
static int get_partition_info_extended (block_dev_desc_t *dev_desc, int ext_part_sector,
				 int relative, int part_num,
				 int which_part, disk_partition_t *info)
{
	unsigned char buffer[DEFAULT_SECTOR_SIZE];
	dos_partition_t *pt;
	int i;

	/* add by Sourcelink */
	printf("get_partition_info_extended dev_desc->dev = %d\n", dev_desc->dev);
	
	if (dev_desc->block_read (dev_desc->dev, ext_part_sector, 1, (ulong *) buffer) != 1) {
		printf ("** Can't read partition table on %d:%d **\n",
			dev_desc->dev, ext_part_sector);
		return -1;
	}
	if (buffer[DOS_PART_MAGIC_OFFSET] != 0x55 ||
		buffer[DOS_PART_MAGIC_OFFSET + 1] != 0xaa) {
		printf ("bad MBR sector signature 0x%02x%02x\n",
			buffer[DOS_PART_MAGIC_OFFSET],
			buffer[DOS_PART_MAGIC_OFFSET + 1]);
		return -1;
	}

	/* Print all primary/logical partitions */
	pt = (dos_partition_t *) (buffer + DOS_PART_TBL_OFFSET);
	for (i = 0; i < 5; i++, pt++) {
		/*
		 * fdisk does not show the extended partitions that
		 * are not in the MBR
		 */
		if (((pt->boot_ind & ~0x80) == 0) &&
		    (pt->sys_ind != 0) &&
		    (part_num == which_part) &&
		    (is_extended(pt->sys_ind) == 0)) {
			info->blksz = 512;
			info->start = ext_part_sector + le32_to_int (pt->start4);
```

关键的是`DOS_PART_TBL_OFFSET`这个偏移值，因为pt的起始操作地址和该数值有关查看它的值发现竟然是0x1be，这样当然无法格式化我们新增的分区，需要修改为0x1ae，如下：

```
/* modified by Sourcelink */
#define DOS_PART_TBL_OFFSET	0x1ae
```

# 四. 支持fastboot

新增的分区需要支持fastboot下载还需要修改fastboot分区表：

> cmd_fastboot.c-----> static int set_partition_table_sdmmc()

```
	get_mmc_part_info((dev_number_write == 0)?"0":"1", 5, &start, &count, &pid);
	if (pid != 0x83)
		goto part_type_error;
	strcpy(ptable[pcount].name, "dtb");
	ptable[pcount].start = start * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].length = count * CFG_FASTBOOT_SDMMC_BLOCKSIZE;
	ptable[pcount].flags = FASTBOOT_PTENTRY_FLAGS_USE_MMC_CMD;
	pcount++;
```

在`set_partition_table_sdmmc`函数中添加上述代码，后重新编译再次执行`fastboot`时就可以看到新增了一个分区信息；

```
iTOP-4412 # fastboot
[Partition table on MoviNAND]
ptn 0 name='bootloader' start=0x0 len=N/A (use hard-coded info. (cmd: movi))
ptn 1 name='kernel' start=N/A len=N/A (use hard-coded info. (cmd: movi))
ptn 2 name='ramdisk' start=N/A len=0x300000(~3072KB) (use hard-coded info. (cmd: movi))
ptn 3 name='Recovery' start=N/A len=0x600000(~6144KB) (use hard-coded info. (cmd: movi))
ptn 4 name='system' start=0x1000000 len=0x40000000(~1048576KB) 
ptn 5 name='userdata' start=0x41000000 len=0x12C00000(~307200KB) 
ptn 6 name='cache' start=0x53C00000 len=0x12C00000(~307200KB) 
ptn 7 name='dtb' start=0x66800000 len=0xA00000(~10240KB)                 // here here
ptn 8 name='fat' start=0x67200000 len=0x3C200000(~985088KB) 
```


**PS:** 在`set_partition_table_sdmmc`函数中你会发现分区时`flags`有两个标志`FASTBOOT_PTENTRY_FLAGS_USE_MMC_CMD`和`FASTBOOT_PTENTRY_FLAGS_USE_MOVI_CMD`；
哪这个两个标志有什么区别？

在查看了源代码的时候发现了，当标志为`FASTBOOT_PTENTRY_FLAGS_USE_MMC_CMD`时进行fastboot更新分区的时候是调用`do_mmcops()`进行写入操作，而另一个标志则是使用`do_movi()`进行相关操作；  











