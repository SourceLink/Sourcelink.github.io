---
layout: post
title:  "如何得到指针指向的数组的长度"
date:   2019-08-23
catalog:  true
author: Sourcelink
tags:
    - 编程技巧
    - c
    - 指针

---



# 一. 概述


最近在编写一个一个16进制的通信协议封包代码时, 遇到了一个**问题**: 如何获取指针指向的数组大小?  
最后得出一个答案无法解决, 心痛的感觉; 但是不解决这个问题, 代码冗余将很大; 不过好在后面用一个小技巧解决;


# 二. 问题


因为是16进制通信所以`strlen()`计算必然是无法使用的, 只能使用`sizeof()`进行数组长度计算;  

```
char xxx_tab[] = {0xB8, 0xBA, 0xD1, 0xF5, 0xC0, 0xEB, 0xD7, 0xD3, 0x3A};
```


已上述数组为例计算数组长度代码如下: 

```
int len = sizeof(xxx_tab) / sizeof(char);
```

上述计算是没有问题的, 但是现在因为有很多类似的tab, 这些tab是通信封包中的一些特定头, 所以希望在函数调用的时候能通过枚举的形式来获取不同tab的长度; 
**即想通过指针获取tab长度**; 假设现在有个指针`ptemp`指向了`xxx_tab`, 代码如下:

```
unsigned char get_tab_len(int _xxtab)
{
   ptemp = get_tab_addr(_xxtab);
   
   len = sizeof(ptemp) / sizeof(char);
    
   return len;
}
```

上述是个大概的代码, 但它并不能得到我们想要的结果, 因为`sizeof()`计算出来的结果只是`ptemp`指针的大小4;  


# 三. 小技巧解决

小技巧仅仅只是小技巧, 最后并没有通过指针获取到tab的长度, 但是解决了我们的需求; 


```
#define TAB_SIZE(a)  sizeof(a) / sizeof(char)

char xxx_tab[] = {0xB8, 0xBA, 0xD1, 0xF5, 0xC0, 0xEB, 0xD7, 0xD3, 0x3A};
char xx_tab[] = {0xF5, 0xC0, 0xEB, 0xD7, 0xD3, 0x3A};

unsigned char get_tab_len(int _xxtab)
{
   unsigned char len = 0;
   switch (_xxtab) {
       case xx: len = TAB_SIZE(xx_tab); break;
       case xxx: len = TAB_SIZE(xx_tab); break;
       
   }
   
   return len;
}
```

这就是我指的小技巧, 在获取指针时, 进行数组长度计算, 因为这时候计算时, 还是用的tab地址本身; 用#define改造下:

```
#define TAB_SIZE(a)  sizeof(a) / sizeof(char)
#define DATA_HEAD_LEN(n) case n: len = TAB_SIZE(n##_tab); break

char xxx_tab[] = {0xB8, 0xBA, 0xD1, 0xF5, 0xC0, 0xEB, 0xD7, 0xD3, 0x3A};
char xx_tab[] = {0xF5, 0xC0, 0xEB, 0xD7, 0xD3, 0x3A};

unsigned char get_tab_len(int _xxtab)
{
   unsigned char len = 0;
   switch (_xxtab) {
      DATA_HEAD_LEN(xx);
      DATA_HEAD_LEN(xxx);
       
   }
   return len;
}
```


如果不用函数调用的话, 而是通过#define调用可以修改成如下:

```
#define DATA_HEAD_LEN(n) TAB_SIZE(n##_tab)
```