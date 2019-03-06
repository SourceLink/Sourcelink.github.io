---
layout: post
title:  "nvidia+cuda安装"
date:   2019-02-22 22:10:54
catalog:  true
tags:
    - cuda
    - nvidia

---


# 一. 显卡驱动安装

## 1.1 方式一

使用ubuntu的系统设置方式更换显卡驱动, 具体如下图:  

![system](/images/ASR/nvidiaCudaInstall/system_install_nvaidia_driver.png)



## 1.2 方式二

使用apｔ更新和方式一差不多

```
sudo apt-get purge remove "nvidia-*"
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt-get update
sudo apt-get install nvidia-390
reboot 
```

## 1.3 方式三

从官网[地址](https://www.geforce.com/drivers)下载安装包进行更新,笔者的显卡为GT740M;

![driver](/images/ASR/nvidiaCudaInstall/fix_install.png)

选择对应的下载包进行下载后, 按下面步骤进行:  

- 禁用nouveau显卡

```
sudo vim /etc/modprobe.d/blacklist.conf
```

在文件最后加入如下两行:  

> blacklist nouveau  
options nouveau modeset=0

再刷新内核

```
sudo update-initramfs -u
```

再重启下电脑; 

- 关闭x-server

按`ctrl+alt+f1`进入tty模式, 登入`root`用户;  

```
service lightdm stop
```

- 安装驱动

```
sh NVIDIA-Linux-x86_64-410.93.run
```

因为笔者显卡和该版本驱动不兼容，所以重启后输入正确密码却还是在登入界面; 

该问题解决办法, 进入tty模式(ctrl+alt+f1):  

```
sudo apt-get install nvidia-current
sudo nvidia-xconfig
reboot
```

因为这个问题搞了我一下午，说多都是眼泪，推荐大家使用前两种方案; 

## 1.4 测试

使用命令`nvidia-smi`进行测试;  

```
Fri Feb 22 16:02:01 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.87                 Driver Version: 390.87                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GT 740M     Off  | 00000000:07:00.0 N/A |                  N/A |
| N/A   47C    P0    N/A /  N/A |    691MiB /  2004MiB |     N/A      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0                    Not Supported                                       |
+-----------------------------------------------------------------------------+
```


# 二. CUDA安装


从CUDA下载[地址](https://developer.nvidia.com/cuda-toolkit)下载最新的`cuda`安装包, 配置表如下: 

释放版本官方[地址](https://developer.nvidia.com/cuda-toolkit-archive)

驱动对应的CUDA版本:  

![version](/images/ASR/nvidiaCudaInstall/cuda_about_dirver_version.png)

驱动版本和cuda版本要相符合不然后面运行会提示错误:  

> CUDA driver version is insufficient for CUDA runtime version

![cuda](/images/ASR/nvidiaCudaInstall/cuda_info.png)

## 2.1 安装

根据提示使用如下命令安装:  

```
sudo sh cuda_9.1.85_387.26_linux.run
```

安装时的选项log如下:  

```
-----------------
Do you accept the previously read EULA?
accept/decline/quit: accept

Install NVIDIA Accelerated Graphics Driver for Linux-x86_64 387.26?
(y)es/(n)o/(q)uit: n

Install the CUDA 9.1 Toolkit?
(y)es/(n)o/(q)uit: y

Enter Toolkit Location
 [ default is /usr/local/cuda-9.1 ]: 

Do you want to install a symbolic link at /usr/local/cuda?
(y)es/(n)o/(q)uit: y

Install the CUDA 9.1 Samples?
(y)es/(n)o/(q)uit: y

Enter CUDA Samples Location
 [ default is /home/sourcelink ]: /home/sourcelink/work

Installing the CUDA Toolkit in /usr/local/cuda-9.1 ...
Missing recommended library: libXmu.so

Installing the CUDA Samples in /home/sourcelink/work ...
Copying samples to /home/sourcelink/work/NVIDIA_CUDA-9.1_Samples now...
Finished copying samples.

===========
= Summary =
===========

Driver:   Not Selected
Toolkit:  Installed in /usr/local/cuda-9.1
Samples:  Installed in /home/sourcelink/work, but missing recommended libraries

Please make sure that
 -   PATH includes /usr/local/cuda-9.1/bin
 -   LD_LIBRARY_PATH includes /usr/local/cuda-9.1/lib64, or, add /usr/local/cuda-9.1/lib64 to /etc/ld.so.conf and run ldconfig as root

To uninstall the CUDA Toolkit, run the uninstall script in /usr/local/cuda-9.1/bin

Please see CUDA_Installation_Guide_Linux.pdf in /usr/local/cuda-9.1/doc/pdf for detailed information on setting up CUDA.

***WARNING: Incomplete installation! This installation did not install the CUDA Driver. A driver of version at least 384.00 is required for CUDA 9.1 functionality to work.
To install the driver using this installer, run the following command, replacing <CudaInstaller> with the name of this run file:
    sudo <CudaInstaller>.run -silent -driver

Logfile is /tmp/cuda_install_3006.log
```

## 2.2 设置环境变量


```
vim ~/.bashrc
```

在文件最后添加如下两行:  

> export PATH=/usr/local/cuda-9.1/bin:$PATH  
export LD_LIBRARY_PATH=/usr/local/cuda-9.1/lib64:$LD_LIBRARY_PATH

使其生效:  

```
source ~/.bashrc
```

笔者使用的是`zsh`环境所以写在`~/.zshrc`中了;


## 2.3 测试

- 测试环境变量是否设置正确
```
nvcc -V
```

输入结果:  

```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2017 NVIDIA Corporation
Built on Fri_Nov__3_21:07:56_CDT_2017
Cuda compilation tools, release 9.1, V9.1.85
```

- 测试cuda

```
cd /home/sourcelink/work/NVIDIA_CUDA-9.1_Samples/1_Utilities/deviceQuery
sudo make
./deviceQuery
```

运行结果详情:  

![result](/images/ASR/nvidiaCudaInstall/cuda_test.png)

















