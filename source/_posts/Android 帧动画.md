---
title: Android 帧动画
date: 2018-05-14 18:22:13
type: "Android"
---

## 前言

Android 中帧动画对应的 xml 文件放于 res/drawable 中，本质是轮流的按顺序播放一组图片。<!-- more --> 

## 帧动画

示例代码如下所示：

```xml
<!--res/drawable/animation_frame oneshot默认为false，true表示只执行一次，false表示不断的执行-->
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="true">
    <item android:duration="200" android:drawable="@drawable/icon1" />
    <item android:duration="200" android:drawable="@drawable/icon2" />
    <item android:duration="200" android:drawable="@drawable/icon3" />
    <item android:duration="200" android:drawable="@drawable/icon4" />
    <item android:duration="200" android:drawable="@drawable/icon5" />
    <item android:duration="200" android:drawable="@drawable/icon6" />
</animation-list>
```

```java
private void startAnimation() {
    ImageView iv = findViewById(R.id.iv);
    iv.setBackgroundResource(R.drawable.animation_frame);
    Drawable background = iv.getBackground();
    if (background instanceof AnimationDrawable) {
        ((AnimationDrawable) background).run();
    }
}
```

