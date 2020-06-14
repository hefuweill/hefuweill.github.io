---
title: Android 性能优化
date: 2018-04-26 14:45:53
type: "Android"
---



## 前言

性能优化能够加快程序启动时间，降低应用卡顿，减少耗电量、流量，降低安装包体积等，本文主要记录布局优化、内存优化、APK 包优化。<!-- more -->

## 布局优化

1. 布局文件中层级不能过多，过多会导致遍历时间成几何倍增长，考虑使用 ConstraintLayout 该布局能做到只有一层布局层次。
2. 能减少 View 的数量尽量减少，比如 android studio 提示一个图片一个文字，最好就使用一个 TextView。
3. 使用 include 复用布局、merge 减少不必要的 ViewGroup、使用 ViewStub 在需要显示的时候再加载。
4. 打开开发者选项观看 GPU 渲染速度，中间的绿线表示一帧耗时 16mm(一秒60帧) 。
5. 打开开发者选择观看 GPU 过度绘制，减少没必要的背景色。

## 内存优化

内存优化包括：内存泄漏、内存溢出、内存抖动、选用合适的容器 四大类。

### 内存泄漏

1. 静态变量导致的内存泄漏，比如将 Actvity、Fragment 引用赋值给静态变量，由于静态变量生命周期和 APP 生命周期一致，所以即使 Activity 被关闭了，也无法被回收，所以不用时要将其置为 null。
2. 单例对象持有 Activity、Fragment 等引用，这与静态变量持有一样。
3. 非静态内部类拥有一个外部类实例，如果在 Activity 内部新建了一个非静态 Handler 派生类，然后给该 Handler 发送延时消息，这时候如果 Activity 被关闭，由于在发送消息时会将 Message.target 设置为当前 Handler 对象，并且 Message 会被放入当前线程的 Looper 的 MessageQueue 中，因此无法回收，导致内存泄漏。引用链  MessageQueue - Message - Handler - Activity，解决方法是采用静态加弱引用 。 再比如 Activity 中新建一个非静态 Thread 派生类，Activity 销毁了它还在运行也是一样。

4. 未取消注册动态广播接收者或者未取消注册监听器。
5. 资源对象未关闭，IO 流、Cursor、Bitmap 等。

使用 LeakCanary 进行内存泄漏的检测，当然还有 MAT 等工具。

### 内存溢出

加载大图片非常可能会导致 OOM ，首先需要做到按需加载，方法是：

1. BitmapFactory.Options.inJustDecodeBounds 设置为 false。
2. 加载图片获取到图片的宽高，根据宽高与 View 的宽高计算缩放比。
3. BitmapFactory.Options.inSampleSize 设置为缩放比。
4. BitmapFactory.Options.inJustDecodeBounds 设置为 true。
5. 重新加载获取缩放后的 Bitmap。

其次可以选择合适的解码方式，ARGB 4444 占用 2 个字节（4 + 4 + 4 + 4 ）位，ARGB 8888 占用 4个字节（8 + 8 + 8 + 8）位。

### 内存抖动

避免创建大量、临时的小对象，防止频发的进行 GC ，因为不管是标记清除还是暂停拷贝都需要暂停，这可能导致卡顿。

1. 避免在 View.onDraw 中创建对象。
2. 避免在循环体内进行字符串的拼接。
3. 如果对象可以复用考虑复用，比如 Message、ByteArrayPool 等等。

### 容器

考虑使用 ArrayMap 以及 SparseArray。

#### ArrayMap

ArrayMap 采取两个数组存储，第一个数组存储了所有的 hashCode。第二个数组存储键值对（第 0 个元素存储键，第 1 个元素存储值，第 2 个元素存储键以此类推），这样当插入元素时避免了创建 Node 对象，并且扩容也只需要进行数组拷贝，不需要重建 Hash 表，此外当元素过少时还可以实现缩容。

实现原理：

每次调用 put 时先使用二分搜索在第一个数组中寻找 key 对应的 hashCode，如果没有找到那么表示元素不存在，需要插入元素。如果找到该 hashCode 在第二个数组中对应的元素位置，进行修改。不过如果存在 hash 冲突那么就没法根据第一个数组的下标找到第二个数组中对应元素的下标，需要进行遍历从 (index+1) << 1 ~ size - 1 以及 (index - 1) ~ 0 寻找对的上 key 的元素。

考虑当元素数量少于 1000 个，并且 Key 不是基本数据类型，可以考虑使用，节省一点内存空间。

#### SparseArray

SparseArray 采用稀疏数组进行实现，保证了内存利用率，不像 HashMap 可能会存在数组元素为空的情况。同时它还避免了自动装箱。

考虑当元素数量少于 1000 个，并且 Key 为基本数据类型时，直接使用 SparseArray，节省内存空间，以及减少装箱时间。

