---
title: Android Handler 源码分析
date: 2018-09-01 12:42:12
type: "Android"
---

## 前言

Handler 消息处理机制在 Android 开发中起着举足轻重的作用，有必要好好理解下其原理，先前我写了一篇文章，感觉疏漏了好多东西，下面先从一个简单的例子出发。<!--more-->

## 日常使用

假设有这么一个需要，请求网络然后将图片展示出来，我们知道网络请求是不允许在主线程执行的，而 UI 是不能在子线程（具体是不允许在非创建 UI 的原始线程）更新的，因此我们需要在子线程请求网络获得了数据以后再切换回主线程更新 UI ，这个例子中 Handler 就是起着切换线程的作用，下面的代码演示了这个例子。

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        loadImage()
    }
    private fun loadImage() {
        Thread {
            val url = URL("https://cn.bing.com/th?id=OHR.SiegeofCusco_ZH-CN9108219313_1920x1080.jpg&rf=LaDigue_1920x1080.jpg&pid=hp")
            val conn = url.openConnection()
            val bitmap = BitmapFactory.decodeStream(conn.inputStream)
            runOnUiThread {
                imageView.setImageBitmap(bitmap)
            }
        }.start()
    }
}
```

咦！说好的Handler去哪了？其实这里的 runOnUIThread 方法内部实现其实就是利用了 Handler，我们来看看它的源码。

```java
public final void runOnUiThread(Runnable action) {
    if (Thread.currentThread() != mUiThread) {
        mHandler.post(action);
    } else {
        action.run();
    }
}
```

该方法首先判断了当前线程是否是主线程，如果不是主线程就调用 `mHandler.post()  `，如果当前线程就是主线程就直接运行，下面我们来分析看看 Handler 的原理。

## Handler 原理

要想分析 Handler 的原理，我们先从 Handler 的创建过程开始分析。

### Handler 的创建

Activity 的这个 mHandler 是怎么来的呢？原来 mHandler 是 Activity 的成员变量，在 Activity 实例创建的时候就创建了。

```java
final Handler mHandler = new Handler();
```

接着看看 Handler 的构造方法

```java
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
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    // 代表发送的消息是否是异步的
    mAsynchronous = async;
}
```

首先如果 FIND_POTENTIAL_LEAKS 为 true 那么判断该 Handler 派生类是否是非静态内部类，如果是的话就打印出日志提示可能导致内存泄露，然后调用了 `Looper.myLooper` 获取到当前线程的 Looper 对象，如果当前线程没有 Looper 就会抛出异常，最后将 Looper 中的 MessageQueue 对象赋值给 ```mQueue```，callback 赋值给 ```mCallback``` ，aync 赋值给 ```mAsynchronous```，我们来看看 `Looper.myLooper` 做了些什么。

```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```
从 ThreadLocal 里面取 Looper ，那么是在哪里把 Looper 设置到 ThreadLocal 里面去的呢？其实 Looper 提供了 `prepare` 方法来创建当前线程的 Looper，我们来看看代码。

```java
public static void prepare() {
    // 表示允许退出循环
    prepare(true);
}
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

只有在当前线程拿不到 Looper 的时候才会去创建 Looper 对象并将其设置到 ThreadLocal 中去，不然就抛出异常说一个线程只能拥有一个 Looper，继续看看 Looper 的构造方法。

```java
// 这里的 quitAllowed 是 true
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
又创建了一个 MessageQueue 对象，继续看看它的构造方法。

```java
// 这里的 quitAllowed 是 true
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```

调用了一个 Native 方法就结束了，其实 mPtr 是 NativeMessageQueue 与 MessageQueue 之间的桥梁，暂时不看 native 层代码。

源码看到这里就会产生一个疑问，既然创建 Handler 的时候判断了当前线程的 Looper 是否为 null，为 null 就会抛出异常，那么 Activity 的 Handler 是怎么创建成功的呢？其实在 Activity 实例创建前主线程就已经有 Looper 对象了，这个得从 ActivityThread 开始说起。ActivityThread 是一个应用程序的入口里面有一个 main 方法，我们来看看。

```java
public static void main(String[] args) {
    ...
    Looper.prepareMainLooper();
    Looper.loop();
    ...
}
```

`Looper.loop() `后面会讲到先忽略，`main  `方法内部调用了 `Looper.prepareMainLooper()` 这个方法跟上面讲到的 `Looper.prepare()` 有什么异同点呢？我们来看看它的源码。

```java
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

`prepare `方法前面已经分析过了但是主线程是不允许退出的，所以传入了 false，后面判断了如果 sMainLooper 不为空那么就抛出异常，至此主线程的 Looper 创建成功这也就解释了为什么 Activity 中可以直接创建 Handler，接着我们分析那个 `post` 方法干了些什么事情。

### Handler 的消息发送

Handler提供了很多方法用于发送消息，比如以下几种：

* sendEmptyMessage(int what) 发送一个空消息，what 用于判断消息类别。
* sendEmptyMessageDelayed(int what, long delayMillis) 发送一个空消息，延迟 delayMillis 毫秒执行，what 用于判断消息类别。
* sendEmptyMessageAtTime(int what, long uptimeMillis) 发送一个空消息，在 uptimeMillis 的时候执行，what 用于判断消息类别。
* sendMessageDelayed(Message msg, long delayMillis) 发送一个消息，延迟 delayMillis 毫秒执行
* sendMessageAtTime(Message msg, long uptimeMillis) 发送一个消息，在 uptimeMillis 的时候执行
* sendMessageAtFrontOfQueue(Message msg) 发送一个消息，该消息会排在消息队列的队首
* executeOrSendMessage(Message msg) 如果 Handler 中的 Looper 与当前线程的 Looper 一致就直接分发消息，不然就发送一个消息。

继续着看 `post` 方法的实现

```java
public final boolean post(Runnable r) {
   return sendMessageDelayed(getPostMessage(r), 0);
}
```

其实`post`方法内部也就是发送了一个消息

```java
private static Message getPostMessage(Runnable r) {
    // message 内部维护了一个 Message 链表，以达到复用的目的，记得不要直接 new。
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

最终调用到了 ```sendMessageAtTime``` ，其实几乎所有发送消息的方法最终都会调用到该方法，继续看 `enqueueMessage` 的实现。

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

这里将本 Handler 的实例赋值给了 `msg.target` ，这个很重要以后会用到，然后判断下当前 Handler 是否是异步的，是的话就将消息设置成异步，我们这里不是异步的，接着继续看 `enqueueMessage` 。

```java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

该方法首先判断了 `msg.target` 是否为空，这个我们刚才看到已经设置了，然后判断 `msg` 是否正在被使用，然后再判断消息队列是否已经退出了，如果已经退出了就将 `msg` 回收并抛出个异常，下面那个同步代码块其实处理的逻辑就是将 `msg` 放入到消息队列中去，插入过程分为以下两步，至于 `needWake` 是用于判断是否要唤醒处于`nativePollOnce `而阻塞的 `Message.next` 方法：

- 如果满足 `p == null || when == 0 || when < p.when` 其实也就是如果消息队列的头指针为空，或者当前消息的执行时间为0，或者当前消息的执行时间先与消息队列队首的执行时间，那么将当前 `msg` 当做头指针。
- 如果不满足第一种情况就根据当前 `msg.when ` 决定插入的位置。

现在已经将消息放到的消息队列中，但是什么时候这个消息才能得到执行呢？这就要看看前面跳过的 ActivityThread 的 `main` 方法中的 `Looper.loop()` 。

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    boolean slowDeliveryDetected = false;
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            return;
        }
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        final long traceTag = me.mTraceTag;
        long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
        long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
        if (thresholdOverride > 0) {
            slowDispatchThresholdMs = thresholdOverride;
            slowDeliveryThresholdMs = thresholdOverride;
        }
        final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
        final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);
        final boolean needStartTime = logSlowDelivery || logSlowDispatch;
        final boolean needEndTime = logSlowDispatch;
        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
        final long dispatchEnd;
        try {
            msg.target.dispatchMessage(msg);
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        if (logSlowDelivery) {
            if (slowDeliveryDetected) {
                if ((dispatchStart - msg.when) <= 10) {
                    Slog.w(TAG, "Drained");
                    slowDeliveryDetected = false;
                }
            } else {
                if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                        msg)) {
                    // Once we write a slow delivery log, suppress until the queue drains.
                    slowDeliveryDetected = true;
                }
            }
        }
        if (logSlowDispatch) {
            showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
        }
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
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

这个方法代码有点长，主要流程如下

* 首先判断一下当前线程是否包含 Looper 不包含就抛出异常。
* 调用 MessageQueue 的 `next` 方法获取 Message，如果返回了null，标志了 MessageQueue 已经退出了，所以 Looper 也要退出。
* 获取 Looper 中的 `mLogging` 用于打印日志，我们可以通过 `setMessageLogging` 进行设置，设置后每次收到消息和消息处理完毕都会有日志我们可以根据这些日志分析 ANR 是由于处理哪个消息超时造成的。
* 设置慢分发时间和慢交付时间，可以通过 adb 进行设置，慢分发时间表示如果这个消息的实际执行时间比其设置的 `slowDeliveryThresholdMs` 要长就会打印警告日志，慢交付时间表示这个消息从消息队列取出时间比其设置的 `when` 超过 `slowDispatchThresholdMs` 就会打印警告日志。
* 记录开始分发的时间。
* 调用 `msg.target.dispatchMessage` 进行分发消息，其中 `msg.target` 就是一个 Handler 实例，上文说到过的。
* 记录结束分发的时间。
* 根据实际情况打印日志。
* 回收 `msg` 。

`loop  `方法调用 `queue.next` 取出消息，我们来看看该方法的实现

```java
Message next() {
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
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
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
            if (mQuitting) {
                dispose();
                return null;
            }
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler
            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        pendingIdleHandlerCount = 0;
        nextPollTimeoutMillis = 0;
    }
}
```

* 调用 `nativePollOnce` ，阻塞等待下一个可执行消息，参数为等待时间 -1 表示永久。
* 判断第一个消息的 `target` 是否为空，如果不为空表示是一个普通的消息，如果为空则表示是一个同步屏障消息(在屏幕刷新的时候会发送)，遍历消息队列找到第一个异步消息赋值给 `msg`。
* 判断 `msg` 是否为空，如果为空那么进行无超时的等待，直到被唤醒。
* 判断 `msg` 是否到了执行时间，如果不到就执行阻塞等待 `msg.when - now` ，如果已经到了就将该消息返回
* 如果消息队列已经退出就返回 null。

**总结下 Looper.loop 的原理，其实就是分为以下步骤：**

1. Looper.loop 死循环执行 MessageQueue.next 。
2. MessageQueue.next 首先 blocked  为 false 、nextPollTimeoutMillis 为 0，于是立即脱离阻塞，获取 Message 链表中的第一个消息，这里分情况讨论。
    * 获取不到 Message ，那么 blocked 为 true ，nextPollTimeoutMills 为 -1，无限制的阻塞在 nativePollOnce，等待 MessageQueue.enqueueMessage 将它唤醒。
    * 获取到 Message ，比较当前时间与 Message.when 如果发现还没到执行时间，那么 blocked 为 true，nextPollTimeoutMillis 为 当前时间 - Message.when，在 nativePollOnce 那里等待一会儿。
    * 获取到 Message，比较当前时间与 Message.when 如果发现到了执行时间，那么直接将该 Message 返回给 Looper.next ，其执行 Message.target.dispatchMessage。执行完毕后再次回到第 2 步。
3. MessageQueue.enqueueMessage

    * Message 需要插入到链首，是否唤醒由 ```blocked``` 字段决定，如果当前其为 false，那么表示其还没阻塞，它自己会从队列中取，不需要唤醒，如果其为 true ，表示其由于 链表无元素或者链表首部元素执行时间未到，因为当前 Message 是插入首部的，于是得唤醒，判断当前 Message 是否可以执行。

    * Message 需要插入到链表中间，如果不考虑同步屏障，那么是不需要唤醒的。

拿到了消息以后就调用了 `handler.dispatchMessage` 我们来看看其实现

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

首先判断是否该 Message 是否设置了 callBack，设置了就直接运行，然后判断 Handler 是否设置了 callBack，设置了就调用 `callback.handleMessage` 如果返回 false，继续调用 `handleMessage` 。

## IdleHandler 用法

先说说作用，其可以在 Looper 所在线程空闲时执行某些操作来提高性能，不过单次执行不要太耗时，不要后续事件可能被延迟了。

```java
public void addIdleHandler(@NonNull IdleHandler handler) {
    if (handler == null) {
        throw new NullPointerException("Can't add a null IdleHandler");
    }
    synchronized (this) {
        mIdleHandlers.add(handler);
    }
}
public void removeIdleHandler(@NonNull IdleHandler handler) {
    synchronized (this) {
        mIdleHandlers.remove(handler);
    }
}
Message next() {
    for (;;) {
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            ...
            if (pendingIdleHandlerCount < 0
                && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; 
            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        pendingIdleHandlerCount = 0;
        nextPollTimeoutMillis = 0;
    }
}
```

从上述代码可以看到，如当前 MessageQueue 为空 或者 MessageQueue 的首个 Message 的执行事件还没到，本来这两种情况都需要 nativePollOnce 阻塞等待的，如果有 IdleHandler 那么会依次其 queuIdle 方法，如果返回 false，那么会自动移除掉，注意：**执行完一次 IdleHandler 后，需要等到下次再调用 MessageQueue.next 才有可能再次执行**。

一般也就是用于 Activity 启动优化将 onCreate，onStart，onResume 中耗时较短但非必要的代码可以放到 IdleHandler 中执行，减少启动时间。

## 总结

- Handler 的作用就是把想要执行的操作从当前线程切换到 Handler 中 Looper 所在的线程进行执行。
- 每次通过 Handler 发送消息其实就是把消息插入到了消息队列中，然后根据情况判断是否要唤醒处于调用 `nativePollOnce` 阻塞状态的线程。

