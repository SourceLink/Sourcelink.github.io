---
layout: post
title:  "kaldi安装"
date:   2019-02-18 22:10:54
catalog:  true
tags:
    - kaldi
    - ASR
---

# 一. 源码下载

```
git clone https://github.com/kaldi-asr/kaldi.git kaldi-trunnk
```

将源码clone到了`kaldi-trunnk`文件夹中, 接下来的操作都是相对于`kaldi-trunnk`目录;  

# 二. tool安装


## 2.1 检测依赖  

输入如下指令进行检测:  

```
cd kaldi-trunnk/tools/extras
./check_dependencies.sh
```

输入结果如下:  

```
./check_dependencies.sh: sox is not installed.
./check_dependencies.sh: we recommend that you run (our best guess):
 sudo apt-get install  sox
You should probably do: 
 sudo apt-get install libatlas3-base
```

输入如下命令进行依赖安装:  

```
sudo apt-get install  sox
sudo apt-get install libatlas3-base
```


再次检测依赖:  

```
./check_dependencies.sh
./check_dependencies.sh: all OK.
```


## 2.2 编译tools

输入如下指令进行编译:  

```
cd kaldi-trunnk/tools
make
```

编译过程log:   

```
touch sctk/.configured
cd sctk; \
make CC="gcc         " CXX="g++" all && make install && make doc
make[1]: Entering directory '/home/sourcelink/work/test/kaldi-trunnk/tools/sctk-2.4.10'
(mkdir -p bin)
(cd src; make all)
make[2]: Entering directory '/home/sourcelink/work/test/kaldi-trunnk/tools/sctk-2.4.10/src'
(cd asclite; make all)
make[3]: Entering directory '/home/sourcelink/work/test/kaldi-trunnk/tools/sctk-2.4.10/src/asclite'
(cd core; make all)
make[4]: Entering directory '/home/sourcelink/work/test/kaldi-trunnk/tools/sctk-2.4.10/src/asclite/core'
gcc          -c  -DPACKAGE_NAME=\"\" -DPACKAGE_TARNAME=\"\" -DPACKAGE_VERSION=\"\" -DPACKAGE_STRING=\"\" -DPACKAGE_BUGREPORT=\"\" -DHAVE_LIBM=1 -DSTDC_HEADERS=1 -DHAVE_SYS_TYPES_H=1 -DHAVE_SYS_STAT_H=1 -DHAVE_STDLIB_H=1 -DHAVE_STRING_H=1 -DHAVE_MEMORY_H=1 -DHAVE_STRINGS_H=1 -DHAVE_INTTYPES_H=1 -DHAVE_STDINT_H=1 -DHAVE_UNISTD_H=1 -DHAVE_UNISTD_H=1 -DHAVE_STRING_H=1 -
```

编译完成且成功结果:  

```
Warning: IRSTLM is not installed by default anymore. If you need IRSTLM
Warning: use the script extras/install_irstlm.sh
All done OK
```

结果中有个警告提示`IRSTLM`默认未安装, 安装情况详见2.3节;  

## 2.3 IRSTLM 安装

先尝试安装:  

```
cd kaldi-trunnk/tools/extras
```

```
./install_irstlm.sh 
****() Installing IRSTLM
****** You are trying to install IRSTLM from the wrong directory.  You should
****** go to tools/ and type extras/install_irstlm.sh.
```

这提示了一个错误, 按照提示执行: 

```
cd kaldi-trunnk/tools
./extras/install_irstlm.sh
```

完成结果如下:  

```
***() Installation of IRSTLM finished successfully
***() Please source the tools/extras/env.sh in your path.sh to enable it
```

根据它的log输出我并没有找到`extras`目录下找到`env.h`, 而是在`tools`目录下:  

![ENV.SH](/images/ASR/kaldiInstall/path_env.sh.png)

想使用`IRSTLM`就需要在`path.sh`中执行`env.sh`脚本;  


# 三. src安装


## 3.1 查看安装教程

```
 cat INSTALL 

These instructions are valid for UNIX-like systems (these steps have
been run on various Linux distributions; Darwin; Cygwin).  For native Windows
compilation, see ../windows/INSTALL.

You must first have completed the installation steps in ../tools/INSTALL
(compiling OpenFst; getting ATLAS and CLAPACK headers).

The installation instructions are

  ./configure --shared
  make depend -j 8
  make -j 8

Note that we added the "-j 8" to run in parallel because "make" takes a long
time.  8 jobs might be too many for a laptop or small desktop machine with not
many cores.

For more information, see documentation at http://kaldi-asr.org/doc/
and click on "The build process (how Kaldi is compiled)".

```

## ３.2 按部就班

> 编译原则: 核心＊２

输入`nproc`命令, 查看核心数:  
![nproc](/images/ASR/kaldiInstall/nproc.png)

依次输入如下命令:  

```
  ./configure --shared
  make depend -j8
  make -j8
```

编译完成结果:  

```
.so /usr/lib/libatlas.so.3 /usr/lib/libf77blas.so.3 /usr/lib/libcblas.so.3 /usr/lib/liblapack_atlas.so.3 -Wl,-rpath=/usr/lib -lm -lpthread -ldl  -o lattice-lmrescore-kaldi-rnnlm-pruned
make[1]: Leaving directory '/home/sourcelink/work/test/kaldi-trunnk/src/latbin'
echo Done
Done
```


# 四. 验证

验证kaldi是否安装成功，可以选择运行 egs/ 目录下的许多测试用例脚本。  
笔者选择yesno孤立词识别进行测试;  

```
cd kaldi-trunnk/egs/yesno/s5
```

```
./run.sh
```

测试的结果如下:  

```
steps/train_mono.sh: Pass 37
steps/train_mono.sh: Pass 38
steps/train_mono.sh: Aligning data
steps/train_mono.sh: Pass 39
...
fstrmepslocal 
fstrmsymbols exp/mono0a/graph_tgpr/disambig_tid.int 
fstisstochastic exp/mono0a/graph_tgpr/HCLGa.fst 
0.5342 -0.000482149
HCLGa is not stochastic
add-self-loops --self-loop-scale=0.1 --reorder=true exp/mono0a/final.mdl exp/mono0a/graph_tgpr/HCLGa.fst 
steps/decode.sh --nj 1 --cmd utils/run.pl exp/mono0a/graph_tgpr data/test_yesno exp/mono0a/decode_test_yesno
decode.sh: feature type is delta
steps/diagnostic/analyze_lats.sh --cmd utils/run.pl exp/mono0a/graph_tgpr exp/mono0a/decode_test_yesno
steps/diagnostic/analyze_lats.sh: see stats in exp/mono0a/decode_test_yesno/log/analyze_alignments.log
Overall, lattice depth (10,50,90-percentile)=(1,1,2) and mean=1.2
steps/diagnostic/analyze_lats.sh: see stats in exp/mono0a/decode_test_yesno/log/analyze_lattice_depth_stats.log
local/score.sh --cmd utils/run.pl data/test_yesno exp/mono0a/graph_tgpr exp/mono0a/decode_test_yesno
local/score.sh: scoring with word insertion penalty=0.0,0.5,1.0
%WER 0.00 [ 0 / 232, 0 ins, 0 del, 0 sub ] exp/mono0a/decode_test_yesno/wer_10_0.0
```

最后输出了解码的误差率为0;  
