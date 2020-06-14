---
title: Android IntentService 源码分析
date: 2018-04-12 13:43:23
type: "Android"

---

## 前言

由于 Service 中的各个回调方法都是运行在主线程的所以没法做耗时任务，不然会导致 ANR，因此 IntentService 就因运而生了。<!-- more -->

> IntentService 是 Service 的子类，客户端通过调用 context.startService(Intent) 发送请求，它根据需要启动，在工作线程中处理每个 Intent ，执行完所有任务后关闭自己。

## IntentService.onCreate

代码如下：

```java
public void onCreate() {
    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]"); // 1
    thread.start(); // 2
    mServiceLooper = thread.getLooper(); // 3
    mServiceHandler = new ServiceHandler(mServiceLooper); // 4
}
```

1. 创建了一个 HandlerThread 实例。
2. 启动了该线程。
3. 获取 HandlerThread 在 run 方法中创建的 Looper 对象。
4. 使用该 Looper 对象创建了 ServiceHandler 对象。

里面用到了 HandlerThread ，那么就先来看看 HandlerThread ，其继承了 Thread，其 run 方法如下所示。

```java
public void run() {
    mTid = Process.myTid(); //1
    Looper.prepare(); //2
    synchronized (this) {
        mLooper = Looper.myLooper(); //3
        notifyAll(); //4
    }
    Process.setThreadPriority(mPriority); //5
    onLooperPrepared();//6
    Looper.loop();//7
    mTid = -1;
}
```

1. 获取到了当前线程的 tid 将其赋值给 mTid。
2. 调用 `Looper.prepare` 为当前线程创建了一个 Looper 对象。
3. 获取当前线程的 Looper 对象并将其赋值给 mLooper。
4. 唤醒 `getLooper` 方法中调用 wait 而阻塞的线程。
5. 设置线程优先级。
6. 死循环不停的从消息队列中取数据。

再来看看其 `getLooper` 方法

```java
public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }
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
```

* 如果线程已经死了，那么返回 null 。
* 线程活着但是 mLooper 还等于 null，就调用 wait() 将锁让给运行在子线程的 run 方法，让其创建 Looper。
* 线程活着并且 mLooper 不等于null，那么就返回这个子线程的 Looper 对象。

思考：这里有个问题，为什么 `getLooper` 里面需要使用 while 循环？照道理当调用 `getLooper` 的线程被唤醒的时候 `mLooper = Looper.myLooper();` 已经被执行了，而 synchronized 又能够提供可见性，强制调用 `getLooper` 的线程去主存里面读取 mLooper 值。猜测是为了防止被唤醒时线程已经死了仍然返回 Looper 对象。

## IntentService.onStart

代码如下：

```java
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}
```

这里将接收到 intent 和 startId 组装成消息发送给 ServiceHandler ，来看看 ServiceHandler 的 `handleMessage` 方法。

```java
public void handleMessage(Message msg) {
    onHandleIntent((Intent)msg.obj);
    stopSelf(msg.arg1);
}
```

注意由于该Handler创建时传入的 Looper 是在子线程创建的，所以 `handleMessage` 方法是运行在子线程的，而 `onHandleIntent` 是个抽象方法由子类实现当其调用完后会调用 `stopSelf(startId)` 该方法会判断给定的 startId 是不是最近一次启动的 startId，如果不是那么什么都不做，如果是那么就会关闭服务。   

## IntentService.onDestroy

代码如下：

```Java
public void onDestroy() {
    mServiceLooper.quit();
}
```

调用了 Looper.quit ，那么 HandlerThread 的 run 方法退出 Looper.loop 阻塞继续向下执行 `mTid = -1;` 然后HandlerThread 线程就执行完毕了。

## 总结

IntentService的原理其实就是在子线程创建了一个 Looper ，然后根据该 Looper 创建一个 Handler，当每次调用 `context.startService` 的时候调用 `handler.sendMessage` 在子线程中完成工作，完成以后看看有没有新任务到来，如果没有就把自己关闭，否则就继续执行下一个任务。