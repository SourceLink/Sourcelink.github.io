---
layout: post
title:  "线程私有数据TSD"
date:   2018-12-15
catalog:  true
tags:
    - thread
    - TSD
---


# 一. 概述

平时写程序时,各个函数内部想共用一个变量,此时会创建一个全局变量来满足这样的操作.  
编写单线程程序,使用全局变量到还不会产生很大影响,但是多线程情况下就容易出现竞争关系;  
在多线程环境下,由于数据空间是共享的,因此全局变量也为所有线程所共有,当各个线程中有函数操作同一个变量就容易出现时序上的错误;  

此时有人可能会说,在创建线程的function中定义一个变量不就可以了,这样整个线程也可以访问该变量(局部变量,栈上申请)而且是私有的,其他线程无法访问到的;   

这个问题以前我也掉进去过,但是后面发现有问题,这样做你必须得把你所有的工作都在该线程函数中完成,不能嵌套;因为你在其它函数内部就无法访问了,除非你愿意为每个函数构造一个形参一层一层的传递;  

在多线程编程中Posix为我们提供一个叫线程私有数据(Thread-specific Data,简称TSD)的家伙来解决这个问题;

# 二. API介绍

## 2.1 创建和注销

***创建:***

```
int pthread_key_create(pthread_key_t *key, void (*destr_function) (void *))
```

该函数从TSD池中分配一项,将其值赋给key供以后访问使用. 如果destr_function不为空,在线程退出时将以key所关联的数据为参数调用destr_function(),以释放分配的缓冲区, 如果此时key中无数据则不会调用该function, 该function的形参为设置进key 的值。

***注销:***

```
int pthread_key_delete(pthread_key_t key)
```

这个函数并不检查当前是否有线程正使用该TSD，也不会调用清理函数（destr_function），而只是将TSD释放以供下一次调用pthread_key_create()使用;   


## 2.2 设置和获取

***设置:***

```
int  pthread_setspecific(pthread_key_t  key, const void *pointer)
```

形参`key`是用于必须先创建且一个应用程序中允许有多个key,`pointer`要设置进去的值;

***获取:***

```
void * pthread_getspecific(pthread_key_t key)
```

多线程中,各线程可以使用同一个`key`进行值的设置和获取,不会被覆盖,各个线程是独立的,虽然使用的是同一个key进行设置;  



# 三. Demo

```
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <string.h>

pthread_key_t g_key;

static pthread_mutex_t g_tsd_mutex = PTHREAD_MUTEX_INITIALIZER;
static int g_have_tsd = 0;


void destructor_fun(void *arg)
{

    printf("%s will delete\n", (char *)arg);
}


int create_self(void *arg)
{
restart: 
    if (g_have_tsd) {
        char * ret = (char *)pthread_getspecific(g_key);
        if (ret) return 0;
        if (pthread_setspecific(g_key, arg) == 0) {
            return 0;
        }
        return -1;
    }

    pthread_mutex_lock(&g_tsd_mutex);
    if (!g_have_tsd) {
        if(pthread_key_create(&g_key, destructor_fun) != 0) {
            pthread_mutex_unlock(&g_tsd_mutex);
            return -1;
        }
        g_have_tsd = 1;
    }
    pthread_mutex_unlock(&g_tsd_mutex);

    goto restart;
}


void *thread_fun(void *arg)
{
    if (create_self(arg) == 0) {
        printf("this is %s\n", (char*)pthread_getspecific(g_key));
        sleep(2);
    }
    // pthread_key_delete(g_key);
}


int main()
{
    pthread_t thread1;
    pthread_t thread2;

    pthread_create(&thread1, NULL, thread_fun, "thread 1");

    pthread_create(&thread2, NULL, thread_fun, "thread 2");

    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);

    return 0;
}
```

运行结果如下: 

```
➜  linux ./tsd
this is thread 1
this is thread 2
thread 1 will delete
thread 2 will delete
```

该demo没有实际的作用,其创建了两个线程且使用同一个程序工作`thread_fun`;  
在`thread_fun`程序中调用了`create_self`进行key的创建和value的设置, 设置成功返回0,否则返回-1;`thread_fun`中屏蔽了`pthread_key_delete`,读者可以打开了看下两个运行结果的区别,再回头看下2.1小节;











