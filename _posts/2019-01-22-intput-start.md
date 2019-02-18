---
layout: post
title:  "Input系统之启动过程"
date:   2019-01-22 14:53:54
catalog:  true
tags:
    - input
    - android
    - jni
---


# 一. 概述

`InputManagerService`是一个android系统服务, 它分为java层和native层两部分.  
java层负责与`WindowsManagerService`通信, 而native层则由`InputReader`, `InputDispatcher`两个关键组件和`EventHub`构成;  


## 1.1 natvie方法

在查看源代码时发现, `InputManagerService`提供很多native接口, 在这记录下jni加载过程;

### 1.1.1 loadLibrary

> [SystemServer.java--->SystemServer.run()]  

```
        System.loadLibrary("android_servers");
        nativeInit();
```


### 1.1.2 JNI_OnLoad

> [onload.cpp--->JNI_OnLoad()]  

```
extern "C" jint JNI_OnLoad(JavaVM* vm, void* reserved)
{
    JNIEnv* env = NULL;
    jint result = -1;

    if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        ALOGE("GetEnv failed!");
        return result;
    }
....
    register_android_server_InputApplicationHandle(env);
    register_android_server_InputWindowHandle(env);
    register_android_server_InputManager(env);
...

    return JNI_VERSION_1_4;
}
```

### 1.1.3 register_android_server_InputManager

> [com_android_server_input_InputManagerService.cpp--->register_android_server_InputManager()]  

```
int register_android_server_InputManager(JNIEnv* env) {
    int res = jniRegisterNativeMethods(env, "com/android/server/input/InputManagerService",
            gInputManagerMethods, NELEM(gInputManagerMethods));
    LOG_FATAL_IF(res < 0, "Unable to register native methods.");

    // Callbacks

    jclass clazz;
    FIND_CLASS(clazz, "com/android/server/input/InputManagerService");

    GET_METHOD_ID(gServiceClassInfo.notifyConfigurationChanged, clazz,
            "notifyConfigurationChanged", "(J)V");

    GET_METHOD_ID(gServiceClassInfo.notifyInputDevicesChanged, clazz,
            "notifyInputDevicesChanged", "([Landroid/view/InputDevice;)V");
....
}
```

这里不仅注册很多native方法而且还获取了很多java层方法;  
具体的native方法都在`gInputManagerMethods`表中, 这里就不贴出来了,有些长;

接下来分析下`InputManagerService`的启动分析过程

# 二. InputManagerService 创建

## 2.1 startOtherServices

> [SystemServer.java--->SystemServer.startOtherServices()]   

```
private void startOtherServices() {
...
    Slog.i(TAG, "Input Manager");
    inputManager = new InputManagerService(context);
...
}
```

## 2.2 InputManagerService

> [InputManagerService.java--->InputManagerService.InputManagerService()]   

```
public InputManagerService(Context context) {
        this.mContext = context;
        this.mHandler = new InputManagerHandler(DisplayThread.get().getLooper());

        mUseDevInputEventForAudioJack =
                context.getResources().getBoolean(R.bool.config_useDevInputEventForAudioJack);
        Slog.i(TAG, "Initializing input manager, mUseDevInputEventForAudioJack="
                + mUseDevInputEventForAudioJack);
        mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());

        LocalServices.addService(InputManagerInternal.class, new LocalService());
    }
```

这里主要调用了`nativeInit()`函数进行初始化;  

## 2.3 nativeInit

> [com_android_server_input_InputManagerService.cpp--->nativeInit()]   

```
static jlong nativeInit(JNIEnv* env, jclass clazz,
        jobject serviceObj, jobject contextObj, jobject messageQueueObj) {
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);    ①
    if (messageQueue == NULL) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }

    NativeInputManager* im = new NativeInputManager(contextObj, serviceObj,                           ②
            messageQueue->getLooper());
    im->incStrong(0);
    return reinterpret_cast<jlong>(im);
}
```

> ①: 返回java层MessageQueue对象的mPtr成员变量;  
> ②: 创建一个NativeInputManager对象;  

## 2.4 NativeInputManager

> [com_android_server_input_InputManagerService.cpp--->NativeInputManager.NativeInputManager()]   

```
NativeInputManager::NativeInputManager(jobject contextObj,
        jobject serviceObj, const sp<Looper>& looper) :
        mLooper(looper), mInteractive(true) {
    JNIEnv* env = jniEnv();

    mContextObj = env->NewGlobalRef(contextObj);
    mServiceObj = env->NewGlobalRef(serviceObj);

...
    sp<EventHub> eventHub = new EventHub();
    mInputManager = new InputManager(eventHub, this, this);
}
```

该构造函数中创建了两个新对象: EventHub 和 InputManager对象;  

## 2.5 EventHub

> [EventHub.cpp--->EventHub.EventHub()]   

```
EventHub::EventHub(void) :
        mBuiltInKeyboardId(NO_BUILT_IN_KEYBOARD), mNextDeviceId(1), mControllerNumbers(),
        mOpeningDevices(0), mClosingDevices(0),
        mNeedToSendFinishedDeviceScan(false),
        mNeedToReopenDevices(false), mNeedToScanDevices(true),
        mPendingEventCount(0), mPendingEventIndex(0), mPendingINotify(false) {
    acquire_wake_lock(PARTIAL_WAKE_LOCK, WAKE_LOCK_ID);

    mEpollFd = epoll_create(EPOLL_SIZE_HINT);                                             ①
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);

    mINotifyFd = inotify_init();
    int result = inotify_add_watch(mINotifyFd, DEVICE_PATH, IN_DELETE | IN_CREATE);       ②
    LOG_ALWAYS_FATAL_IF(result < 0, "Could not register INotify for %s.  errno=%d",
            DEVICE_PATH, errno);

    struct epoll_event eventItem;
    memset(&eventItem, 0, sizeof(eventItem));
    eventItem.events = EPOLLIN;
    eventItem.data.u32 = EPOLL_ID_INOTIFY;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mINotifyFd, &eventItem);                  ③
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add INotify to epoll instance.  errno=%d", errno);

    int wakeFds[2];
    result = pipe(wakeFds);                                                               ④
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);

    mWakeReadPipeFd = wakeFds[0];
    mWakeWritePipeFd = wakeFds[1];

    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);                                 ⑤
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",
            errno);

    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",
            errno);

    eventItem.data.u32 = EPOLL_ID_WAKE;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, &eventItem);             ⑥
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",
            errno);

    int major, minor;
    getLinuxRelease(&major, &minor);
    // EPOLLWAKEUP was introduced in kernel 3.5
    mUsingEpollWakeup = major > 3 || (major == 3 && minor >= 5);
}
```

> ①: 创建 epoll  
> ②: 创建 inotify, 并添加监控`/dev/input`路径  
> ③: 监控 inotify 文件描述符  
> ④: 创建管道  
> ⑤: 设置管道属性为非阻塞  
> ⑥: 监控管道读端  

输入系统主要使用了`inotify` + `epoll` 进行监控输入设备的插拔变化和事件产生;  


## 2.6 InputManager

> [InputManager.cpp--->InputManager::InputManager()]

```
InputManager::InputManager(
        const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& readerPolicy,
        const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
    mDispatcher = new InputDispatcher(dispatcherPolicy);
    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
    initialize();
}
```

该构造函数中又创建了两个对象, 然后调用`initialize()`函数; 


## 2.7 InputDispatcher

> [InputDispatcher.cpp--->InputDispatcher::InputDispatcher()]

```
InputDispatcher::InputDispatcher(const sp<InputDispatcherPolicyInterface>& policy) :
    mPolicy(policy),
    mPendingEvent(NULL), mAppSwitchSawKeyDown(false), mAppSwitchDueTime(LONG_LONG_MAX),
    mNextUnblockedEvent(NULL),
    mDispatchEnabled(false), mDispatchFrozen(false), mInputFilterEnabled(false),
    dontNeedFocusHome(true),
    mInputTargetWaitCause(INPUT_TARGET_WAIT_CAUSE_NONE) {
    mLooper = new Looper(false);

    mKeyRepeatState.lastKeyEntry = NULL;

    policy->getDispatcherConfiguration(&mConfig);
}
```


## 2.8 InputReader

> [InputReader.cpp--->InputReader::InputReader()]

```
InputReader::InputReader(const sp<EventHubInterface>& eventHub,
        const sp<InputReaderPolicyInterface>& policy,
        const sp<InputListenerInterface>& listener) :
        mContext(this), mEventHub(eventHub), mPolicy(policy),
        mGlobalMetaState(0), mGeneration(1),
        mDisableVirtualKeysTimeout(LLONG_MIN), mNextTimeout(LLONG_MAX),
        mConfigurationChangesToRefresh(0) {
    mQueuedListener = new QueuedInputListener(listener);

    { // acquire lock
        AutoMutex _l(mLock);

        refreshConfigurationLocked(0);
        updateGlobalMetaStateLocked();
    } // release lock
}
```

这个构造函数中引用了`Eventhub`对象, 还创建了一个队列监听者;  


## 2.9 initialize


> [InputManager.cpp--->InputManager::initialize()]

```
void InputManager::initialize() {
    mReaderThread = new InputReaderThread(mReader);
    mDispatcherThread = new InputDispatcherThread(mDispatcher);
}
```


这里只是创建线程, 但不执行;  
至此`InputManagerService`对象的创建就基本完成了, 接下来就是启动 input manager;   

## 2.10 流程总结

![InputManagerService_create](/images/input/start/InputManagerService_create.jpg)

# 三. InputManagerService 启动

## 3.1 startOtherServices

> [SystemServer.java--->SystemServer.startOtherServices()]   

```
    inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
    inputManager.start();
```

该方法的调用就在创建inputManager对象的下方;  

## 3.2 InputManagerService.start

> [InputManagerService.java--->InputManagerService.start()]   

```
    public void start() {
        Slog.i(TAG, "Starting input manager");
        nativeStart(mPtr);

        // Add ourself to the Watchdog monitors.
        Watchdog.getInstance().addMonitor(this);

        registerPointerSpeedSettingObserver();
        registerShowTouchesSettingObserver();

        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                updatePointerSpeedFromSettings();
                updateShowTouchesFromSettings();
            }
        }, new IntentFilter(Intent.ACTION_USER_SWITCHED), null, mHandler);

        updatePointerSpeedFromSettings();
        updateShowTouchesFromSettings();
    }
```

这里调用了native层的start方法, 接着往下看; 

## 3.3 nativeStart

> [com_android_server_input_InputManagerService.cpp--->nativeStart()]   


```
static void nativeStart(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    status_t result = im->getInputManager()->start();
    if (result) {
        jniThrowRuntimeException(env, "Input manager could not be started.");
    }
}
```

这里使用单例方法再调用`mInputManager`对象的start方法, 该对象的创建是在`NativeInputManager()`函数中实现的;  

## 3.3 InputManager::start

> [InputManager.cpp--->InputManager.start()]   

```
status_t InputManager::start() {
    status_t result = mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
    if (result) {
        ALOGE("Could not start InputDispatcher thread due to error %d.", result);
        return result;
    }

    result = mReaderThread->run("InputReader", PRIORITY_URGENT_DISPLAY);
    if (result) {
        ALOGE("Could not start InputReader thread due to error %d.", result);

        mDispatcherThread->requestExit();
        return result;
    }

    return OK;
}
```

这里先运行了`InputDispatcher`线程再运行`InputReader`线程, 是为了防止事件丢失;  
接下来看看这两个线程中做了哪些事情;   


## 3.4 InputDispatcherThread::threadLoop

> [InputDispatcher.cpp--->InputDispatcherThread.threadLoop()]   

```
bool InputDispatcherThread::threadLoop() {
    mDispatcher->dispatchOnce();
    return true;
}
```

这里面调用了InputDispatcher中的`dispatchOnce()`方法;  


## 3.5 InputReaderThread::threadLoop

> [InputReader.cpp--->InputReaderThread.threadLoop()]   

```
bool InputReaderThread::threadLoop() {
    mReader->loopOnce();
    return true;
}
```

这里面调用了InputReader中的`loopOnce`方法;

## 3.6 流程总结

![InputManagerService_start](/images/input/start/InputManagerService_start.jpg)


# 四. 总结

当两个线程启动后, InputReader 在线程循环中不断从 EventHub 中抽取原始输入事件, 进行  
加工处理后讲其放入InputDispatcher的派发队列中. InputDispatcher 则在其线程循环中将队列  
中事件取出, 查找合适的窗口,将事件写入窗口的事件接收管道中.  

输入系统的主要三块就是`InputReader`, `EventHub` 和 `InputDispatcher`;  
