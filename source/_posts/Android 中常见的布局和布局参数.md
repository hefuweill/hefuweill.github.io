---
title: Android 中常见的布局和布局参数
date: 2018-04-16 15:12:32
type: "Android"
---

## 前言

前几篇文章复习总结了 Android 的四大组件，现在开始复习 Android 中常用的布局包括以下几种，先从 LinearLayout 开始说起 <!-- more -->

- LinearLayout
- RelativeLayout
- FrameLayout
- TableLayout
- AbsoluteLayout
- ContraintLayout

## LinearLayout（线性布局）

> LinearLayout是一种将其子View水平单列或者垂直单行进行排列的布局

线性布局有以下几个重要的属性。

`android:orientation` 取值为 vertical 或者 horizontal ，分别表示垂直布局和水平布局。

`android:layout_weight `权重，应用场景主要有需要按比例分配空间或者填充剩余空间。

`android:layout_gravity` 重心，当 orientation 为 vertical 时 left 和 right 生效，为 horizontal 时 top 和 bottom 生效

`android:divider `分割线图片，需要配合下面那个属性。

`android:showDividers` 显示分割线的位置四选一，none、begin、end、middle。

假设需要实现把一个 EditText 和 Bottom 放置于一行并且 EditText 填充除了 Button 以外的所有区域，可以使用以下代码实现。

```xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal">
    <EditText
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"/>
    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="send"/>
</LinearLayout>
```

## RelativeLayout（相对布局）

> RelativeLayout 是一种子 View 能够描述与其它子 View 或者父 View 之间相对关系的布局

相对布局有以下几个重要的属性。

`android:layout_alignTop` 与指定 View 的顶部对齐。

`android:layout_alignBottom` 与指定 View 的底部对齐。

`android:layout_alignLeft` 与指定 View 的左边对齐。

`android:layout_alignRight` 与指定 View 的右边对齐。

`android:layout_alignParentTop` 与父 View 的顶部对齐。

`android:layout_alignParentBottom` 与父 View 的底部对齐。

`android:layout_alignParentLeft` 与父 View 的左边对齐。

`android:layout_alignParentRight` 与父 View 的顶部对齐。

`android:layout_toLeftOf` 当前 View 的 Right 与指定 View 的左边对齐。

`android:layout_toRightOf` 当前 View 的 Left 与指定 View 的右边对齐。

`android:layout_above` 当前 View 的 Bottom 与指定 View 的上面对齐。

`android:layout_alignBaseLine` 当前 View 的文本底部与指定 View 的文本底部对齐。

`android:layout_centerHorizontal` 是否水平居中。

`android:layout_centerVertical` 是否垂直居中。

`android:layout_centerInParent` 是否水平垂直居中于父 View。

这里以列表中的一个 Item 为例左边是一张图片，右边上面是一个 title，下面是 subTitle。

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <ImageView
        android:id="@+id/iv"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:layout_margin="10dp"
        android:background="@mipmap/ic_launcher"
        android:layout_alignParentLeft="true"/>

    <TextView
        android:id="@+id/tv_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_toRightOf="@id/iv"
        android:layout_alignTop="@id/iv"
        android:text="我是标题"/>

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_above=""
        android:layout_toRightOf="@id/iv"
        android:layout_below="@+id/tv_title"
        android:text="我是子标题"/>

</RelativeLayout>
```

## FrameLayout（帧布局）

> FrameLayout是一种将每个子View当做一帧，层叠显示的View

该布局没有比较特别的属性，就只是一层层的叠加。

## TableLayout（表格布局）

> TableLayout 继承于 LinearLayout，将其子 View 横向或者竖向排列，一般子 View 是 TableRow

表格布局有以下几个重要的属性

- `android:collapseColumns` 隐藏哪几列，比如 0，2 表示隐藏第 1、3 两列。
- `android:stretchColumns` 设置哪些列可以被拉伸，如果水平方向还有空余的空间则拉伸这些列。
- `android:shrinkColumns` 设置哪些列可以收缩，当水平方向空间不够时会进行收缩。

## AbsoluteLayout（绝对布局）

> AbsoluteLayout 由于无法适配屏幕在API3已经被标为过时了

## ContraintLayout（约束布局）

> ContraintLayout 是一种允许您以灵活的方式定位和调整小部件的布局

使用该布局能够显著的解决布局嵌套过多的问题。

* 通过可视化布局参考[这篇文章]("https://blog.csdn.net/guolin_blog/article/details/53122387")。
* 通过代码布局参考这篇文章[这篇文章]("https://www.jianshu.com/p/17ec9bd6ca8a")。

