---
layout: post
title:  "linux音频播放"
date:   2019-05-12 16:28:56
catalog:  true
author: Sourcelink
tags:
    - audio
    - Linux
    - alsa
    - aplay
    - amixer

---

# 一. 基本概念

这里了解一下各个参数的含义以及一些基本概念; 

- **样本长度(量化位数)**

样本是记录音频数据最基本的单位, 计算机对每个通道采样量化时数字比特位数; 一般为8位或16位;  

- **通道数**

单双通道, 1表示单通道, 2表示双通道;  

- **帧**

帧记录了一个声音单元, 其长度为样本长度与通道数的乘积; 一段音频数据就是由苦干帧组成;  

- **采样率**

每秒钟采样次数, 该次数是针对帧而言, 常用的采样率如8KHz的人声,  16KHz的TTS合成音, 44.1KHz的mp3音乐, 96Khz的蓝光音频; 

- **周期时间**

音频设备一次处理的长度数据, 单位为微妙, 一般设置为125000us;  
一般会将该时间换算成帧数, 即音频设备一次处理所需要的桢数, 对于音频设备的数据访问以及音频数据的存储, 都是以此为单位;  
比如125000us, 在采样率16000, 量化位数为16,单通道情况下, 换算成帧数则为2000帧, 即换算成帧数和采样率, 样本长度, 通道数有关;  

- **交错模式**

是一种音频数据的记录方式, 在交错模式下, 数据以连续桢的形式存放, 即首先记录完桢1的左声道样本和右声道样本(假设为立体声格式), 再开始桢2的记录;  
而在非交错模式下, 首先记录的是一个周期内所有桢的左声道样本, 再记录右声道样本, 数据是以连续通道的方式存储;  

- **缓存时间**

音频驱动开辟的缓存大小, 单位为微妙;  
具体需要分配内存大小的计算方式与周期时间相同, 以帧数为单位;  

# 二. 交叉编译alsa-lib

## 2.1 下载资源

```
wget ftp://ftp.alsa-project.org/pub/lib/alsa-lib-1.1.8.tar.bz2
```

再解压:  

```
tar -xf alsa-lib-1.1.8.tar.bz2
```

## 2.2 配置并编译

- 配置环境 :  

```
cd alsa-lib-1.1.8
./configure –host=arm-none-linux-gnueabi –prefix=/home/sourcelink/opt/arm-sound
```

其中`–host`指定交叉工具链, `–prefix`指定安装目录, 该目录需要自己创建;  

指定安装目录还有一种方案是直接安装到你的交叉工具链中, 例如`–prefix=/opt/friendly/4.4.1/arm-none-linux-gnueabi/libc/usr/`
这样接下交叉编译`alsa-utils`就比较安逸了;  

- 编译并安装:  

```
make && make install 
```


# 三. 交叉编译alsa-utils

## 2.1 下载资源

```
wget ftp://ftp.alsa-project.org/pub/utils/alsa-utils-1.1.8.tar.bz2
```

再解压:  

```
tar -xf alsa-utils-1.1.8.tar.bz2
```

## 2.2 配置并编译

- 配置环境 :  

```
cd alsa-utils-1.1.8
./configure –host=arm-none-linux-gnueabi 
```

这样做的前提是你已经把前面编译的alsa-lib库安装到了交叉编译器中; 否则需要使用如下配置命令:  

```
./configure –host=arm-none-linux-gnueabi CFLAGS="-I/home/sourcelink/opt/arm-sound/include"  LDFLAGS="-L/home/sourcelink/opt/arm-sound/lib"
```

- 编译并安装:  

```
make
```

# 四. 命令使用

## 4.1 aplay

### 4.1.1 查看声卡设备

```
aplay -l
```

输出结果:  

```
**** List of PLAYBACK Hardware Devices ****
card 0: HDMI [HDA Intel HDMI], device 3: HDMI 0 [HDMI 0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: HDMI [HDA Intel HDMI], device 7: HDMI 1 [HDMI 1]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: HDMI [HDA Intel HDMI], device 8: HDMI 2 [HDMI 2]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: PCH [HDA Intel PCH], device 0: ALC282 Analog [ALC282 Analog]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

注意`card`和`device`的值;  


### 4.1.2 PCM文件播放

```
aplay -t raw -c 1 -f S16_LE -r 16000 bz_woman343.pcm
```


> -t: type raw表示是PCM  
-c: channel 1  
-f S16_LE: Signed 16bit-width Little-Endian  
-r: sample rate 8000  


## 4.2 amixer

### 4.2.1 查看控制单元

```
amixer controls
```

输出结果如下:  

```
numid=1,iface=CARD,name='HDMI/DP,pcm=3 Jack'
numid=7,iface=CARD,name='HDMI/DP,pcm=7 Jack'
numid=13,iface=CARD,name='HDMI/DP,pcm=8 Jack'
numid=2,iface=MIXER,name='IEC958 Playback Con Mask'
numid=8,iface=MIXER,name='IEC958 Playback Con Mask',index=1
numid=14,iface=MIXER,name='IEC958 Playback Con Mask',index=2
numid=3,iface=MIXER,name='IEC958 Playback Pro Mask'
numid=9,iface=MIXER,name='IEC958 Playback Pro Mask',index=1
numid=15,iface=MIXER,name='IEC958 Playback Pro Mask',index=2
numid=4,iface=MIXER,name='IEC958 Playback Default'
numid=10,iface=MIXER,name='IEC958 Playback Default',index=1
numid=16,iface=MIXER,name='IEC958 Playback Default',index=2
numid=5,iface=MIXER,name='IEC958 Playback Switch'
numid=11,iface=MIXER,name='IEC958 Playback Switch',index=1
numid=17,iface=MIXER,name='IEC958 Playback Switch',index=2
numid=6,iface=PCM,name='ELD',device=3
numid=19,iface=PCM,name='Playback Channel Map',device=3
numid=12,iface=PCM,name='ELD',device=7
numid=20,iface=PCM,name='Playback Channel Map',device=7
numid=18,iface=PCM,name='ELD',device=8
numid=21,iface=PCM,name='Playback Channel Map',device=8
```

该命令打印出的结果是默认声卡的结果, 你还可以指定声卡打印信息, 实例如下: 

```
amixer -c 1 controls
```

`-c 1` 表示指定声卡１;  


### 4.2.2 音量设置

- 获取音量

```
amixer cget numid=$id
```

示例如下: 

```
amixer -c 1 cget numid=3
```

结果输出: 

```
numid=3,iface=MIXER,name='Speaker Playback Volume'
  ; type=INTEGER,access=rw---R--,values=2,min=0,max=87,step=0
  : values=87,87
  | dBscale-min=-65.25dB,step=0.75dB,mute=0
```

- 设置音量

```
amixer cset numid=$id $val
```

示例如下: 

```
amixer -c 1 cset numid=3 55
```

结果输出:  

```
numid=3,iface=MIXER,name='Speaker Playback Volume'
  ; type=INTEGER,access=rw---R--,values=2,min=0,max=87,step=0
  : values=55,55
  | dBscale-min=-65.25dB,step=0.75dB,mute=0
```

### 4.2.3 静音

```
amixer -c 1 sset 'Speaker',0 100%,100% mute
```

### 4.2.4 解静音

```
amixer -c 1 sset 'PCM',0 100%,100% unmute
```

### 4.2.5 查看声卡详细信息

```
amixer scontents
```

指定声卡查看

```
amixer -c ${cardid} scontents
```



**PS:**  

- 将wav文件转换成单通道16K pcm文件:  

```
ffmpeg -y  -i 1.wav  -acodec pcm_s16le -f s16le -ac 1 -ar 16000 1.pcm 
```


- wav文件格式信息:  `https://zhuanlan.zhihu.com/p/45518641`


