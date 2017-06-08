# 第10章 Android的消息机制

Handler可以将一个任务切换到Handler所在的线程去执行。

Handler的运行需要底层的MessageQueue和Looper的支持。

MessageQueue是一个单链表的数据结构，Looper会以无限循环的形式去查找是否有新消息，如果有的话就处理消息，否则就一直等待。线程默认是没有Looper的，如果需要使用Handler就必须为线程创建Looper。我们经常提到的主线程，也叫UI线程，它就是ActivityThread。

## 10.1 Android的消息机制概述

ViewRootImple对UiI操作做了验证：

```java
void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
        "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```

系统为什么不允许在子线程访问UI？因为Android的UI控件不是线程安全的。为什么不对UI控件的访问加上锁机制？首先加上锁机制会让UI访问的逻辑变得复杂；其次锁机制会降低UI访问的效率，因为锁机制会阻塞某些线程的执行。

## 10.2 Android的消息机制分析

### 10.2.1 ThreadLocal的工作原理

ThreadLocal是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储后，只有在指定线程中可以获取到存储的数。

例子：

```java
private ThreadLocal<Boolean> mBooleanThreadLocal = new ThreadLocal<>();

mBooleanThreadLocal.set(true);
System.out.println(Thread.currentThread()+":"+mBooleanThreadLocal.get());
new Thread("Thread#1"){
    @Override
    public void run() {
        mBooleanThreadLocal.set(false);
        System.out.println(Thread.currentThread()+":"+mBooleanThreadLocal.get());
    }
}.start();

new Thread("Thread#2"){
    @Override
    public void run() {
        System.out.println(Thread.currentThread()+":"+mBooleanThreadLocal.get());
    }
}.start();
```

结果为：
Thread#main:true
Thread#1:false
Thread#2:null

Looper类中有个变量为每个线程提供Looper对象:

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

### 10.2.2 消息对列的工作原理

MessageQueue主要包含两个操作：插入和读取。enqueueMessage的作用是往消息对列中插入一条消息，而next的作用是从消息对列中取出一条消息并将其从消息对列中移除。

都是对链表的插入和删除操作，对next方法说明下：

```java
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

    nativePollOnce(ptr, nextPollTimeoutMillis);

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
            } while (msg != null && !msg.isAsynchronous());
    }
    if (msg != null) {
        if (now < msg.when) {
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
            if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                msg.markInUse();
                return msg;
        }
    } else {
        // No more messages.
        nextPollTimeoutMillis = -1;
    }

    // Process the quit message now that all pending messages have been handled.
    if (mQuitting) {
        dispose();
        return null;
    }
    ...
}
```

读取消息是在一个无限循环里，若消息对列中无消息，则一直循环处于阻塞状态。正常情况，消息是从对列头部开始处理，除了消息没有指定target的情况下会去寻找待处理的异步消息。只有消息对列不为空或者处于退出状态才会结束阻塞状态。

### 10.2.3 Looper的工作原理

Handler的工作需要Looper，没有Looper的线程会报错，如何创建Looper？

 1. Looper.prepare()为当前线程创建一个Looper
 2. Looper.loop()开启消息循环

Looper除了prepare()方法外，还提供了prepareMainLooper方法，这个方法主要是给主线程(ActivityThread)创建Looper使用的。Looper提供了quit和quitSafetly来退出一个Looper，二者的区别是：quit会直接退出Looper，而quitSafetly只是设定一个退出标记，然后把消息对列中的已有消息处理完后才安全退出。

### 10.2.4 Handler的工作原理

Handler的工作主要包含消息的发送和接收。消息的发送可以通过post的一系列方法以及send的一系列方法实现。

dispatchMessage方法是Handler处理消息的阶段:

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

一般我们都是继承Handler重写handlerMessage方法，从上面可以看到也可以mCallback来处理消息。通过Hanler的有参构造函数来实例化Handler。

```java
 public Handler(Callback callback, boolean async)
```

<font color="#FF0000">最后提出一个问题，Looper.loop()方法中有无线循环为什么主线程不会 阻塞？</font>和Android四大组件的工作过程有关。
