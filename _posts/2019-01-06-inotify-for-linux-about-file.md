---
layout: post
title:  "inotify监控linux文件系统事件"
date:   2019-01-06
catalog:  true
tags:
    - inotify
    - epoll
---

# 一. 概述

从 Linux 2.6.13 内核开始，Linux 就推出了 inotify，允许监控程序打开一个独立文件描述符，并针对事件集监控一个或者多个文件，例如打开、关闭、移动/重命名、删除、创建或者改变属性。 实例：android的输入系统中就使用inotify监控设备的变化。


# 二. API 介绍

## 2.1 inotify_init

```
extern int inotify_init (void) __THROW;
```

是用于创建一个 inotify 实例的系统调用，并返回一个指向该实例的文件描述符。


## 2.2 inotify_init1

```
extern int inotify_init1 (int __flags) __THROW;
```

与 inotify_init 相似，并带有附加标志。如果这些附加标志没有指定，将采用与 inotify_init 相同的值。

## 2.3 inotify_add_watch

```
extern int inotify_add_watch (int __fd, const char *__name, uint32_t __mask) __THROW;
```

增加对文件或者目录的监控，并指定需要监控哪些事件。  
返回值为已经添加进监控项目的描述符或者为异常值;

mask 可以是以下值的组合： 


>IN_ACCESS，文件被访问  
IN_ATTRIB，文件属性被修改  
IN_CLOSE_WRITE，可写文件被关闭  
IN_CLOSE_NOWRITE，不可写文件被关闭  
IN_CREATE，文件/文件夹被创建  
IN_DELETE，文件/文件夹被删除  
IN_DELETE_SELF，被监控的对象本身被删除  
IN_MODIFY，文件被修改  
IN_MOVE_SELF，被监控的对象本身被移动  
IN_MOVED_FROM，文件被移出被监控目录  
IN_MOVED_TO，文件被移入被监控目录  
IN_OPEN，文件被打开  


## 2.4 inotify_rm_watch

```
extern int inotify_rm_watch (int __fd, int __wd) __THROW;
```

根据wd，将加入的监控目录从监控项目中移除了；


# 三. 事件处理

事件的信息的获取是通过读取`inotify_init()`生成的文件描述符;  
比如这样：  

```
read_size = read(inotify_fd, event_buf, sizeof(event_buf));
```

读取的事件都在`event_buf`中，但是事件的结构体为`struct inotify_event`，如下：  

```
struct inotify_event
{
  int wd;		        /* Watch descriptor.  */
  uint32_t mask;	   /* Watch mask.  */
  uint32_t cookie;	 /* Cookie to synchronize two events.  */
  uint32_t len;		 /* Length (including NULs) of name.  */
  char name __flexarr;	/* Name.  */
};
```

可以根据`wd`来过滤是否为我们监控的目录发生的变化。



# 四. epoll + inotify的demo

```
#include <stdio.h>
#include <sys/epoll.h>
#include <unistd.h>
#include "sl_debug.h"
#include <sys/inotify.h>
#include <string.h>

DEBUG_SET_LEVEL(DEBUG_LEVEL_DEBUG);

#define  EPOLL_MAX_EVENTS  16

static char *path_name;
static int inotify_fd = -1;
static int path_watcher = -1;

static int add_to_epoll(int fd, int epoll_fd)
{
	int ret;
    struct epoll_event eventItem;
    memset(&eventItem, 0, sizeof(eventItem));
    eventItem.events = EPOLLIN;
    eventItem.data.fd = fd;
    ret = epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &eventItem);
	return ret;
}

static int poll_event(int _poll_fd, int _time_out)
{
    int event_count = 0;
    int i = 0;
    struct epoll_event event_items[EPOLL_MAX_EVENTS];

    event_count = epoll_wait(_poll_fd, event_items, EPOLL_MAX_EVENTS, _time_out);

    if (event_count == 0) {
        INFO("is wait time out");
    }

    for (i = 0; i < event_count; i++) {
        if (event_items[i].data.fd == inotify_fd) {
            int read_size = -1;
            char event_buf[512];
            int event_size = -1;
            struct inotify_event *event;
            int index = 0;
            read_size = read(inotify_fd, event_buf, sizeof(event_buf));

            if (read_size < sizeof(*event)) {
                ERR("read event"); 
                continue;
            }

            event = (struct inotify_event *)event_buf;

            while (read_size > index) {
                if (event->wd == path_watcher) {
                    INFO("%d: %08x \"%s\"\n", event->wd, event->mask, event->len ? event->name : "");
                }
                event++;
                index += sizeof(struct inotify_event);
            }
        } 
    }
}


int main(int argc, char **argv)
{
    int epoll_fd = -1;

    if (argc != 2) {
        ERR("usage: %s <pathname>", argv[0]);
        return -1;
    }

    path_name = argv[1];
    if (access(path_name, F_OK) != 0) {
        ERR("the path is not existence");
        return -1;
    }

    epoll_fd = epoll_create(5);
    if (epoll_fd < 0) {
        ERR("create epoll");
        return -1;
    }

    inotify_fd = inotify_init();
    if (inotify_fd < 0) {
        ERR("inotify init");
        return -1;
    }

    path_watcher = inotify_add_watch(inotify_fd, path_name, IN_CREATE | IN_DELETE);
    if (path_watcher < 0) {
       ERR("inotify add watch"); 
       return -1;
    }

    add_to_epoll(inotify_fd, epoll_fd);

    while (1) {
        poll_event(epoll_fd, -1);
    }


    return 0;
}
```

- 运行测试如下：

```
➜  inotify_epoll ./main tmp &
[1] 23641
➜  inotify_epoll echo pwa > tmp/1
INFO: 1: 00000100 "1"

➜  inotify_epoll echo pwa > tmp/2
INFO: 1: 00000100 "2"

➜  inotify_epoll rm tmp/1
INFO: 1: 00000200 "1"
```





