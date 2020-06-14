---
title: Android 四大组件之 BroadcastReceiver 篇
date: 2018-04-13 11:45:53
type: "Android"
---

## 前言

上篇文章复习总结了 Service 的相关知识，现在开始复习 BroadcastReceiver。BroadcastReceiver 能够接受系统或者其他 App 发来的特定广播，本文先从广播的发送开始。<!-- more -->

## 两种发送方式

### sendOrderdBroadcast(Intent, String)

发送有序广播，优先级（通过 `android:priority` 或者 `setPriority` 设置）高的 BroadcastReceiver 会被优先收到（如果优先级相同，则动态注册的优先级高于静态注册的），其可以改变广播传递的内容或者直接中止该广播，通过调用 `abortBroadcast `。

### sendBroadcast(Intent, String)

发送无序广播，忽略广播接受者的优先级（因此无法被中止），所有广播接受者按照一种不确定的顺序接收到。

## 两种注册方式

### 静态注册

只需要新建一个类继承 BroadcastReceiver ，然后在清单文件 application 节点下，配置 receiver 节点即可，例如下面这段代码就配置了一个可用于接受飞行模式开关状态的 BroadcastReceiver。

```xml
<receiver android:name=".MyBroadcastReceiver">
    <intent-filter>
        <action android:name="android.intent.action.AIRPLANE_MODE" />
    </intent-filter>
</receiver>
```

静态注册的 BroadcastReceiver 只要应用程序进程没被杀死就能接受到对应的广播，但是其没法接受某些系统的广播例如屏幕打开关闭、电量改变等。

### 动态注册

动态注册首先新建一个类继承 BroadcastReceiver 的实例，然后再创建一个 IntentFilter 实例设置该BroadcastReceiver 关注哪几种广播，最后调用 `registerReceiver` 注册广播，例如以下代码就动态注册了一个BroadcastReceiver。

```kotlin
fun registerBroadcastReceiver() {
    val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION).apply {
    	addAction(Intent.ACTION_AIRPLANE_MODE_CHANGED)
    }
    registerReceiver(br, filter)
}
```

动态注册可以接受到静态注册无法接收到广播例如屏幕打开关闭、电量改变等，此外动态注册的广播我们在用完以后应该调用 `unregisterReceiver` 解除注册，虽然系统也会帮我们解除，但是会打印错误。

### 使用 BroadcastReceiver 的注意点

虽然系统规定 BroadcastReceiver 的 `onReceive` 最多执行 10 秒，但是由于该方法运行于主线程所以其实超过5秒就会 ANR，如果我们要在 onReceiver 执行耗时任务里面就会想到开一个线程执行，但是当 `onReceive` 调用完毕后系统就会去销毁该实例，这样那个新建的线程优先级就非常低了很容易被杀死，导致达不到我们要效果。有以下两种方法可以用于解决这种情况。

1. 通过在 `onReceive` 中调用 `goAsync` 获得 PendingResult 对象以后再开启子线程，在执行结束时调用PendingResult.finish，不过只能运行 10 秒。
2. 通过在 `onReceive` 中启动服务（比如IntentService）运行耗时任务。

### 本地广播和粘性广播

这两种广播都可以使用开源库 [EventBus]("https://github.com/greenrobot/EventBus") 代替。

