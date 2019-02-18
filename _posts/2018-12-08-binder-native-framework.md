---
layout: post
title:  "Binder机制情景分析之native层浅析"
date:   2018-12-8
catalog:  true
tags:
    - android
    - binder
    - native
---

# 一. 概述

浅析下android的native层binder的实现,基于android源码5.1,参考mediaService的代码;


## 1.1 问题

因为前两篇我们用c写过binder的实例,实现了service和client端,也分析了驱动,从上到下好好的看了下binder的实现原理,但是当我自己看到binder的native的源码时,一脸懵逼,封装的太厉害了,脑海中产生了很多疑问,如下:  

- a. native是如何去和ServiceManager通信的?
- b. service是如何去注册服务的?
- c. service是如何响应服务的?
- d. 怎么发送数据给驱动的?

## 1.2 源码参考

> frameworks\av\include\media\IMediaPlayerService.h  
frameworks\av\media\libmedia\IMediaPlayerService.cpp  
frameworks\av\media\libmediaplayerservice\MediaPlayerService.h  
frameworks\av\media\libmediaplayerservice\MediaPlayerService.cpp  
frameworks\av\media\mediaserver\Main_mediaserver.cpp   (server, addService)  

## 1.3 模板类

binder的native涉及到两个模板类,分别是`Bnxxx`和`Bpxxx`,前者你可以理解为binder native,主要用于service端的服务对象,后者你可以理解为binder proxy,一个代理对象,用service封装好的接口,这要client端调用后,就可以执行对应的服务;对比下c代码实现的client时调用`interface_led_on`函数,现在不需要自己实现而是service端帮忙封装好,


## 1.4. 剖析源码

从主函数看起,参考media源码:  `Main_mediaserver.cpp `

```
int main(int argc __unused, char** argv)
{
      ....
    if (doLog && (childPid = fork()) != 0) {
      .....
    } else {
        ....
        sp<ProcessState> proc(ProcessState::self());            ①
        sp<IServiceManager> sm = defaultServiceManager();       ②
        ....
        MediaPlayerService::instantiate();                      ③
        ....
        ProcessState::self()->startThreadPool();                ④
        IPCThreadState::self()->joinThreadPool();               ⑤
    }
}
```

> ①: 获取一个ProcessState实例(详见2.1);  
②: 获取一个ServiceManager服务(BpServiceManager对象);  
③: 添加媒体播放服务;  
④: 创建一个线程池;  
⑤: 进入主循环;  


# 二. 打开驱动通道

## 2.1 ProcessState::self

```
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState;
    return gProcess;
}
```

可以看出这是个单例,也就是说一个进程只允许一个`ProcessState`对象被创建;

## 2.2 ProcessState::ProcessState

```
ProcessState::ProcessState()
    : mDriverFD(open_driver())                                                                        ①
    , mVMStart(MAP_FAILED)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // XXX Ideally, there should be a specific define for whether we
        // have mmap (or whether we could possibly have the kernel module
        // availabla).
#if !defined(HAVE_WIN32_IPC)
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);     ②
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
            close(mDriverFD);
            mDriverFD = -1;
        }
#else
        mDriverFD = -1;
#endif
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}
```

> ①: 打开binder驱动设备并初始化文件描述符;  
②: 建立内存映射;  

是不是觉得场景很熟悉,这和我们前面用c写的流程是一样的,先打开一个binder设备再建立内存映射;  
不过`open_driver`函数中它多做一个功能就是设置binder的最大线程数,具体代码我这就不贴了,避免篇幅过长;   


# 三. 获取沟通桥梁

## 3.1 defaultServiceManager

```
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
    
    {
        AutoMutex _l(gDefaultServiceManagerLock);
        while (gDefaultServiceManager == NULL) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(NULL));
            if (gDefaultServiceManager == NULL)
                sleep(1);
        }
    }
    
    return gDefaultServiceManager;
}
```

这也是个单例,通过`interface_cast`这个模板函数将`getContextObject`的值转换成`IServiceManager`类型,我们先看下`getContextObject`到底返回了什么给我们?  

## 3.2 ProcessState::getContextObject

```
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}
```

通过`getStrongProxyForHandle`获取一个`IBinder`对象;  

## 3.3 ProcessState::getStrongProxyForHandle

```
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);                                ①

    if (e != NULL) {

        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {                       ②
            if (handle == 0) {
                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }

            b = new BpBinder(handle);                                            ③
            e->binder = b;                                                       ④
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}

```

> ①:  根据handle在`mHandleToObject`容器中查找是否有对应object,没有则插入一个新的object;  
②: 判断刚返回的object中的binder成员是否为空,为空说明是新创建的;  
③: 创建一个新的`BpBinder`对象,根据`handle`值(详见3.4);  
④: 将创建的`BpBinder`对象填充进object;  

注意`BpBinder`这是个代理对象且它的基类为`IBinder`,接下来看下这个代理对象到底做了什么?   
其实根据这个函数的名字我们也能猜出点东西,结合笔者前面`C服务应用`时写的,client端在获取到一个handle时,一直都是通过这个handle去与驱动进行交互;  
那么我们这个handle是不是也是做同样的功能,转换成面向对象的写法变成了一个代理对象?  

## 3.4 BpBinder::BpBinder

```
BpBinder::BpBinder(int32_t handle)
    : mHandle(handle)
    , mAlive(1)
    , mObitsSent(0)
    , mObituaries(NULL)
{
    ALOGV("Creating BpBinder %p handle %d\n", this, mHandle);

    extendObjectLifetime(OBJECT_LIFETIME_WEAK);
    IPCThreadState::self()->incWeakHandle(handle);
}
```

`BpBinder`的构造函数保留下handle的值并对该handle增加了引用,看到这里可以结合`C服务应用`篇联想下该类会做什么功能;  
当笔者看到这的时候,认为这是一个客户端用来和service交互的代理类,完成驱动交互的工作,类似`binder_call`函数;   


## 3.5 BpBinder::transact

```
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```

看到`BpBinder`类中的这个方法,完全验证了我们的思想,但是发现它并不是直接去和驱动交互而是利用`IPCThreadState::self()->transact`这个方法进行,这个我们后面再分析;  


## 3.6 interface_cast

前面在分析`defaultServiceManager`時,中有个模板类进行类型转换,前面也分析`ProcessState::self()->getContextObject`是返回一个`Bpbinder`对象,现在看下如何转换;  

```
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}
```
将里面 的`INTERFACE`替换成`IServiceManager`,其实是这样的`IServiceManager::asInterface(obj)`; 在`IServiceManager`类中找了一圈发现没有该方法,最后发现是用宏定义实现,如下:  

> #define DECLARE_META_INTERFACE(INTERFACE)   
#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)   

## 3.7 IMPLEMENT_META_INTERFACE

因为`DECLARE_META_INTERFACE`仅仅只是声明,替换下就可以看出实现的东西了,我们就分析下`IMPLEMENT_META_INTERFACE`看看是这其中做了什么;  

```
#define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)                       \
    const android::String16 I##INTERFACE::descriptor(NAME);             \
    const android::String16&                                            \
            I##INTERFACE::getInterfaceDescriptor() const {              \
        return I##INTERFACE::descriptor;                                \
    }                                                                   \
    android::sp<I##INTERFACE> I##INTERFACE::asInterface(                \
            const android::sp<android::IBinder>& obj)                   \
    {                                                                   \
        android::sp<I##INTERFACE> intr;                                 \
        if (obj != NULL) {                                              \
            intr = static_cast<I##INTERFACE*>(                          \
                obj->queryLocalInterface(                               \
                        I##INTERFACE::descriptor).get());               \
            if (intr == NULL) {                                         \
                intr = new Bp##INTERFACE(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    I##INTERFACE::I##INTERFACE() { }                                    \
    I##INTERFACE::~I##INTERFACE() { }    
```

`IMPLEMENT_META_INTERFACE(ServiceManager, "android.os.IServiceManager");`转换后如下:  

```
    const android::String16 IServiceManager::descriptor("android.os.IServiceManager");             \
    const android::String16&                                            \
            IIServiceManager::getInterfaceDescriptor() const {              \
        return IIServiceManager::descriptor;                                \
    }                                                                   \
    android::sp<IServiceManager> IServiceManager::asInterface(                \
            const android::sp<android::IBinder>& obj)                   \
    {                                                                   \
        android::sp<IServiceManager> intr;                                 \
        if (obj != NULL) {                                              \
            intr = static_cast<IServiceManager*>(                          \
                obj->queryLocalInterface(                               \
                        IServiceManager::descriptor).get());               \
            if (intr == NULL) {                                         \
                intr = new BpServiceManager(obj);                          \
            }                                                           \
        }                                                               \
        return intr;                                                    \
    }                                                                   \
    IServiceManager::IServiceManager() { }                                    \
    IServiceManager::~IServiceManager() { }
```

***注意:***`IServiceManager::asInterface`方法,他将传进去的`BpBiner`对象又用来创建`BpServiceManager`对象;接下来关注下`BpServiceManager`的实现;  

## 3.8 BpServiceManager::BpServiceManager

```
    BpServiceManager(const sp<IBinder>& impl)
        : BpInterface<IServiceManager>(impl)
    {
    }

```

被用来初始化`BpInterface`模板了,接着往下看这个模板类做了什么;  

## 3.9 BpInterface::BpInterface

```
template<typename INTERFACE>
inline BpInterface<INTERFACE>::BpInterface(const sp<IBinder>& remote)
    : BpRefBase(remote)
{
}

```

这里回顾下,怕读者看蒙圈了,传进来的`remote`变量是我们前面通过`getContextObject`方法,获取到的handle为0的一个`BpBinder`对象,该对象是个代理类,是`IBinder`的派生类,它可以与ServiceManager进行通信的一个代理类;`client`和`service`都可以通过该与`ServiceManager`进行通信,前者可以通过它获取服务,后者可以通过它注册服务,重点是handle为0,它代表着`ServiceManager`,不理解的笔者可以看下C服务应用篇;   

接下来看下`BpRefBase`里做了什么;   

## 3.10 BpRefBase::BpRefBase

```
BpRefBase::BpRefBase(const sp<IBinder>& o)
    : mRemote(o.get()), mRefs(NULL), mState(0)
{
    extendObjectLifetime(OBJECT_LIFETIME_WEAK);

    if (mRemote) {
        mRemote->incStrong(this);           // Removed on first IncStrong().
        mRefs = mRemote->createWeak(this);  // Held for our entire lifetime.
    }
}
```

***注意:***`mRemote`成员,它是个`IBinder`指针类型,它指向了传进来的`BpBinder`对象;当笔者看到者时突然顿悟,这样`BpServiceManager`就被赋予了力量可以通过它去和`ServiceManager`打交道了;


# 四. 到达彼岸

***PS:***`defaultServiceManager`单例获取到的对象是`BpServiceManager`对象;  
`ServiceManager`进行沟通的对象,我们已经知道如何获取到了现在看下`service`如何去注册自己;  


## 4.1 MediaPlayerService::instantiate

```
void MediaPlayerService::instantiate() {
    defaultServiceManager()->addService(
            String16("media.player"), new MediaPlayerService());
}
```
通俗点就是调用`defaultServiceManager`单例中的`addService`方法,将`MediaPlayerService`服务注册,接下来看下`addService`做了什么;  

## 4.2 BpServiceManager::addService
```
    virtual status_t addService(const String16& name, const sp<IBinder>& service,
            bool allowIsolated)
    {
        Parcel data, reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeString16(name);
        data.writeStrongBinder(service);
        data.writeInt32(allowIsolated ? 1 : 0);
        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
        return err == NO_ERROR ? reply.readExceptionCode() : err;
    }
```

是不是觉得事成相识的感觉,与前面`C服务应用篇`一样,构造好数据然后发送;`remote`方法不就是我们前面分析时`BpRefBase`类中的方法,返回一个`BpBinder`指针(mRemote);该方法怎么实现后面再说,这里马后炮一下,先讲下`writeStrongBinder`;  

## 4.3 Parcel::writeStrongBinder

```
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}


status_t flatten_binder(const sp<ProcessState>& /*proc*/,
    const sp<IBinder>& binder, Parcel* out)
{
    flat_binder_object obj;

    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    if (binder != NULL) {
        IBinder *local = binder->localBinder();                        ①
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();                  ②
            if (proxy == NULL) {
                ALOGE("null proxy");
            }
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.type = BINDER_TYPE_HANDLE;
            obj.binder = 0; /* Don't pass uninitialized stack data to a remote process */
            obj.handle = handle;
            obj.cookie = 0;
        } else {
            obj.type = BINDER_TYPE_BINDER;
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);           ③
        }
    } else {
        obj.type = BINDER_TYPE_BINDER;
        obj.binder = 0;
        obj.cookie = 0;
    }

    return finish_flatten_binder(binder, obj, out);                    ④
}
```

> ①: 获取BBinder对象;  
②: 获取一个BpBinder对象;  
③: 将获取到BBinder对象保存进cookie;  
④: 将该构造好的obj写进数据块中;  

***注意:***形参中传进来的`binder`是个Service服务对象,通过`localBinder`方法查看传进来的binder是否有由BBinder对象派生,不是说明这是一个handle,即请求服务,否则为注册服务;与C服务应用篇做对别,有个差别就是这里用到了cookie,将Service对象保存下来了,一开始看到这里不理解,看到后面才明白的;  

`localBinder`方法由`BBinder`继承后,会返回一个`BBinder`本身的指针,否则会返回null;  


## 4.4 BnMediaPlayerService::onTransact

我们不关心` new MediaPlayerService()`是如何构造的,前面分析到需要通过继承`BBinder`来判断是注册还是请求服务;  
接下来看下媒体类的继承关系;  

```
class MediaPlayerService : public BnMediaPlayerService
    class BnMediaPlayerService: public BnInterface<IMediaPlayerService>
        class BnInterface : public INTERFACE, public BBinder
```

这样一层层下来,最后发现是通过`BnInterface`继承了`BBinder`函数;  继续看`BBinder`中的`localBinder`方法:  

```
BBinder* BBinder::localBinder()
{
    return this;
}
```

果然和我们前面分析的一样,返回了BBinder本身;  

其实主角是我们的`onTransact`方法:  

```
status_t BBinder::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t /*flags*/)
{
    switch (code) {
        case INTERFACE_TRANSACTION:
            reply->writeString16(getInterfaceDescriptor());
            return NO_ERROR;

        case DUMP_TRANSACTION: {
            int fd = data.readFileDescriptor();
            int argc = data.readInt32();
            Vector<String16> args;
            for (int i = 0; i < argc && data.dataAvail() > 0; i++) {
               args.add(data.readString16());
            }
            return dump(fd, args);
        }

        case SYSPROPS_TRANSACTION: {
            report_sysprop_change();
            return NO_ERROR;
        }

        default:
            return UNKNOWN_TRANSACTION;
    }
}
```

该方法由派生类复写,用于服务客户端的请求服务,根据客户端传来的不同的`code`执行不同的服务;  

```
class BnMediaPlayerService: public BnInterface<IMediaPlayerService>
{
public:
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
};
```

`BnMediaPlayerService`是有复写该方法的;  


## 4.5 IPCThreadState::transact

数据的构造,注册我们都知道怎么实现了,但是怎么发送给驱动我们还没有分析,前面分析时`BpBinder::transact`的实现是通过`IPCThreadState::transact`的方法来实现,接下来分析下:  

```
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();

    flags |= TF_ACCEPT_FDS;

    if (err == NO_ERROR) {
        LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
            (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
        err = writeTransact
        ionData(BC_TRANSACTION, flags, handle, code, data, NfaULL);    ①
    }
    
    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }
    
    if ((flags & TF_ONE_WAY) == 0) {

        if (reply) {
            err = waitForResponse(reply);                                               ②
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
              
        IF_LOG_TRANSACTIONS() {
            TextOutput::Bundle _b(alog);
            alog << "BR_REPLY thr " << (void*)pthread_self() << " / hand "
                << handle << ": ";
            if (reply) alog << indent << *reply << dedent << endl;
            else alog << "(none requested)" << endl;
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }
    
    return err;
}
```

> ①: 构造发送数据和命令;  
②: 等待响应;  


笔者刚开始看的时候一脸懵逼,根据方法名,构造数据,等待响应,那发送呢?  
一开始以为在构造数据的时候并发出去了,毕竟它用了write;后来才发现秘密在`waitForResponse`方法中;  


## 4.6 IPCThreadState::waitForResponse


```
status_t IPCThreadState::sendReply(const Parcel& reply, uint32_t flags)
{
    status_t err;
    status_t statusBuffer;
    err = writeTransactionData(BC_REPLY, flags, -1, 0, reply, &statusBuffer);
    if (err < NO_ERROR) return err;
    
    return waitForResponse(NULL, NULL);
}

status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;                        ①
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;
        
        cmd = mIn.readInt32();                                               ②
        
        IF_LOG_COMMANDS() {
            alog << "Processing waitForResponse Command: "
                << getReturnString(cmd) << endl;
        }

        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;
        
        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;

        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;
        
        case BR_ACQUIRE_RESULT:
            {
                ALOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");
                const int32_t result = mIn.readInt32();
                if (!acquireResult) continue;
                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
            }
            goto finish;
        
        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(NULL,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t), this);
                    }
                } else {
                    freeBuffer(NULL,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t), this);
                    continue;
                }
            }
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }
    
    return err;
}
```

> ①: 将数据写入驱动  
②: 读取驱动返回的数据,根据不同的cmd值做出相应回应;  

写进去,接下来就读了说明`talkWithDriver`是个阻塞方法,点进去看下;  


## 4.7 IPCThreadState::talkWithDriver
```
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    ...
    
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        IF_LOG_COMMANDS() {
            alog << "About to read/write, write size = " << mOut.dataSize() << endl;
        }
#if defined(HAVE_ANDROID_OS)
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)                ①
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD <= 0) {
            err = -EBADF;
        }
        IF_LOG_COMMANDS() {
            alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
        }
    } while (err == -EINTR);

   ...
    return err;
}
```

> ①: 调用`ioctl`函数将数据写进驱动;  

看到了吗一个do-while阻塞在这里,其实看过驱动就知道调用`ioctl`时驱动层在没有数据时会休眠;  

到这里注册服务就完成了,最后是通过`BnMediaPlayerService`来提供服务的; 


# 五. 等待服务

服务实现了,也注册了,还有最后一步等待服务请求;

## 5.1 ProcessState::startThreadPool

```
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}

void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);                            ①
        t->run(name.string());
    }
}

class PoolThread : public Thread
{
public:
    PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }
    
protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);                   ②
        return false;
    }
    
    const bool mIsMain;
};

```

> ①: 创建了一个线程池;  
②: 也是通过`IPCThreadState::self()->joinThreadPool`进入循环;

还记得一开始我们设置过线程最大数吗,每个Service是可能同时被多个client请求提供服务的,忙不过来就只能动态创建线程来对应请求服务;  
接下来看下`joinThreadPool`做了什么,大胆的猜测下,是不是进入一个循环,接着调用`transact`读取数据然后等待数据,来数据后进行解析,然后执行对应的服务;  


## 5.2 IPCThreadState::joinThreadPool

```
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
    
    set_sched_policy(mMyThreadId, SP_FOREGROUND);
        
    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
            abort();
        }
        
        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%p\n",
        (void*)pthread_self(), getpid(), (void*)result);
    
    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}
```

打脸了,只进入了一个循环然后调用了`getAndExecuteCommand`方法,来看看该方法做了什么;  

## 5.3 IPCThreadState::getAndExecuteCommand

```
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    result = talkWithDriver();                                                ①
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32();
        IF_LOG_COMMANDS() {
            alog << "Processing top-level Command: "
                 << getReturnString(cmd) << endl;
        }

        result = executeCommand(cmd);                                         ②

        // After executing the command, ensure that the thread is returned to the
        // foreground cgroup before rejoining the pool.  The driver takes care of
        // restoring the priority, but doesn't do anything with cgroups so we
        // need to take care of that here in userspace.  Note that we do make
        // sure to go in the foreground after executing a transaction, but
        // there are other callbacks into user code that could have changed
        // our group so we want to make absolutely sure it is put back.
        set_sched_policy(mMyThreadId, SP_FOREGROUND);
    }

    return result;
}
```
> ①: 前面我们分析过了这是和驱动交互的,它回去读和写;  
②: 从读取到的数据中执行相应的code;  



## 5.4 IPCThreadState::executeCommand

```
status_t IPCThreadState::executeCommand(int32_t cmd)
{
...
    
    case BR_TRANSACTION:
        {
            binder_transaction_data tr;
            result = mIn.read(&tr, sizeof(tr));
            ALOG_ASSERT(result == NO_ERROR,
                "Not enough command data for brTRANSACTION");
            if (result != NO_ERROR) break;
            
            Parcel buffer;
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);
            
            const pid_t origPid = mCallingPid;
            const uid_t origUid = mCallingUid;
            const int32_t origStrictModePolicy = mStrictModePolicy;
            const int32_t origTransactionBinderFlags = mLastTransactionBinderFlags;

            mCallingPid = tr.sender_pid;
            mCallingUid = tr.sender_euid;
            mLastTransactionBinderFlags = tr.flags;

            int curPrio = getpriority(PRIO_PROCESS, mMyThreadId);
            if (gDisableBackgroundScheduling) {
                if (curPrio > ANDROID_PRIORITY_NORMAL) {
                    // We have inherited a reduced priority from the caller, but do not
                    // want to run in that state in this process.  The driver set our
                    // priority already (though not our scheduling class), so bounce
                    // it back to the default before invoking the transaction.
                    setpriority(PRIO_PROCESS, mMyThreadId, ANDROID_PRIORITY_NORMAL);
                }
            } else {
                if (curPrio >= ANDROID_PRIORITY_BACKGROUND) {
                    // We want to use the inherited priority from the caller.
                    // Ensure this thread is in the background scheduling class,
                    // since the driver won't modify scheduling classes for us.
                    // The scheduling group is reset to default by the caller
                    // once this method returns after the transaction is complete.
                    set_sched_policy(mMyThreadId, SP_BACKGROUND);
                }
            }

            //ALOGI(">>>> TRANSACT from pid %d uid %d\n", mCallingPid, mCallingUid);

            Parcel reply;
            status_t error;

            if (tr.target.ptr) {
                sp<BBinder> b((BBinder*)tr.cookie);
                error = b->transact(tr.code, buffer, &reply, tr.flags);

            } else {
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            }

            //ALOGI("<<<< TRANSACT from pid %d restore pid %d uid %d\n",
            //     mCallingPid, origPid, origUid);
            
            if ((tr.flags & TF_ONE_WAY) == 0) {
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                if (error < NO_ERROR) reply.setError(error);
                sendReply(reply, 0);
            } else {
                LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
            }
            
            mCallingPid = origPid;
            mCallingUid = origUid;
            mStrictModePolicy = origStrictModePolicy;
            mLastTransactionBinderFlags = origTransactionBinderFlags;

            IF_LOG_TRANSACTIONS() {
                TextOutput::Bundle _b(alog);
                alog << "BC_REPLY thr " << (void*)pthread_self() << " / obj "
                    << tr.target.ptr << ": " << indent << reply << dedent << endl;
            }
            
        }
        break;
    
   ...

    if (result != NO_ERROR) {
        mLastError = result;
    }
    
    return result;
}
```

似曾相识的场景,但是我们重点关注下`BR_TRANSACTION`, 先回想下用C实现时在接收到`BR_TRANSACTION`消息时是什么流程?

> ①: 将读取的数据转换成`binder_transaction_data`类型;  
> ②: 调用形参的函数指针;  
> ③:发送回复数据;  


那看看我们这里好像和原来描述的步骤一样,只是执行服务的方式变了,注意看着:  

```
            if (tr.target.ptr) {
                sp<BBinder> b((BBinder*)tr.cookie);
                error = b->transact(tr.code, buffer, &reply, tr.flags);
            }
```

将cookie转换成了BBinder对象,这个值是我们写C时没有用到的,笔者在写4.3节的时候马后炮指的就是这了;  
在注册服务的时候binder的服务已经被保存在这了,这里执行`transact`方法就相当于执行` BnXXXService::onTransact`的方法,为什么这么说,详见BBinder的transact方法;  


## 5.5 BBinder::transact

```
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    data.setDataPosition(0);

    status_t err = NO_ERROR;
    switch (code) {
        case PING_TRANSACTION:
            reply->writeInt32(pingBinder());
            break;
        default:
            err = onTransact(code, data, reply, flags);
            break;
    }

    if (reply != NULL) {
        reply->setDataPosition(0);
    }

    return err;
}
```

`onTransact`该方法由具体的派生类复写;  


# 六. 总结

`BpxxxService`主要用于封装了一些请求服务的方法供client使用，不需要像写C语言时，客户端只是获得handle，构造数据发送啥的都要自己去实现，现在只要获取到通过`interface_cast`就可以将对应的`BpBinder`转换成你想要的Service对象（详见3.1），然后就可以调用封装好的方法去请求服务；

`BnxxxService`主要封装了提供服务的方法，在收到`BR_TRANSACTION`cmd后执行，主要继承于`BBinder`类;  

PS:  
踩个坑，在IServiceManager.cpp中有个BnServiceManager::onTransact方法，ServiceManager的服务是用c实现的，这个方法没有使用;











