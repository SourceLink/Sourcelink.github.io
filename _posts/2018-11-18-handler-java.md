---
layout: post
title:  "Handler机制情景分析之java层浅析"
date:   2018-11-18
catalog:  true
tags:
    - android
    - handle
---


# 一. 概述

在整个Android的源码世界里，有两大利剑，其一是Binder IPC机制，，另一个便是消息机制(由Handler/Looper/MessageQueue等构成的).  
Android有大量的消息驱动方式来进行交互，比如Android的四剑客Activity, Service, Broadcast, ContentProvider的启动过程的交互，都离不开消息机制，Android某种意义上也可以说成是一个以消息驱动的系统。消息机制涉及MessageQueue/Message/Looper/Handler这4个类。

# 1.1 模型
消息机制主要包含:  
- Message：消息分为硬件产生的消息(如按钮、触摸)和软件生成的消息；
- MessageQueue：消息队列的主要功能向消息池投递消息(`MessageQueue.enqueueMessage`)和取走消息池的消息(`MessageQueue.next`)；
- Handler：消息辅助类，主要功能向消息池发送各种消息事件(`Handler.sendMessage`)和处理相应消息事件(`Handler.handleMessage`)；
- Looper：不断循环执行(`Looper.loop`)，按分发机制将消息分发给目标处理者。


# 1.2 架构图
![](/images/handler/handler_framwork.png)

# 1.3 Demo

```
public class MainActivity extends AppCompatActivity {

    private Button mButton;
    private final String TAG="MessageTest";
    private int ButtonCount = 0;
    private MyThread myThread;
    private Handler mHandler;
    private int mMessageCount = 0;

    class MyThread extends Thread {
        private Looper mLooper;
        @Override
        public void run() {
            super.run();
            /* Initialize the current thread as a looper */
            Looper.prepare();
            synchronized (this) {
                mLooper = Looper.myLooper();
                notifyAll();
            }
            /* Run the message queue in this thread */
            Looper.loop();
        }

        public Looper getLooper(){
            if (!isAlive()) {
                return null;
            }

            // If the thread has been started, wait until the looper has been created.
            synchronized (this) {
                while (isAlive() && mLooper == null) {
                    try {
                        wait();
                    } catch (InterruptedException e) {
                    }
                }
            }
            return mLooper;
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mButton = (Button)findViewById(R.id.button);
        mButton.setOnClickListener(new View.OnClickListener() {
            public void onClick(View v) {
                // Perform action on click
                Log.d(TAG, "Send Message "+ ButtonCount);
                ButtonCount++;
                /* 按下按键后通过mHandler发送一个消息 */
                Message msg = new Message();
                mHandler.sendMessage(msg);
            }
        });

        myThread = new MyThread();
        myThread.start();

        /* 创建一个handle实例(详见4.3.2),这个handle为线程myThread服务,当收到mesg时会调用设置的回调函数*/
        mHandler = new Handler(myThread.getLooper(), new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                Log.d(TAG, "get Message "+ mMessageCount);
                mMessageCount++;
                return false;
            }
        });
    }
}
```
大概流程:先创建的一个线程,该线程中调用了`Looper.prepare()`(详见2.1)和`Looper.loop()`(详见2.2)方法,接着启动了该线程,紧接着初始化了一个Handler实例(详见4.3.2).用于服务message,在按下按键后通过`mHandler`发送了一个消息(详见4.2),此时`handleMessage`被回调(详见4.1).接下来进行详细分析.

该Demo中有个两点`getLooper`方法,当外界调用该方法时,他会判断当前`mLooper`是否为空,空的话就会一直等待.  
为什么要这么做?  
因为在创建线程后去获取`mLooper`,此时线程的`run`方法可能还为运行,所以此时`mLooper`值应该为null;  
当运行了`Looper.prepare()`方法创建了`looper`后,通过`Looper.myLooper()`获取到`mLooper`,再`notifyAll`;  

# 二. Looper

# 2.1 Looper.prepare()

```
    public static void prepare() {
        prepare(true);                                    ①
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {                 ②
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));        ③
    }
```
> ①：无参情况下调用`prepare(true)`,形参置true表示允许退出。  
> ②： sThreadLocal 会先去获取本地的数据，如果能获取到说明已经prepare过，则抛出异常。  
> ③：设置sThreadLocal数据

sThreadLocal是ThreadLocal类型（static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();）  
ThreadLocal: 线程本地存储区，每个线程都有自己的私有本地存储区域，不同的线程之间彼此不能访问对方的存储区。

接下来看下刚保存的TLS区域的Looper对象：

```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);           ①
        mThread = Thread.currentThread();                 ②
    }
```
> ①：创建一个消息队列  
> ②：获取当前线程对象

这里为该线程创建了一个消息队列`MessageQueue`的构造函数中调用的hal层的本地方法：

```java
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
```

这个流程的分析先到这。


# 2.2 Looper.loop()

```
public static void loop() {
        final Looper me = myLooper();                      ①
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block     ②
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            msg.target.dispatchMessage(msg);               ③

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();                        ④
        }
    }
```

> ①: 获取TLS中存储的Looper对象  
> ②: 获取消息,没有消息的时候会阻塞(详见3.1)  
> ③: 分发消息(详见4.1)  
> ④: 回收消息到消息池(详见5.1)  



# 三. MesageQueue

# 3.1 next()

```
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);                                   ①

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());                       ②
                }
                if (msg != null) {
                    if (now < msg.when) {                                                 ③
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;                                           ④
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {                                                          ⑤
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {                 ⑥
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {                                       ⑦
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();                                             ⑧
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;                                                  ⑨

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

> ①:  调用本地epoll方法, 当没有消息时会阻塞在这,阻塞时间为nextPollTimeoutMillis(详见6.1.1)  
> ②:  查找消息队列中的异步消息(详见4.2)  
> ③:  如果当前时间小于异步消息的触发时间,则设置下一轮poll的超时时间(相当于休眠时间),否则返回将要执行的异步消息.  
> ④:  没有异步消息,下轮poll则无限等待,直到新的消息来临  
> ⑤:  检测下退出标志  
> ⑥:  如果消息队列未空或是第一个msg(消息刚放进队列且未达到触发时间),则执行空闲的handler  
> ⑦:  IdleHandler一个临时存放数组对象(下面可以看到一个列表转数组的方法被调用)  
> ⑧:  运行空闲的handler(只有第一次循环时会运行idle handle)  
> ⑨:  重置idle handler计数,防止下次运行  

往往在第一次进入next函数循环时,在`nativePollOnce`阻塞之后,都会执行idle handle函数.
获取到异步消息,立马把该消息返回给上一层,否则继续循环等待新的消息产生.


# 3.2  enqueueMessage()

```
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {                                                             ①
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {                                                                  ②
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w("MessageQueue", e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {                                    ③
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {                                         ④
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);                                                             ⑤
        }
    }
    return true;
    }
```

> ①: 判断该消息是否有handler,每个msg必须有个对应的handler;  
> ②: 判断该消息是否已经使用;  
> ③: 判断是否有已经准备好的消息(表头消息)或当前发送消息的延时时间为0或next ready msg延时时间大于当前消息延时时间则将当前消息变为新的表头.;  
>       根据判断当前阻塞标志,来觉得是否需要唤醒;  
> ④: 根据时间将消息插入到消息对列中;  
> ⑤: 上文分析在`next()`方法中会被阻塞,在这里就可以唤醒阻塞(详见6.1.2);  


# 四. Handler

# 4.1 消息分发

```
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);                                                          ①
        } else {
            if (mCallback != null) {                                                      ②
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);                                                           ③
        }
    }

```

> ①: 如果该msg设置了回调函数,则直接调用回调方法`message.callback.run()`;  
> ②: 当handler设置了回调函数,则回调方法`mCallback.handleMessage(msg)`;  
> ③: 调用handler自身的方法`handleMessage`,该方法默认为空,一般通过子类覆盖来完成具体的逻辑;  

我们Demo程序中,是使用第二种方法,设置回调来实现具体的逻辑,分发消息的本意是响应消息的对应的执行方法.  


# 4.2 消息发送
![](/images/handler/handler_sendmessag.png)

可以看到调用`sendMessage`方法后,最终调用的是`enqueueMessage`方法.  

```
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```
可以看到发送消息时都有一个时间参数选择,该参数就是我们前面分析的延时触发时间(相对时间).

```
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;                                                        ①
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;                                                                  ②
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
> ①: 判断handler创建时,传进来的消息对列是否为空(详见4.3)
> ②: 消息的`target`为该对象本身,handler类型  

这里有对发生的消息进行异步标志设置,通过判断`mAsynchronous`标志,该标志是在创建handler时初始化的(详见4.3);  
`Handler.enqueueMessage`方法调用的是`MessageQueue.enqueueMessage`方法(详见3.2);  


# 4.3 创建Handler

## 4.3.1 无参构造

```
    public Handler() {
        this(null, false);
    }

    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

无参构造方式比起我们Demo中的方式,它自己回调用` Looper.myLooper()`静态方法获取looper;


## 4.3.2 有参构造

```
    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);                                                    ①
    }
    
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
> ①: 调用有参构造函数创建`handler`且异步标志置`false`说明该`handler`发送的消息都为同步消息.

Demo中的handler就是使用该方式创建,自己传入`looper`参数.


# 五. Message


# 5.1 recycle()

```
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;                                                              ①
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {                                              ②
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```
> ①: 将该消息标志置为使用中并清除其他参数为default  
> ②: 将该消息加入消息池,当消息池未满时  

将消息回收到消息池都是将消息加入到消息池的链表表头.

# 5.2 obtain()

```
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;                                                        ①
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();                                                             ②
    }
```

> ①: 从消息池表头拿出一个消息  
> ②: 如果消息池为空则创建一个消息  

可以看出每次从消息池取出消息都是从链表的表头取出,再对消息的计数做减法.  



# 六. HAL层

native层本身也有一套完整的消息机制,用于处理native的消息;  
在整个消息机制中,`MessageQueue`是连接java层和native层的纽带;

# 6.1 MessageQueue

文件:  
```
android_os_MessageQueue.c
```

```
static JNINativeMethod gMessageQueueMethods[] = {
    /* name, signature, funcPtr */
    { "nativeInit", "()J", (void*)android_os_MessageQueue_nativeInit },
    { "nativeDestroy", "(J)V", (void*)android_os_MessageQueue_nativeDestroy },
    { "nativePollOnce", "(JI)V", (void*)android_os_MessageQueue_nativePollOnce },
    { "nativeWake", "(J)V", (void*)android_os_MessageQueue_nativeWake },
    { "nativeIsIdling", "(J)Z", (void*)android_os_MessageQueue_nativeIsIdling }
};
```

以上可以看出上层调用`nativePollOnce`方法实质是调用HAL层的`android_os_MessageQueue_nativePollOnce`方法


## 6.1.1 nativePollOnce

```
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jclass clazz,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, timeoutMillis);
}

void NativeMessageQueue::pollOnce(JNIEnv* env, int timeoutMillis) {
    mInCallback = true;
    mLooper->pollOnce(timeoutMillis);
    mInCallback = false;
    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

通过源码可以看出消息队列中的`pollOnce`实质是调用的`looper`中的`pollOnce`方法(详见6.2.1)

## 6.1.2 nativeWake

```
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    return nativeMessageQueue->wake();
}

void NativeMessageQueue::wake() {
    mLooper->wake();
}
```
通过源码可以看出消息队列中的`wake`实质是调用的`looper`中的`wake`方法(详见6.2.4)

## 6.1.3 nativeInit

```
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}


NativeMessageQueue::NativeMessageQueue() : mInCallback(false), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

可以看到hal层和java层中创建looper的时序几乎是一样的,先创建一个消息对列,再创建一个looper(Looper的构造详见6.2.3);  

## 6.1.4 nativeIsIdling

```
static jboolean android_os_MessageQueue_nativeIsIdling(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    return nativeMessageQueue->getLooper()->isIdling();
}

bool Looper::isIdling() const {
    return mIdling;
}
```

还是调用looper中的方法,来看看这个标志具体表示什么状态:

```
    // We are about to idle.
    mIdling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // No longer idling.
    mIdling = false;
```

以上代码片为`Looper::pollInner`中的一段,在wait时是空闲,当有数据来临时是非空闲的;  
以前也用过这样的方法来判断线程是否在使用,想不到在这里也看到了这种方法;  

## 6.1.5 nativeDestroy

```
static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->decStrong(env);
}
```


# 6.2 Looper

```
Looper.cpp: system/core/lib/libutils
```

## 6.2.1 Looper::pollOnce

```
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE
                ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
                        "fd=%d, events=0x%x, data=%p",
                        this, ident, fd, events, data);
#endif
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }

        if (result != 0) {
#if DEBUG_POLL_AND_WAKE
            ALOGD("%p ~ pollOnce - returning result %d", this, result);
#endif
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }

        result = pollInner(timeoutMillis);
    }
}
```

`Looper::pollOnce`是通过调用`Looper::pollInner`方法实现;  

## 6.2.2 Looper::pollInner

```
int Looper::pollInner(int timeoutMillis) {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - waiting: timeoutMillis=%d", this, timeoutMillis);
#endif

    // Adjust the timeout based on when the next message is due.
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - next message in %lldns, adjusted timeout: timeoutMillis=%d",
                this, mNextMessageUptime - now, timeoutMillis);
#endif
    }

    // Poll.
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    // We are about to idle.
    mIdling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);    ①

    // No longer idling.
    mIdling = false;

    // Acquire lock.
    mLock.lock();

    // Check for poll error.
    if (eventCount < 0) {                                                                   ②
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error, errno=%d", errno);
        result = POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    if (eventCount == 0) {                                                                  ③
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - timeout", this);
#endif
        result = POLL_TIMEOUT;
        goto Done;
    }

    // Handle all events.
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
#endif

    /* 处理epoll后的所有事件 */
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeReadPipeFd) {                                                        ④
            if (epollEvents & EPOLLIN) {
                    awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
Done: ;

    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();

#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
                ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
                        this, handler.get(), message.what);
#endif
                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // Release lock.
    mLock.unlock();

    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd);
            }
            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

> ①: 等待mEpollFd有事件产生,等待时间为timeoutMilli;  
> 当上层发消息时且判断需要唤醒,则会往管道的读端写入数据用于唤醒(详见6.2.3);  
> ②: 检测poll是否出错;  
> ③: 检测poll是否超时;  
> ④: 如果是因为往管道读端写入数据被唤醒,则都去并清空管道中的数据;  


## 6.2.3 Looper::Looper()

```
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    int wakeFds[2];
    int result = pipe(wakeFds);                                                              ①
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);

    mWakeReadPipeFd = wakeFds[0];
    mWakeWritePipeFd = wakeFds[1];
    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);                                    ②
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",
            errno);
    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",
            errno);

    mIdling = false;

    // Allocate the epoll instance and register the wake pipe.
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);

    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
    eventItem.events = EPOLLIN;                                                             ③
    eventItem.data.fd = mWakeReadPipeFd;
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);              ④
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",
            errno);
}

```
> ①: 创建一个无名管道, wakeFds[0]:读文件描述符, wakeFds[1]: 写文件描述符;  
> ②: 更改为无阻塞方式;  
> ③: EPOLLIN:连接到达,有数据来临;  
> ④: 监测管道读端是否有数据来临;  



## 6.2.4 Looper::wake()

```
void Looper::wake() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ wake", this);
#endif

    ssize_t nWrite;
    do {
        nWrite = write(mWakeWritePipeFd, "W", 1);
    } while (nWrite == -1 && errno == EINTR);

    if (nWrite != 1) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
```

唤醒只是向管道的写端写入一个字节数据,epoll_wait则会得到返回;  



# 总结

在这里做个总结针对java层(因为native层的消息机制未进行详细分析不过估计和java层的流程差不多);  
当调用j静态方法`Looper.prepare()`初始化后,再调用`Looper.loop()`方法进行消息循环处理;  
`Looper.loop()`方法中调用`MesageQueue.next()`方法检索新消息,没有则阻塞,有则将消息插入消息链表头后立即返回;  
阻塞方式是调用本地的`nativePollOnce()`方法实现,其原理是利用epoll管道文件描述符实现;  
`Looper.loop()`调用`dispatchMessage`方法实现消息的分发处理;  
发送一个消息的实质是调用个`MessageQueue.enqueueMessage()`方法往消息链表中插入一个消息,插入位置的条件为延时时间;  
然后再调用一个本地方法`nativeWake`对前面阻塞的进行唤醒,实质是往管道中写入一个字节数据;  

