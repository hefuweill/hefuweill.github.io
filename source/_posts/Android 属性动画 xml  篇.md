---
title: Android 属性动画 xml 篇
date: 2018-05-14 14:36:26
type: "Android"
---

## 前言

Android 中属性动画对应的 xml 文件放于 res/animator 中，其根节点共有下列三种，objectanimator、animator、set。<!-- more --> 

## objectanimator

objectanimator 对应 ObjectAnimator.java，其拥有如下属性：

1. android:propertyName 必须的，需要操作的属性。
2. android:valueTo 必须的，目标值。
3. android:valueFrom 起始值，如果不设置，会调用 target 的 get 方法，如果 get 方法不存在则会报错（可以使用装饰器模式）。
4. android:duration 持续时间单位 ms，默认值 300ms。
5. android:startOffset 调用 start 后延迟多少 ms 执行动画。
6. android:repeatCount 重复次数，0 表示不重复，1 表示重复一次，-1 表示无限循环。
7. android:repeatMode reverse | restart 前者表示偶数次动画将动画起点当做终点，将终点当成起点，默认 restart 。
8. android:valueType intType | floatType(default)，当 value 为颜色时，不要设置该值。

```xml
<!--res/animator/animator_object.xml-->
<?xml version="1.0" encoding="utf-8"?>
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"  android:propertyName="translationX" android:valueTo="700f" android:duration="2000"  android:startOffset="1000" android:repeatCount="-1" android:repeatMode="reverse" />
```

```java
private void startAnimator() {
    Animator animator = AnimatorInflater.loadAnimator(this, R.animator.animator_object);
    animator.setTarget(btn);
    animator.start();
}
```

## animator

animator 对应 ValueAnimator.java，其拥有如下属性：

1. android:valueTo 必须的，目标值。
2. android:valueFrom 必须的，起始值。
3. android:duration 持续时间单位 ms，默认值 300ms。
4. android:startOffset 调用start后延迟多少 ms 执行动画。
5. android:repeatCount 重复次数，0 表示不重复，1 表示重复一次，-1 表示无限循环。
6. android:repeatMode reverse | restart 前者表示偶数次动画将动画起点当做终点，将终点当成起点，默认restart。
7. android:valueType intType | floatType(default)，当 value 为颜色时，不要设置该值。

```xml
<!--res/animator/animator_animator.xml-->
<?xml version="1.0" encoding="utf-8"?>
<animator xmlns:android="http://schemas.android.com/apk/res/android" android:valueFrom="0" android:valueTo="100" android:duration="1000" android:repeatMode="reverse" android:repeatCount="0" android:startOffset="1000"/>
```

```java
private void startAnimator() {
    ValueAnimator animator = (ValueAnimator) AnimatorInflater.loadAnimator(this, R.animator.animator_animator);
    animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
          @Override
          public void onAnimationUpdate(ValueAnimator animation) {
              Log.d(TAG, "onAnimationUpdate: " + animation.getAnimatedValue());
          }
    });
    animator.start();
}
```

## set

set 对应 AnimatorSet.java，其拥有如下属性：

1. android:ordering 执行顺序 sequentially | together (default)，前者表示顺序执行，上个动画执行完了，下个动画才开始执行，如果上个动画 repeatCount 为 -1 则下个动画永远不可能执行，后者表示同时执行。

```xml
<!--res/animator/animator_set.xml-->
<?xml version="1.0" encoding="utf-8"?>
<set android:ordering="together" xmlns:android="http://schemas.android.com/apk/res/android">
    <objectAnimator xmlns:android="http://schemas.android.com/apk/res/android" android:propertyName="translationX" android:valueTo="700f" android:duration="2000" android:startOffset="1000" android:repeatCount="-1" android:repeatMode="reverse" />
    <objectAnimator xmlns:android="http://schemas.android.com/apk/res/android" android:propertyName="backgroundColor" android:valueTo="#000" android:duration="2000" android:startOffset="1000" android:repeatCount="-1" android:repeatMode="reverse" />
</set>
```

```java
private void startAnimator() {
    Animator animator = AnimatorInflater.loadAnimator(this, R.animator.animator_set);
    animator.setTarget(btn);
    animator.start();
}
```

