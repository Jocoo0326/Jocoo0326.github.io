---
layout: post
title: Android Looper Message Handler
date: 2015-07-21
categories:
- android
tags:
- Handler
---

由于android中线程完成后需要在主线程中更新UI, 就不得不使用Handler, 现结合源码分析其实现
<!-- more -->
Handler原理总结:
* 1、android使用Looper实现线程中的消息循环，Looper是以线程为作用域，且每个线程的Looper各不相同，Android使用ThreadLocal（线程局部存储机制，为线程存储只和本线程有关的变量，每个线程所存储的值互不干扰）实现了Thread与Looper的关联；

* 2、使用Looper.prepare方法创建Looper，创建Looper时会创建一个MessageQueue消息队列；

``` java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

``` java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

* 3、在Activity的UI线程中执行的ActivityThread（实际并非线程）的main函数中，也创建了一个Looper，即sMainLooper，调用了Looper.prepareMainLooper方法，开发中可以使用Looper.myLooper()
== Looper.getMainLooper()判断是否是UI线程；

ActivityThread#main(Strings[] args)
``` java
public static void main(String[] args) {
    ...
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    AsyncTask.init();

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

* 4、创建Handler时判断当前线程是否存在Looper，若没有会抛出异常；
``` java
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

* 5、调用Looper.loop()方法启动消息循环，它会启动一个死循环，从MessageQueue中取出消息将消息分发（dispatchMessage）到消息的目标对象（msg.target，其实是发送这条消息的handler）中去，这里的dispatchMessage方法是在创建Looper、Handler的线程中执行的，这样就将其他线程发送的消息切换到目标线程中执行了；
``` java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        Message msg = queue.next(); // might block
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

        msg.target.dispatchMessage(msg);

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

        msg.recycleUnchecked();
    }
}
```

* 6、ActivityThread中的消息循环：ActivityThread中有一个内部类ApplicationThread，ActivityThread通过它与ActivityManagerService进行IPC，AMS调用ApplicationThread中的方法向ActivityThread中的Handler即H对象发送消息，完成Activity的生命周期的处理；

* 7、Looper.loop()为何不会卡死主线程？
Looper.loop()->MessageQueue.next()->MessageQueue.nativePollOnce()->Looper::pollInner()(cpp)->epoll_wait
利用了pipe/epoll机制，当MessageQueue.enqueMessage时若处于block状态会调用nativeWake->Looper::wake->write data to eventFd->epoll_wait unblocked

* 8、MessageQueue.postSyncBarrier 当设置了障碍后，队列中的同步消息会停止执行，直到removeSyncBarrier被调用，默认情况下，消息都是同步的
