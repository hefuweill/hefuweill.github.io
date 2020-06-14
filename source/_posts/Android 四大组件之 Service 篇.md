---
title: Android 四大组件之 Service 篇
date: 2018-04-12 11:45:53
type: "Android"
---

## 前言

上篇文章复习总结了 Android 的启动模式，现在开始复习 Service 相关的知识，Service 是一种可以在后台执行长时间运行操作而没有用户界面的应用组件。首先从 Service 的生命周期开始。<!-- more --> 

## Service 的生命周期

![](生命周期.png)

* `onCreate()`  当实例创建时调用，一个实例只会被调用一次该方法
* `onBind(Intent)`  当通过 bindService 启动服务时，会调用该方法，该方法会返回一个 IBinder 实例，一个 Service 实例永远只会调用一次。
* `onStartCommand()`  当通过 startService 启动服务时，会调用该方法，不停的调用 startService 会重复调用该方法。
* `onUnbind(Intent)`  当所有的连接都断开后会回调该方法，默认该方法返回 false，如果返回  true，再有新的连接会回调  `onRebind()`。
* `onRebind(Intent)` 当 onUnbind 返回 true 后如果服务还在运行，这时又调用 bindService 会回调该方法，如果返回 false 服务还在运行，那么再调用 bindService 会无反应。
* `onDestory()` 当 Service 将要被销毁时调用，用于资源的回收。

## Service 的启动方式

### startService(Intent)

单独通过该方式启动服务，当目标服务还没运行会调用服务的 `onCreate `、 `onStartCommand `，如果已经运行了就只会调用 `onStartCommand` ，当服务开启后就与启动方没有任何关系了，只有当自己调用 `stopSelf`，或者外界调用 `stopService ` 才会停止，并且无论启动多少次都只需要停止一次就行了。

* **例一** Activity A 连续调用 `startService` 10次启动 Service S，则最后 S 会调用1次 `onCreate`、10次 `onStartCommand`，并且当 A 被退出后 S 会继续运行。

### BindService(ServiceConnection)

单独通过该方式启动服务拥有以下特性

* 当目标服务还没运行会调用服务的 `onCreate` 、 `onBind` ，如果已经运行了就只会调用 `onBind` ，重复 `bindService` 不会重复调用 `onBind`。

* 我们可以在 `onServiceConnected` 获取到一个 IBinder 对象(注意 Service 的 `onBind` 方法返回的**必须不是 null**，不然不会调用该方法)，如果 Service 与启动的 Activity 同属于一个进程那么返回的就是 Service 的 `onBind` 返回的那个对象，不然返回的是一个 BinderProxy 对象，如果是相同进程我们可以直接将其强转然后调用其方法。不同进程则需要通过AIDL生成的类将其上转型成一个接口，然后才能调用。
* 当通过该方式启动它的 Activity 退出时，如果没有手动调用 `unBindService`，系统会给我们调用该方法并打印出一段错误，当 Service 的所有的连接都断开了以后会调用 Service 的 `onUnbind` ，然后调用 `onDestroy`。

### 结合调用 bindService 和 startService

* 先调用 `startService` 然后调用 `bindService` ，会调用Service的 `onCreate` 、 `onStartCommand` 、 `onBind` 。
* 先调用 `bindService` 然后调用 `startService`， 会调用 Service 的 `onCreate` 、`onBind `、 `onStartCommand`。

当 Service 的所有连接都断了并且调用了 `stopService`，这种混合启动的 Service 才会被销毁。