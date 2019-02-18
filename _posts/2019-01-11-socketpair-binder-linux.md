---
layout: post
title:  "socketpair+binder实现进程间通信"
date:   2019-01-11
catalog:  true
tags:
    - socketpair
    - binder
---

# 一. 概述

在android的binder系统中一直是`client`端单向发起请求,而`service`端不能主动向`client`端发送数据.  

## 1.1 原理  
引入一个新方法, `socketpair()`:  该系统调用能创建一对已连接的无名socket. 在Linux中, 完全可以把这一对  
socket当成pipe返回的文件描述符一样使用, 唯一的区别就是这一对文件描述符中的任何一个都可读和可写.  
这样把其中一个文件描述送入目标进程中就可以往里面写东西,但是如何让对方知道该文件描述符呢?  
这里就需要使用binder进行传送了,利用binder驱动为目标进程创建一个新的文件描述符指向该socket;  

## 1.2 环境

运行系统: arm linux  
运行平台: imx6ul  

# 二. socket API 介绍

## 2.1 socketpair

```
extern int socketpair (int __domain, int __type, int __protocol, int __fds[2]) __THROW;
```

该系统调用会生成两个已链接的无名socket, 存放在`__fds[2]`中,返回0表示创建成功. 


## 2.2 setsockopt

```
extern int setsockopt (int __fd, int __level, int __optname, const void *__optval, socklen_t __optlen) __THROW;
```

根据`__optname`设置创建的`socket`的属性,返回0表示设置成功.


# 三. binder新增接口介绍

在前面`binder机制情景分析之c服务应用`的基础上增加的一下接口


## 3.1 boi_put_fd

```
void boi_put_fd(struct binder_io *bio, uint32_t fd)
{
    struct flat_binder_object *obj;

    if (fd)
        obj = bio_alloc_obj(bio);
    else
        obj = bio_alloc(bio, sizeof(*obj));

    if (!obj)
        return;

    obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    obj->type = BINDER_TYPE_FD;
    obj->handle = fd;
    obj->cookie = 0;
}
```

`type`变为了`BINDER_TYPE_FD`, 文件描述符通过`handle`进行传送;  

## 3.2 bio_get_fd

```
uint32_t bio_get_fd(struct binder_io *bio)
{
    struct flat_binder_object *obj;

    obj = _bio_get_obj(bio);
    if (!obj)
        return 0;

    if (obj->type == BINDER_TYPE_FD)
        return obj->handle;

    return 0;
}
```

当object类型为`BINDER_TYPE_FD`返回结果;  

## 3.3 binder_call


```
int binder_call(struct binder_state *bs,
                struct binder_io *msg, struct binder_io *reply,
                uint32_t target, uint32_t code)
{
....
    writebuf.cmd = BC_TRANSACTION;   // binder call transaction 
    writebuf.txn.target.handle = target;
    writebuf.txn.code = code;
    writebuf.txn.flags |= TF_ACCEPT_FDS;
    writebuf.txn.data_size = msg->data - msg->data0;
    writebuf.txn.offsets_size = ((char*) msg->offs) - ((char*) msg->offs0);
    writebuf.txn.data.ptr.buffer = (uintptr_t)msg->data0;
    writebuf.txn.data.ptr.offsets = (uintptr_t)msg->offs0;

....
}
```

这里修改`flags`为`TF_ACCEPT_FDS`, 因为驱动中有检测目标进程是否需要获取文件描述符回复;


# 四. 流程

## 4.1 service端

- 1. 创建无名socket

```
/* 创建通信频道 */
    if (socketpair(AF_UNIX, SOCK_STREAM, 0, socket_fd) < 0)
    {
        ALOGE("failed to creat socketpair\n");
    } else {
        g_fd = socket_fd[1];
        pthread_create(&thread, NULL, socetpair_fun, &socket_fd[0]);
    }
```

创建socket成功则创建一个线程,等待客户端发送消息;


- 2. socket线程处理函数

```
static void *socetpair_fun(void *arg)
{
    char buf[500];
    int len;
    int cnt = 0;
    int fd = *(int*)(arg);

    while(1)
    {
        /* 读数据: test_client发出的数据 */
        len = read(fd, buf, 500);
        buf[len] = '\0';
        ALOGI("%s\n", buf);
        
        /* 向 test_client 发出: Hello, test_client */
        len = sprintf(buf, "Hello, test_client, cnt = %d", cnt++);
        write(fd, buf, len);
    }

    return NULL;
}
```


- 3. 文件描述符回复

```
static int get_fd(void *arg)
{
    struct io_manager *manager = (struct io_manager*)arg;

    if (g_fd) {
        bio_put_uint32(manager->tx, 0);     /* 发送头帧 */
        boi_put_fd(manager->tx, g_fd);     
    }

    return 0;
}
```

当`client`调用该方法时,回复文件描述符给客户端;  


## 4.2  client端

- 1. 获取文件描述符

```
int interface_get_fd(struct binder_state *bs, unsigned int handle)
{
    unsigned iodata[512/4];
    struct binder_io msg, reply;
    int ret = -1;
	int exception;

    bio_init(&msg, iodata, sizeof(iodata), 4);
    bio_put_uint32(&msg, 0);  // strict mode header
    

    if (binder_call(bs, &msg, &reply, handle, LED_CONTROL_FD))
        return ret;

    exception = bio_get_uint32(&reply);  /* 获取头帧 */
    ALOGE("exception = %d\n", exception);  
	if (exception == 0)
		ret = bio_get_fd(&reply);

    binder_done(bs, &msg, &reply);

    return ret;
}
```

调用`service`端的`LED_CONTROL_FD`方法获取文件描述,返回值为获取到的文件描述符;  

- 2. 利用socket描述符进行通信

```
ALOGI("client call get_fd = %d", fd);

char buf[500];
int len;
int cnt = 0;

while (1)
{
    len = sprintf(buf, "Hello, test_server, cnt = %d", cnt++);
    write(fd, buf, len);

    len = read(fd, buf, 500);
    buf[len] = '\0';
    ALOGI("%s len = %d\n", buf, len);

    sleep(5);
}
```

每5s往socket写入数据, 然后service端收到就会回复数据;  


这是个简单的范例,实用范例还应该检测文件描述符状态;  


# 五. 驱动分析

当`service`端调用`boi_put_fd`回复文件描述符时对应的驱动端`binder_transaction`代码:

```
case BINDER_TYPE_FD: {
			int target_fd;
			struct file *file;

			if (reply) {
				if (!(in_reply_to->flags & TF_ACCEPT_FDS)) {                                            ①
					binder_user_error("%d:%d got reply with fd, %d, but target does not allow fds\n",
						proc->pid, thread->pid, fp->handle);
					return_error = BR_FAILED_REPLY;
					goto err_fd_not_allowed;
				}
			} else if (!target_node->accept_fds) {
				binder_user_error("%d:%d got transaction with fd, %d, but target does not allow fds\n",
					proc->pid, thread->pid, fp->handle);
				return_error = BR_FAILED_REPLY;
				goto err_fd_not_allowed;
			}

			file = fget(fp->handle);                                                                 ②
			if (file == NULL) {
				binder_user_error("%d:%d got transaction with invalid fd, %d\n",
					proc->pid, thread->pid, fp->handle);
				return_error = BR_FAILED_REPLY;
				goto err_fget_failed;
			}
			if (security_binder_transfer_file(proc->tsk, target_proc->tsk, file) < 0) {
				fput(file);
				return_error = BR_FAILED_REPLY;
				goto err_get_unused_fd_failed;
			}
			target_fd = task_get_unused_fd_flags(target_proc, O_CLOEXEC);                            ③
			if (target_fd < 0) {
				fput(file);
				return_error = BR_FAILED_REPLY;
				goto err_get_unused_fd_failed;
			}
			task_fd_install(target_proc, target_fd, file);                                           ④
			trace_binder_transaction_fd(t, fp->handle, target_fd);
			binder_debug(BINDER_DEBUG_TRANSACTION,
				     "        fd %d -> %d\n", fp->handle, target_fd);
			/* TODO: fput? */
			fp->handle = target_fd;
		} break;
```

> ①: 检测下目标进程是否允许获取文件文件描述符  
> ②: 根据文件描述符获取到该文件的结构指针  
> ③: 从目标进程中获取一个空闲的文件描述符  
> ④: 将空闲的文件描述符绑定到文件上.  







