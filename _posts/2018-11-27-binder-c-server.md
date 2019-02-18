---
layout: post
title:  "Binder机制情景分析之C服务应用"
date:   2018-11-27
catalog:  true
tags:
    - android
    - binder
    - Linux

---

# 一. 概述

这里只讲下binder的实现原理，不牵扯到android的java层是如何调用;  
涉及到的会有`ServiceManager`,`led_control_server`和`test_client`的代码,这些都是用c写的.其中`led_control_server`和`test_client`是  
仿照`bctest.c`写的; 在linux平台下运行binder更容易分析binder机制实现的原理(可以增加大量的log,进行分析);
在Linux运行时.先运行`ServiceManager`,再运行`led_control_server`最后运行`test_client`;  

# 1.1 Binder通信模型

Binder通信采用C/S架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中`ServiceManager`用于管理系统中的各种服务。

# 1.2 运行环境

本文中的代码运行环境是在imx6ul上跑的，运行的是linux系统，内核版本4.10(非android环境分析);

# 1.3 文章代码

文章所有代码已上传  
> https://github.com/SourceLink/linux_binder

# 二. ServiceManager

涉及到的源码地址:  

> frameworks/native/cmds/servicemanager/sevice_manager.c  
> frameworks/native/cmds/servicemanager/binder.c  
> frameworks/native/cmds/servicemanager/bctest.c  

`ServiceManager`相当于binder通信过程中的守护进程，本身也是个binder服务、好比一个root管理员一样;  
主要功能是查询和注册服务;接下来结合代码从main开始分析下serviceManager的服务过程;  


# 2.1 main

源码中的`sevice_manager.c`中主函数中使用了`selinux`，为了在我板子的linux环境中运行，把这些代码屏蔽，删减后如下：



``` 
int main(int argc, char **argv)
{
    struct binder_state *bs;

    bs = binder_open(128*1024);                                                ①
    if (!bs) {
        ALOGE("failed to open binder driver\n");
        return -1;
    }

    if (binder_become_context_manager(bs)) {                                   ②
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }


    svcmgr_handle = BINDER_SERVICE_MANAGER;
    binder_loop(bs, svcmgr_handler);                                           ③

    return 0;
}
```


> ①: 打开binder驱动(详见2.2.1)  
> ②: 注册为管理员(详见2.2.2)  
> ③: 进入循环，处理消息(详见2.2.3)  

从主函数的启动流程就能看出`sevice_manager`的工作流程并不是特别复杂;  
其实`client`和`server`的启动流程和`manager`的启动类似，后面再详细分析;  

# 2.2 binder_open

```
struct binder_state *binder_open(size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }

    bs->fd = open("/dev/binder", O_RDWR);                                     ①
    if (bs->fd < 0) {
        fprintf(stderr,"binder: cannot open device (%s)\n",
                strerror(errno));
        goto fail_open;
    }

    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||                       ②
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        fprintf(stderr, "binder: driver version differs from user space\n");
        goto fail_open;
    }

    bs->mapsize = mapsize;
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);      ③
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
```

> ①: 打开binder设备  
> ②: 通过ioctl获取binder版本号  
> ③: mmp内存映射  

这里说明下为什么binder驱动是用ioctl来操作，是因为ioctl可以同时进行读和写操作；


# 2.2 binder_become_context_manager

```
int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```
还是通过`ioctl`请求类型`BINDER_SET_CONTEXT_MGR`注册成manager;  



# 2.3 binder_loop



```
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));                                       ①

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);                                  ②

        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }

        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);       ③
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}
```

> ①: 写入命令`BC_ENTER_LOOPER`通知驱动该线程已经进入主循环，可以接收数据;  
> ②: 先读一次数据，因为刚才写过一次;  
> ③: 然后解析读出来的数据(详见2.2.4);  


binder_loop函数的主要流程如下:  
![](/images/binder/binder_loop.png)

# 2.4 binder_parse

```
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);
#if TRACE
        fprintf(stderr,"%s:\n", cmd_name(cmd));
#endif
        switch(cmd) {
        case BR_NOOP:
            break;
        case BR_TRANSACTION_COMPLETE:
            /* check服务 */
            break;
        case BR_INCREFS:
        case BR_ACQUIRE:
        case BR_RELEASE:
        case BR_DECREFS:
#if TRACE
            fprintf(stderr,"  %p, %p\n", (void *)ptr, (void *)(ptr + sizeof(void *)));
#endif
            ptr += sizeof(struct binder_ptr_cookie);
            break;
		case BR_SPAWN_LOOPER: {
			/* create new thread */
			//if (fork() == 0) {
			//}
			pthread_t thread;
			struct binder_thread_desc btd;

			btd.bs = bs;
			btd.func = func;
			
			pthread_create(&thread, NULL, binder_thread_routine, &btd);

			/* in new thread: ioctl(BC_ENTER_LOOPER), enter binder_looper */

			break;
		}
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: txn too small!\n");
                return -1;
            }
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);                               ①
                bio_init_from_txn(&msg, txn);
                res = func(bs, txn, &msg, &reply);                                       ②
                binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);                ③
            }
            ptr += sizeof(*txn);
            break;
        }
        case BR_REPLY: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: reply too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            if (bio) {
                bio_init_from_txn(bio, txn);
                bio = 0;
            } else {
                /* todo FREE BUFFER */
            }
            ptr += sizeof(*txn);
            r = 0;
            break;
        }
        case BR_DEAD_BINDER: {
            struct binder_death *death = (struct binder_death *)(uintptr_t) *(binder_uintptr_t *)ptr;
            ptr += sizeof(binder_uintptr_t);
            death->func(bs, death->ptr);
            break;
        }
        case BR_FAILED_REPLY:
            r = -1;
            break;
        case BR_DEAD_REPLY:
            r = -1;
            break;
        default:
            ALOGE("parse: OOPS %d\n", cmd);
            return -1;
        }
    }

    return r;
}
```

> ①: 按照一定的格式初始化rdata数据,请注意这里rdata是在用户空间创建的buf;  
> ②: 调用设置进来的处理函数`svcmgr_handler`(详见2.2.5);  
> ③: 发送回复信息;

这个函数我们只重点关注下`BR_TRANSACTION`其他的命令含义可以参考表格A;
# 2.5 svcmgr_handler

```
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;

    //ALOGI("target=%x code=%d pid=%d uid=%d\n",
    //  txn->target.handle, txn->code, txn->sender_pid, txn->sender_euid);

    if (txn->target.handle != svcmgr_handle)
        return -1;

    if (txn->code == PING_TRANSACTION)
        return 0;

    // Equivalent to Parcel::enforceInterface(), reading the RPC
    // header with the strict mode policy mask and the interface name.
    // Note that we ignore the strict_policy and don't propagate it
    // further (since we do no outbound RPCs anyway).
    strict_policy = bio_get_uint32(msg);                                           ①
    s = bio_get_string16(msg, &len);
    if (s == NULL) {
        return -1;
    }

    if ((len != (sizeof(svcmgr_id) / 2)) ||                                        ②
        memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
        fprintf(stderr,"invalid id %s\n", str8(s, len));
        return -1;
    }


    switch(txn->code) {                                                            ③
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = do_find_service(bs, s, len, txn->sender_euid, txn->sender_pid);   ④
        if (!handle)
            break;
        bio_put_ref(reply, handle);
        return 0;

    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        if (do_add_service(bs, s, len, handle, txn->sender_euid,                   ⑤
            allow_isolated, txn->sender_pid))
            return -1;
        break;

    case SVC_MGR_LIST_SERVICES: {
        uint32_t n = bio_get_uint32(msg);

        if (!svc_can_list(txn->sender_pid)) {
            ALOGE("list_service() uid=%d - PERMISSION DENIED\n",
                    txn->sender_euid);
            return -1;
        }
        si = svclist;
        while ((n-- > 0) && si)                                                    ⑥
            si = si->next;
        if (si) {
            bio_put_string16(reply, si->name);
            return 0;
        }
        return -1;
    }
    default:
        ALOGE("unknown code %d\n", txn->code);
        return -1;
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```

> ①: 获取帧头数据,一般为0，因为发送方发送数据时都会在数据最前方填充4个字节0数据(分配数据空间的最小单位4字节);  
> ②: 对比`svcmgr_id`是否和我们原来定义相同`#define SVC_MGR_NAME "linux.os.ServiceManager"`(我改写了);  
> ③: 根据`code` 做对应的事情,就想到与根据编码去执行对应的fun(client请求服务后去执行服务，service也是根据不同的code来执行。接下来会举例说明);、
> ④: 从服务名在server链表中查找对应的服务，并返回handle(详见2.6);  
> ⑤: 添加服务，一般都是service发起的请求。将handle和服务名添加到服务链表中(这里的handle是由binder驱动分配);  
> ⑥: 查找server_manager中链表中第`n`个服务的名字(该数值由查询端决定);  

# 2.6  do_find_service

```
uint32_t do_find_service(struct binder_state *bs, const uint16_t *s, size_t len, uid_t uid, pid_t spid)
{
    struct svcinfo *si;

    if (!svc_can_find(s, len, spid)) {                                             ①
        ALOGE("find_service('%s') uid=%d - PERMISSION DENIED\n",
             str8(s, len), uid);
        return 0;
    }
    si = find_svc(s, len);                                                         ②
    //ALOGI("check_service('%s') handle = %x\n", str8(s, len), si ? si->handle : 0);
    if (si && si->handle) {
        if (!si->allow_isolated) {                                                 ③
            // If this service doesn't allow access from isolated processes,
            // then check the uid to see if it is isolated.
            uid_t appid = uid % AID_USER;
            if (appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END) {
                return 0;
            }
        }
        return si->handle;                                                         ④
    } else {
        return 0;
    }
}
``` 

> ①: 检测调用进程是否有权限请求服务(这里用selinux管理权限,为了让代码可以方便允许,这里面的代码有做删减);  
> ②: 遍历server_manager服务链表;  
> ③: 如果binder服务不允许服务从沙箱中访问,则执行下面检查;  
> ④: 返回查询到handle;  

`do_find_service`函数主要工作是搜索服务链表,返回查找到的服务


# 2.7  do_add_service

```
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid)
{
    struct svcinfo *si;

    //ALOGI("add_service('%s',%x,%s) uid=%d\n", str8(s, len), handle,
    //        allow_isolated ? "allow_isolated" : "!allow_isolated", uid);

    if (!handle || (len == 0) || (len > 127))
        return -1;

    if (!svc_can_register(s, len, spid)) {                                     ①
        ALOGE("add_service('%s',%x) uid=%d - PERMISSION DENIED\n",
             str8(s, len), handle, uid);
        return -1;
    }

    si = find_svc(s, len);                                                     ②
    if (si) {
        if (si->handle) {
            ALOGE("add_service('%s',%x) uid=%d - ALREADY REGISTERED, OVERRIDE\n",
                 str8(s, len), handle, uid);
            svcinfo_death(bs, si);
        }
        si->handle = handle;
    } else {                                                                   ③
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) {
            ALOGE("add_service('%s',%x) uid=%d - OUT OF MEMORY\n",
                 str8(s, len), handle, uid);
            return -1;
        }
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->next = svclist;
        svclist = si;
    }

    ALOGI("add_service('%s'), handle = %d\n", str8(s, len), handle);

    binder_acquire(bs, handle);                                               ④
    binder_link_to_death(bs, handle, &si->death);                             ⑤
    return 0;
}
```

> ①: 判断请求进程是否有权限注册服务;  
> ②: 查找ServiceManager的服务链表中是否已经注册了该服务,如果有则通知驱动杀死原先的binder服务,然后更新最新的binder服务;  
> ③: 如果原来没有创建该binder服务,则进行一系列的赋值,再插入到服务链表的表头;  
> ④: 增加binder服务的引用计数;  
> ⑤: 告诉驱动接收服务的死亡通知;  


# 2.8 调用时序图
从上面分析,可以知道`ServiceManager`的主要工作流程如下:  
![](/images/binder/serviermanager.png)

# 三. led_control_server

# 3.1 main

```
int main(int argc, char **argv) 
{
    int fd;
    struct binder_state *bs;
    uint32_t svcmgr = BINDER_SERVICE_MANAGER;
    uint32_t handle;
	  int ret;

    struct register_server  led_control[3] = {                                ①
        [0] = {
            .code = 1,
            .fun = led_on
        } , 
        [1] = {
            .code = 2,
            .fun = led_off
        }
    };

    
    bs = binder_open(128*1024);                                               ②
    if (!bs) {
        ALOGE("failed to open binder driver\n");
        return -1;
    }

    
    ret = svcmgr_publish(bs, svcmgr, LED_CONTROL_SERVER_NAME, led_control);   ③

    if (ret) {
        ALOGE("failed to publish %s service\n", LED_CONTROL_SERVER_NAME);
        return -1;
    }

    binder_set_maxthreads(bs, 10);                                            ④

    binder_loop(bs, led_control_server_handler);                              ⑤

    return 0;
}
```

> ①: `led_control_server`提供的服务函数;  
> ②:   初始化binder组件( 详见2.2);  
> ③:   注册服务,`svcmgr`是发送的目标, `LED_CONTROL_SERVER_NAME`注册的服务名, `led_control`注册的binder实体;  
> ④:  设置创建线程最大数(详见3.5);  
> ⑤:  进入线程循环(详见2.3);  

# 3.2 svcmgr_publish

```
int svcmgr_publish(struct binder_state *bs, uint32_t target, const char *name, void *ptr)
{
    int status;
    unsigned iodata[512/4];
    struct binder_io msg, reply;

    bio_init(&msg, iodata, sizeof(iodata), 4);                                ①
    bio_put_uint32(&msg, 0);  // strict mode header
    bio_put_string16_x(&msg, SVC_MGR_NAME);
    bio_put_string16_x(&msg, name);
    bio_put_obj(&msg, ptr);

    if (binder_call(bs, &msg, &reply, target, SVC_MGR_ADD_SERVICE))           ②
        return -1;

    status = bio_get_uint32(&reply);                                          ③

    binder_done(bs, &msg, &reply);                                            ④

    return status;
}
``` 

> ①: 初始化用户空间的数据iodata,设置了四个字节的offs,接着按一定格式往buf里面填充数据;  
> ②: 调用`ServiceManager`服务的`SVC_MGR_ADD_SERVICE`功能;  
> ③: 获取`ServiceManager`回复数据,成功返回`0`;  
> ④: 结束注册过程,释放内核中刚才交互分配的buf;  


## 3.2.1 bio_init

```
void bio_init(struct binder_io *bio, void *data,
              size_t maxdata, size_t maxoffs)
{
    size_t n = maxoffs * sizeof(size_t);

    if (n > maxdata) {
        bio->flags = BIO_F_OVERFLOW;
        bio->data_avail = 0;
        bio->offs_avail = 0;
        return;
    }

    bio->data = bio->data0 = (char *) data + n;                               ①
    bio->offs = bio->offs0 = data;                                            ②
    bio->data_avail = maxdata - n;                                            ③
    bio->offs_avail = maxoffs;                                                ④
    bio->flags = 0;                                                           ⑤
}
``` 

> ①: 根据传进来的参数,留下一定长度的offs数据空间, data指针则从` data + n`开始;  
> ②: offs指针则从` data`开始,则offs可使用的数据空间只有`n`个字节;  
> ③: 可使用的data空间计数;  
> ④: 可使用的offs空间计数;  
> ⑤: 清除buf的flag;  

init后此时buf空间的分配情况如下图:

![](/images/binder/bio_init.png)


## 3.2.2 bio_put_uint32

```
void bio_put_uint32(struct binder_io *bio, uint32_t n)
{
    uint32_t *ptr = bio_alloc(bio, sizeof(n));
    if (ptr)
        *ptr = n;
}
``` 

这个函数往buf里面填充一个uint32的数据,这个数据的最小单位为4个字节;  
前面`svcmgr_publish`调用bio_put_uint32(&msg, 0);,实质buf中的数据是`00 00 00 00` ;  

## 3.2.3 bio_alloc

```
static void *bio_alloc(struct binder_io *bio, size_t size)
{
    size = (size + 3) & (~3);
    if (size > bio->data_avail) {
        bio->flags |= BIO_F_OVERFLOW;
        return NULL;
    } else {
        void *ptr = bio->data;
        bio->data += size;
        bio->data_avail -= size;
        return ptr;
    }
}
```

这个函数分配的数据宽度为4的倍数,先判断当前可使用的数据宽度是否小于待分配的宽度;  
如果小于则置标志`BIO_F_OVERFLOW`否则分配数据,并对`data`往后偏移`size`个字节,可使用数据宽度`data_avail`减去`size`个字节;  

## 3.2.4 bio_put_string16_x

```
void bio_put_string16_x(struct binder_io *bio, const char *_str)
{
    unsigned char *str = (unsigned char*) _str;
    size_t len;
    uint16_t *ptr;

    if (!str) {                                                            ①
        bio_put_uint32(bio, 0xffffffff);
        return;
    }

    len = strlen(_str);

    if (len >= (MAX_BIO_SIZE / sizeof(uint16_t))) {
        bio_put_uint32(bio, 0xffffffff);
        return;
    }

    /* Note: The payload will carry 32bit size instead of size_t */
    bio_put_uint32(bio, len);                                             ②
    ptr = bio_alloc(bio, (len + 1) * sizeof(uint16_t));
    if (!ptr)
        return;

    while (*str)                                                          ③
        *ptr++ = *str++;
    *ptr++ = 0;
}
``` 

> ①: 这里到`bio_alloc`前都是为了计算和判断自己串的长度再填充到buf中;  
> ②: 填充字符串前会填充字符串的长度;  
> ③: 填充字符串到buf中,一个字符占两个字节,注意` uint16_t *ptr;`;  


## 3.2.5 bio_put_obj

```
void bio_put_obj(struct binder_io *bio, void *ptr)
{
    struct flat_binder_object *obj;

    obj = bio_alloc_obj(bio);                                             ①
    if (!obj)
        return;

    obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    obj->type = BINDER_TYPE_BINDER;                                       ②
    obj->binder = (uintptr_t)ptr;                                         ③
    obj->cookie = 0;
}
``` 

```
struct flat_binder_object {
/* WARNING: DO NOT EDIT, AUTO-GENERATED CODE - SEE TOP FOR INSTRUCTIONS */
 __u32 type;
 __u32 flags;
 union {
 binder_uintptr_t binder;
/* WARNING: DO NOT EDIT, AUTO-GENERATED CODE - SEE TOP FOR INSTRUCTIONS */
 __u32 handle;
 };
 binder_uintptr_t cookie;
};
``` 

> ①: 分配一个`flat_binder_object`大小的空间(详见3.2.6);  
> ②: type的类型为`BINDER_TYPE_BINDER`时则type传入的是binder实体,一般是服务端注册服务时传入;  
>  type的类型为`BINDER_TYPE_HANDLE`时则type传入的为handle,一般由客户端请求服务时;  
> ③: `obj->binder`值,跟随type改变;  

## 3.2.6 bio_alloc_obj

```
static struct flat_binder_object *bio_alloc_obj(struct binder_io *bio)
{
    struct flat_binder_object *obj;

    obj = bio_alloc(bio, sizeof(*obj));                                    ①

    if (obj && bio->offs_avail) {
        bio->offs_avail--;
        *bio->offs++ = ((char*) obj) - ((char*) bio->data0);               ②
        return obj;
    }

    bio->flags |= BIO_F_OVERFLOW;
    return NULL;
}
``` 

> ①: 在data后分配`struct flat_binder_object`长度的空间;  
> ②: bio->offs空间记下此时插入obj,相对于data0的偏移值;   

看到这终于知道offs是干嘛的了,原来是用来记录数据中是否有obj类型的数据;  

## 3.2.7 完整数据格式图
综上分析,传输一次完整的数据的格式如下:

![](/images/binder/data_fomat.png)



# 3.3  binder_call

```
int binder_call(struct binder_state *bs,
                struct binder_io *msg, struct binder_io *reply,
                uint32_t target, uint32_t code)
{
    int res;
    struct binder_write_read bwr;
    struct {
        uint32_t cmd;
        struct binder_transaction_data txn;
    } __attribute__((packed)) writebuf;
    unsigned readbuf[32];

    if (msg->flags & BIO_F_OVERFLOW) {
        fprintf(stderr,"binder: txn buffer overflow\n");
        goto fail;
    }

    writebuf.cmd = BC_TRANSACTION;   // binder call transaction 
    writebuf.txn.target.handle = target;                                               ①
    writebuf.txn.code = code;                                                          ②
    writebuf.txn.flags = 0;
    writebuf.txn.data_size = msg->data - msg->data0;                                   ③
    writebuf.txn.offsets_size = ((char*) msg->offs) - ((char*) msg->offs0);
    writebuf.txn.data.ptr.buffer = (uintptr_t)msg->data0;
    writebuf.txn.data.ptr.offsets = (uintptr_t)msg->offs0;

    bwr.write_size = sizeof(writebuf);                                                 ④
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) &writebuf;

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);                                  ⑤

        if (res < 0) {
            fprintf(stderr,"binder: ioctl failed (%s)\n", strerror(errno));
            goto fail;
        }

        res = binder_parse(bs, reply, (uintptr_t) readbuf, bwr.read_consumed, 0);      ⑥
        if (res == 0) return 0;
        if (res < 0) goto fail;
    }

fail:
    memset(reply, 0, sizeof(*reply));
    reply->flags |= BIO_F_IOERROR;
    return -1;
}
```

> ①: 这个target就是我们这次请求服务的目标,即ServiceManager;  
> ②: code是我们请求服务的功能码,由服务端提供;  
> ③: 把`binder_io`数据转化成`binder_transaction_data`数据;   
> ④: 驱动进行读写是根据这个size来的,分析驱动的时候再详细分析;  
> ⑤: 进行一次读写;  
> ⑥: 解析发送的后返回的数据,判断是否注册成功;  

# 3.4  binder_done

```
void binder_done(struct binder_state *bs,
                 struct binder_io *msg,
                 struct binder_io *reply)
{
    struct {
        uint32_t cmd;
        uintptr_t buffer;
    } __attribute__((packed)) data;

    if (reply->flags & BIO_F_SHARED) {
        data.cmd = BC_FREE_BUFFER;
        data.buffer = (uintptr_t) reply->data0;
        binder_write(bs, &data, sizeof(data));
        reply->flags = 0;
    }
}
``` 

这个函数比较简单发送`BC_FREE_BUFFER`命令给驱动,让驱动释放内核态由刚才交互分配的buf;  


# 3.5 binder_set_maxthreads

```
void binder_set_maxthreads(struct binder_state *bs, int threads)
{
	ioctl(bs->fd, BINDER_SET_MAX_THREADS, &threads);
}
```

这里主要调用`ioctl`函数写入命令`BINDER_SET_MAX_THREADS`进行设置最大线程数;  

# 3.6 调用时序图

led_control_server主要提供led的控制服务,具体的流程如下:  
![](/images/binder/led_control_server.png)



# 四. test_client

# 4.1 main

```
int main(int argc, char **argv)
{
    struct binder_state *bs;
    uint32_t svcmgr = BINDER_SERVICE_MANAGER;
    unsigned int g_led_control_handle;

    if (argc < 3) {
        ALOGE("Usage:\n");
        ALOGE("%s led <on|off>\n", argv[0]);
        return -1;
    }

    bs = binder_open(128*1024);                                                        ①
    if (!bs) {
        ALOGE("failed to open binder driver\n");
        return -1;
    }

    g_led_control_handle = svcmgr_lookup(bs, svcmgr, LED_CONTROL_SERVER_NAME);         ②
    if (!g_led_control_handle) {
        ALOGE( "failed to get led control service\n");
        return -1;
    }

    ALOGI("Handle for led control service = %d\n", g_led_control_handle);

    if (!strcmp(argv[1], "led")) {
        if (!strcmp(argv[2], "on")) {
            if (interface_led_on(bs, g_led_control_handle, 2) == 0) {                  ③
                ALOGI("led was on\n");
            }
        } else if (!strcmp(argv[2], "off")) {
            if (interface_led_off(bs, g_led_control_handle, 2) == 0) {
                ALOGI("led was off\n");
            }
        }
    }

    binder_release(bs, g_led_control_handle);                                          ④

    return 0;
}
``` 

> ①: 打开binder设备(详见2.2);  
> ②: 根据名字获取led控制服务;  
> ③: 根据获取到的handle,调用led控制服务(详见4.3);  
> ④: 释放服务;  

client的流程也很简单,按步骤1.2.3.4读下来就是了;  

# 4.2 svcmgr_lookup

```
uint32_t svcmgr_lookup(struct binder_state *bs, uint32_t target, const char *name)
{
    uint32_t handle;
    unsigned iodata[512/4];
    struct binder_io msg, reply;

    bio_init(&msg, iodata, sizeof(iodata), 4);                                         ①
    bio_put_uint32(&msg, 0);  // strict mode header
    bio_put_string16_x(&msg, SVC_MGR_NAME);
    bio_put_string16_x(&msg, name);

    if (binder_call(bs, &msg, &reply, target, SVC_MGR_GET_SERVICE))                    ②
        return 0;

    handle = bio_get_ref(&reply);                                                      ③

    if (handle)
        binder_acquire(bs, handle);                                                    ④

    binder_done(bs, &msg, &reply);                                                     ⑤

    return handle;
}
``` 

> ①: 因为是请求服务,所以这里不用添加binder实体数据,具体的参考3.2,这里就不重复解释了;  
> ②: 向target进程(ServiceManager)请求获取led_control服务(详细参考3.3);  
> ③: 从ServiceManager返回的数据buf中获取led_control服务的handle;  
> ④: 增加该handle的引用计数;  
> ⑤: 释放内核空间buf(详3.4);  

## 4.2.1 bio_get_ref

```
uint32_t bio_get_ref(struct binder_io *bio)
{
    struct flat_binder_object *obj;

    obj = _bio_get_obj(bio);                                                          ①
    if (!obj)
        return 0;

    if (obj->type == BINDER_TYPE_HANDLE)                                              ②
        return obj->handle;

    return 0;
}
```

> ①: 把bio的数据转化成flat_binder_object格式;  
> ②: 判断binder数据类型是否为引用,是则返回获取到的handle;  


## 4.2.2 _bio_get_obj

```
static struct flat_binder_object *_bio_get_obj(struct binder_io *bio)
{
    size_t n;
    size_t off = bio->data - bio->data0;                                              ①
    /* TODO: be smarter about this? */
    for (n = 0; n < bio->offs_avail; n++) {
        if (bio->offs[n] == off)
            return bio_get(bio, sizeof(struct flat_binder_object));                   ②
    }

    bio->data_avail = 0;
    bio->flags |= BIO_F_OVERFLOW;
    return NULL;
}
```

> ①: 一般情况下该值都为0,因为在reply时获取ServiceManager传来的数据,bio->data和bio->data都指向同一个地址;  
> ②: 获取到`struct flat_binder_object`数据的头指针;  

从ServiceManager传来的数据是`struct flat_binder_object`的数据,格式如下:  
![](/images/binder/flat_object.png)


# 4.3  interface_led_on

```
int interface_led_on(struct binder_state *bs, unsigned int handle, unsigned char led_enum)
{
    unsigned iodata[512/4];
    struct binder_io msg, reply;
    int ret = -1;
    int exception;

    bio_init(&msg, iodata, sizeof(iodata), 4);
    bio_put_uint32(&msg, 0);  // strict mode header
    bio_put_uint32(&msg, led_enum);

    if (binder_call(bs, &msg, &reply, handle, LED_CONTROL_ON))
        return ret;

    exception = bio_get_uint32(&reply);
    if (exception == 0)
        ret = bio_get_uint32(&reply);

    binder_done(bs, &msg, &reply);

    return ret;
}
``` 

这个流程和前面`svcmgr_lookup`的请求服务差不多,只是最后是获取`led_control_server`的返回值.  
注意这里为什么获取了两次`uint32`类型的数据,这是因为服务方在回复数据的时候添加了头帧,这个是可以调节的,非规则;  

# 4.4 binder_release

```
void binder_release(struct binder_state *bs, uint32_t target)
{
    uint32_t cmd[2];
    cmd[0] = BC_RELEASE;
    cmd[1] = target;
    binder_write(bs, cmd, sizeof(cmd));
}
``` 

通知驱动层减小对`target`进程的引用,结合驱动讲解就更能明白了;  


# 4.5 调用时序图
test_client的调用时序如下,过程和`led_control_server`的调用过程相识:  
![](/images/binder/test_client.png)
# A: 表BR_含义

BR个人理解是缩写为binder reply


| 消息 | 含义 | 参数 |
| ------ | ------ | ------ |
| BR_ERROR | 发生内部错误（如内存分配失败） | --- |
| BR_OK <br> BR_NOOP | 操作完成 | --- |
|BR_SPAWN_LOOPER | 该消息用于接收方线程池管理。当驱动发现接收方所有<br>线程都处于忙碌状态且线程池里的线程总数没有超过<br>BINDER_SET_MAX_THREADS设置的最大线程数时，<br>向接收方发送该命令要求创建更多线程以备接收数据。| --- |
| BR_TRANSACTION |对应发送方的BC_TRANSACTION | binder_transaction_data |
| BR_REPLY | 对应发送方BC_REPLY的回复| binder_transaction_data |
| BR_ACQUIRE_RESULT <br> BR_FINISHED | 未使用 | --- |
| BR_DEAD_REPLY | 交互时向驱动发送binder调用，如果对方已经死亡，则<br>驱动回应此命令| --- |
|BR_TRANSACTION_COMPLETE | 发送方通过BC_TRANSACTION或BC_REPLY发送<br>完一个数据包后，都能收到该消息做为成功发送的反馈。<br>这和BR_REPLY不一样，是驱动告知发送方已经发送成<br>功，而不是Server端返回请求数据。所以不管<br>同步还是异步交互接收方都能获得本消息。| --- |
| BR_INCREFS <br> BR_ACQUIRE <br> BR_RELEASE <br> BR_DECREFS | 这一组消息用于管理强/弱指针的引用计数。只有<br>提供Binder实体的进程才能收到这组消息。| binder_uintptr_t binder：Binder实体在用户空间中的指针  <br> binder_uintptr_t cookie：与该实体相关的附加数据 |
| BR_DEAD_BINDER <br>  |向获得Binder引用的进程发送Binder实体<br>死亡通知书；收到死亡通知书的进程接下<br>来会返回BC_DEAD_BINDER_DONE做确认。 | --- |
|BR_CLEAR_DEATH_NOTIFICATION_DONE | 回应命令BC_REQUEST_DEATH_NOTIFICATION | --- |
|BR_FAILED_REPLY| 如果发送非法引用号则返回该消息 |--- |


# B: 表BC_含义

BC个人理解是缩写为binder call or cmd

| 消息 | 含义 | 参数 |
| ------ | ------ | ------ |
|BC_TRANSACTION <br> BC_REPLY| BC_TRANSACTION用于Client向Server发送请求数据；<br>BC_REPLY用于Server向Client发送回复（应答）数据。<br>其后面紧接着一个binder_transaction_data结构体表明要写<br>入的数据。| struct binder_transaction_data |
|BC_ACQUIRE_RESULT <br> BC_ATTEMPT_ACQUIRE| 未使用 | --- |
|BC_FREE_BUFFER |请求驱动释放调刚在内核空间创建用来保存用户空间数据的内存块| --- |
|BC_INCREFS <br> BC_ACQUIRE <br> BC_RELEASE <br> BC_DECREFS |这组命令增加或减少Binder的引用计数，用以实现强指针或<br>弱指针的功能。| --- |
|BC_INCREFS_DONE <br> BC_ACQUIRE_DONE|第一次增加Binder实体引用计数时，驱动向Binder<br>实体所在的进程发送BR_INCREFS， BR_ACQUIRE消息；<br>Binder实体所在的进程处理完毕回馈BC_INCREFS_DONE，<br> BC_ACQUIRE_DONE| --- |
|BC_REGISTER_LOOPER <br> BC_ENTER_LOOPER <br>  BC_EXIT_LOOPER |这组命令同BINDER_SET_MAX_THREADS一道实现Binder驱<br> 动对接收方线程池管理。BC_REGISTER_LOOPER通知驱动线程<br>池中一个线程已经创建了；BC_ENTER_LOOPER通知驱动该线程<br>已经进入主循环，可以接收数据；BC_EXIT_LOOPER通知驱动<br>该线程退出主循环，不再接收数据。| --- |
| BC_REQUEST_DEATH_NOTIFICATION | 获得Binder引用的进程通过该命令要求驱动在Binder实体销毁得到<br>通知。虽说强指针可以确保只要有引用就不会销毁实体，但这毕竟<br>是个跨进程的引用，谁也无法保证实体由于所在的Server关闭Binder<br>驱动或异常退出而消失，引用者能做的是要求Server在此刻给出通知。| --- |
|BC_DEAD_BINDER_DONE | 收到实体死亡通知书的进程在删除引用后用本命令告知驱动。| --- |

# 参考
表格参考博客:  
- https://blog.csdn.net/universus/article/details/6211589

