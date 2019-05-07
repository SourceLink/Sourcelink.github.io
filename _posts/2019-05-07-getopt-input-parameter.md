---
layout: post
title:  "解析命令行参数"
date:   2019-05-07 14:24:50
catalog:  true
author: Sourcelink
tags:
    - getopt_long
    - getopt

---

# 一. 概述

在程序中一般都会用到命令行选项, 我们可以使用 getopt 和 getopt_long 函数来解析命令行参数;  

因为前段时间在编写linux下音频播放程序时参考了**aplay**的源码, 其中看到了`getopt()`和`getopt_long()`, 所以在这记录下使用方法;  



# 二. getopt()


## 2.1 参数详解

getopt主要用来处理短命令行选项, 例如`aplay -h`, 其中`-h`是条短命令, 用于获取帮助;  在linux下使用man命令查看getopt获取到如下信息:  

```
#include <unistd.h>

int getopt(int argc, char * const argv[],
          const char *optstring);

extern char *optarg;
extern int optind, opterr, optopt;
```

可以看到使用getopt所需要包含的头文件, 函数体和一起全局变量;  

如下是参数 `optstring` 的示例值:  

```
static const char short_options[] = "hnlLD:qt:c:f:r:d:s:MNF:A:R:T:B:vV:IPCi";
```


## 2.2 Demo

其中符号`:`表示该短命令后面需要带参数, 而这参数的传递依靠全局变量`optarg`, 看个小例子:  

```
#include <stdio.h>
#include <unistd.h>
int main(int argc, char **argv) {
    int ch;
    while((ch = getopt(argc, argv, "a:b")) != -1) {
        switch(ch) {
            case 'a':
                printf("option a: %s\n", optarg);
                break;
            case 'b':
                printf("option b \n");
                break;
            case '?': // 输入未定义的选项, 都会将该选项的值变为 ?
                printf("unknown option \n");
                break;
            default:
                printf("default \n");
        }
    }
    if (optind >= argc) {
		     printf("Expected argument after options optind = %d argc = %d\n", optind, argc);
		     return -1;
	   }
	
    printf("name argument = %s\n", argv[optind]);
}
```

测试结果如下:  

```
./getopt_test -a 1 -b -c                                                           
option a: 1                                                                        
option b                                                                           
./getopt_test: invalid option -- 'c'                                               
unknown option                                                                     
Expected argument after options optind = 5 argc = 5
```

```
./getopt_test -a 1 -b 123                                                          
option a: 1                                                                        
option b                                                                           
name argument = 123
```

对比上面两个测试结果再结合代码可以看出全局变量`optind`是指符合格式的命令和参数的计数, 就算输入命令`-d`也是可以的,  
因为它虽然没在命令列表中, 但是它符合命令格式;  

可以再验证下:  

```
./getopt_test -d  -d                                                              
./getopt_test: invalid option -- 'd'                                              
unknown option                                                                    
./getopt_test: invalid option -- 'd'                                              
unknown option                                                                    
Expected argument after options optind = 3 argc = 3
```

接下来还有两个参数`opterr`和`optopt`:  

- opterr

该参数可以控制getopt的日志输入, 设置为1则会输出, 否则不会;

- optopt

当解析到的命令未在`optstring`列表中, 则会将其赋值给`optopt`; 比如`-d`, 则对应的`optopt`的值为100, 十进制的形式存放;  


# 三. getopt_long()

## 3.1 参数详解 

调用`getopt_long`需要包含的头文件和函数体如下:  

```
#include <getopt.h>

int getopt_long(int argc, char * const argv[],
          const char *optstring,
          const struct option *longopts, int *longindex);
```

该函数相对与`getopt`多了两个参数, `longopts`和`longindex`;  

其中`longopts`是个`struct option`指针, 具体如下:  

```
struct option
{
  const char *name;
  /* has_arg can't be an enum because some compilers complain about
     type mismatches in all the code that assumes it is an int.  */
  int has_arg;
  int *flag;
  int val;
};
```

结构体内变量的含义如下:  


> name:  该变量保存了长命令名;  
> has_arg: 命令后面是否带参;  
> flag:  指定长选项如何返回结果, 如果flag为NULL, getopt_long() 会返回val. 如果flag不为NULL, getopt_long会返回0, 并且将val的值存储到flag指向的地址中
> val: 长命令对应的值, 将要被getopt_long返回或者存储到flag指向的变量中的值;  


如下是`aplay`源码中对应的长命令部分数据:  

```
	static const struct option long_options[] = {
		{"help", 0, 0, 'h'},
		{"version", 0, 0, OPT_VERSION},
		{"list-devnames", 0, 0, 'n'},
		{"list-devices", 0, 0, 'l'},
		{"list-pcms", 0, 0, 'L'},
		{"device", 1, 0, 'D'},
		{"quiet", 0, 0, 'q'},
		{"file-type", 1, 0, 't'},
		{"channels", 1, 0, 'c'},
		{"format", 1, 0, 'f'},
		{"rate", 1, 0, 'r'},
		{"duration", 1, 0 ,'d'},
		{"samples", 1, 0, 's'},
		{"mmap", 0, 0, 'M'},
		{"nonblock", 0, 0, 'N'},
		{"period-time", 1, 0, 'F'},
		{"period-size", 1, 0, OPT_PERIOD_SIZE},
		{"avail-min", 1, 0, 'A'},
		{"start-delay", 1, 0, 'R'},
		{"stop-delay", 1, 0, 'T'},
		{"buffer-time", 1, 0, 'B'},
		{"buffer-size", 1, 0, OPT_BUFFER_SIZE},
		{"verbose", 0, 0, 'v'},
		{"vumeter", 1, 0, 'V'},
```


参数`longindex`则表示当前返回命令在参数列表中的偏移值; 

## 3.2 Demo

```
#include <stdio.h>     /* for printf */
#include <stdlib.h>    /* for exit */
#include <getopt.h>

int main(int argc, char **argv)
{
    int c;
    int digit_optind = 0;

    while (1) {
        int this_option_optind = optind ? optind : 1;
        int option_index = 0;
        static struct option long_options[] = {
            {"add",     required_argument, 0,  'a' },
            {"append",  no_argument,       0,  0 },
            {"delete",  required_argument, 0,  'd' },
            {"verbose", no_argument,       0,  'v' },
            {"create",  required_argument, 0, 'c'},
            {"file",    required_argument, 0,  0 },
            {0,         0,                 0,  0 }
        };

        c = getopt_long(argc, argv, "a:bc:d:012", long_options, &option_index);
        if (c == -1)
            break;

        switch (c) {
            case 0:
                printf("option %s", long_options[option_index].name);
                if (optarg)
                printf(" with arg %s", optarg);
                printf("\n");
                break;

            case '0':
            case '1':
            case '2':
                if (digit_optind != 0 && digit_optind != this_option_optind)
                printf("digits occur in two different argv-elements.\n");
                digit_optind = this_option_optind;
                printf("option %c\n", c);
                break;

            case 'a':
                printf("option a or add arg = %s\n", optarg);
                break;

            case 'b':
                printf("option b arg = %s\n", optarg);

            case 'd':
                printf("option d or delete with value '%s'\n", optarg);
                break;

            case 'v':
                printf("option v or verbose\n");
                break;
                
            case '?':
                break;

            default:
                printf("?? getopt returned character code 0%o ??\n", c);
                break;
        }
    }

    return 0;
}
```

测试结果如下:  


```
./getopt_long_test --add 12 -a 2 --verbose --append                                         
option a or add arg = 12                                                           
option a or add arg = 2                                                            
option v or verbose
option append
```


# 四. getopt_long_only()

将上节demo中的`getopt_long()` 替换为 `getopt_long_only()`, 再重新测试, 结果如下:  


```
./getopt_long_test -add 12 -a 2 -verbose -append                                         
option a or add arg = 12                                                           
option a or add arg = 2                                                            
option v or verbose
option append
```

可以发现结果和上节是一样的, 但是长命令的标示符`--`已经变成了`-`;  
即使用 **getopt_long_only()** 时,  `-` 和 `--` 都可以作用于长选项, 而使用 **getopt_long()** 时, 只有 --可以作用于长选项;  



**PS**: 上述函数均为线程安全函数;  












