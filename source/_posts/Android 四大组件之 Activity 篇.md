---
title: Android 四大组件之 Activity 篇
date: 2018-04-10 11:45:53
type: "Android"
---

## 前言

以前学习的 Android 知识杂乱无章，形成不了一个完整的知识体系，因此打算根据知识星球的内容，完整的过一遍，首先来复习一下 Activity。Activity 是 Android 四大组件中最为重要的一个，负责直接与用户交互，先来看看 Activity 的生命周期。 <!-- more --> 

## 生命周期

 官方文档给出了下面这个图

![生命周期](生命周期.png)

1. `onCreate(Bundle)`  是Activity中除了 `attachBaseContext` 以外的第一个回调方法，该方法一般用于调用 `setContentView`，如果在该方法内直接调用 ```finish``` 那么该方法执行完后会直接执行 ```onDestroy``` 。
2. `onStart()`  当Activity可见时回调，注意**可见不代表 Activity 的 UI 被显示出来**，因为 View 的三大流程需要在 onResume 回调完后才会开始。
3. `onResume()` 当 Activity 获得了焦点后回调（开始可以与用户交互），同样的，Activity 的 UI 也没被显示出来，因此在该方法中直接获取 View 的宽高拿到的都是0，回调完该方法 Activity 就处于运行状态了。
4. `onPause()` 当 Activity 失去了焦点后回调，该方法后面可能会调用 onStop、onResume 或者可能被系统杀死。
5. `onStop()`  当 Activity 不可见时回调，该方法后面可能会调用 `onDestory`、`onRestart` 或者可能被系统杀死
6. `onRestart()`  当 Activity 处于 stop 状态后，又被切换到的前台时回调，该方法后面紧跟着 ```onStart``` 回调。
7. `onDestroy() ` 当Activity 将要被销毁后调用，可能的情况是 `finish` 方法被调用或者配置改变时(没有配置configChange)时回调，该回调中可以做一些资源回收等操作

重点

* 当 Activity A 启动一个透明的 Activity 或者一个 Dialog 主题的 Activity 时，A 的 onStop 不会调用，因为其还可见。
* 当 Activity A 开启一个 Dialog，不会调用 A 的任何生命周期方法。
* 当 Activity A 开启 Activity B 会调用 A.onPause - B.onCreate - B.onStart - B.Resume - A.onStop （视情况，当A完全不可见时会调用）。
* 当 Activity A 启动 Activity B 然后点击 back 键会回调 B.onPause - （如果 A 处于 stop 状态还有 A.onRestart - A.start）A.onResume - B.onStop - B.onDestroy。
* 当 Activity A 失去焦点后，当内存不足时系统可能会将 A 杀死，当点击回退后会调用A的 onCreate - onStart - onResume 进行重建。

## Fragment 的生命周期与 Activity 的关系

首先来看看Fragment的生命周期

1. `onAttach(Activity)` 当 Fragment 与 Activity 建立联系时调用。
2. `onCreate(Bundle)` 当初始化创建 Fragment 的时候回调。
3. `onCreateView(LayoutInflate, ViewGroup, Bundle)` 创建和返回 Fragment 显示的根 View。
4. ```onViewCreated(View)``` 在 onCreateView 执行完毕后就调用。 
5. `onActivityCreated(Bundle)` 当与 Fragment 相联系的 Activity 完成了 onCreate 回调。
6. `onStart()` 当 Fragment 可见时回调。
7. `onResume()` 当 Fragment 获取焦点时回调。
8. `onPause()` 当 Fragment 失去焦点时回调。
9. `onStop()` 当 Fragment 不再可见时回调。
10. `onDestoryView()` 用于清除与 Fragment 相关联 View 的资源。
11. `onDestory()` 当 Fragment 将要被销毁时调用。
12. `onDetach()` 当调用该方法后 Fragment 就与 Activity 失去联系了。

假设有 Activity A 在 onCreate 通过 replace 方法显示了 Fragment F，生命周期调用顺序为 A.onCreate - A.onStart - F.onAttach - F.onCreate - F.onCreateView - F.onViewCreated - F.onActivityCreated - F.onStart - A.onResume - F.onResume。

## Activity 与 menu 创建先后顺序

在 Activity 的  `onResume`  调用完后会回调  `onCreateOptionMenu`  来创建 Menu。

## onSavedInstanceState() 和 onRestoreInstanceState()

* `onSavedInstanceState`  当 Activity 被意外的杀死或者是当配置方式改变后会回调 onSavedInstanceState 在 **API28 及以上**其调用时机在 onStop 之后，在 API28 之前调用时机 onStop 之前与 onPause 无时序关系。

* `onRestoreInstanceState` 再次启动后我们可以在 onRestoreInstanceState 中恢复 onSavedInstanceState 保存的数据。

