---
title: Android View 动画 xml 篇
date: 2018-05-13 15:16:32
type: "Android"
---

## 前言

Android 中 View 动画对应的 xml 文件放于 res/anim 中，其根节点共有以下五种，alpha、scale、translate、rotate、set，里面的值可以直接使用 float，也可以使用 %，或者 %p。<!-- more --> 

## 公共属性

1. android:detachWallpaper 特殊选项窗口动画，如果这个窗口显示在壁纸上不要使用该动画。
2. android:duration 动画执行时间，单位 ms。
3. android:fillAfter 如果设置为 true，动画结束后，UI 保留在最后一帧。
4. android:interpolator 给动画设置插补器，android 提供了多种插补器， @android:anim/linear_interpolator、@android:anim/accelerate_decelerate_interpolator等，也可以在 res/anim中 自定义插补器，android 提供的插补器在结尾说明。
5. android:repeatCount 重复次数 1表示执行两次，0 表示执行一次，-1 无限循环。
6. android:android:repeatMode repeat | restart(default)。
7. android:startOffset 调用 view.startAnimation 后多少 ms 开始执行。

## alpha

alpha 渐入渐出动画对应于 AlphaAnimation.java，其拥有如下属性：

1. android:fromAlpha 从哪个alpha值开始 0.0 表示全透明，1.0 表示全不透明。
2. android:toAlpha 目标 alpha 值。

## scale

scale 缩放动画对应于 ScaleAnimation.java，其拥有如下属性：

1. android:fromXScale 起始 X 缩放比。
2. android:toXScale 结束 X 缩放比。
3. android:fromYScale 起始 Y 缩放比。
4. android:toYScale 结束 Y 缩放比。
5. android:pivotX X 轴缩放中心点，如果为0，则向右缩放。
6. android:pivotY Y 轴缩放中心点，如果为0，则向下缩放。

## translate

 translate 缩放动画对应于 TranslateAnimation.java，其拥有如下属性：

1. android:fromXDelta 起始位置的 X 轴的偏移量。
2. android:toXDelta 结束位置的 X 轴的偏移量。
3. android:fromYDelta 起始位置的 Y 轴的偏移量。
4. android:toYDelta 结束位置的 Y 轴的偏移量。

## rotate

rotate 旋转动画对应于 RotateAnimation.java，其拥有如下属性：

1. android:fromDegrees 起始的旋转角度。
2. android:toDegrees 结束的旋转角度。
3. android:pivotX 旋转的 X 轴中心点。
4. android:pivotY 旋转的 Y 轴中心点。

## set

set 组合动画对应于 AnimationSet.java，给 set 设置属性会覆盖内部的属性，其拥有如下属性：

1. android:shareInterpolator 如果设置为 true 插补器会共享给所有子元素。

## Android 提供的差值器

1. AccelerateDecelerateInterpolator(default) 先慢中间快后慢。
2. AccelerateInterpolator 加速运动。
    - android:factor 加速因子，默认1，该值越大最大速度越大，持续时间一致，所以初始速度回很小。
3. AnticipateInterpolator 先往反方向运动一段距离再前进。
    - android:tension 张力， 默认2，该值越大往反方向运动距离越大。
4. AnticipateOvershootInterpolator 先往反方向运动再前进并且超过目标点，最后回到目标点。
    - android:tension 张力， 默认2，该值越大往反方向运动距离越大。
    - android:extraTension 额外的张力， 默认1.5，该值越大超过目标点距离越大。
5. BounceInterpolator 类似球落地时的反弹效果。
6. CycleInterpolator 循环，先往正向运动到终点，然后向反方向运动知道运动到-终点处再回来。
    - android:cycles 循环次数，默认值为1。
7. DecelerateInterpolator 减速运动。
    - android:factor 减速因子，默认为1，越大速度变化越不明显。
8. LinearInterpolator 匀速运动。
9. OvershootInterpolator 移动超过目标点，然后再回来。
    - android:tension 张力， 默认2，该值越大超过目标点距离越大。