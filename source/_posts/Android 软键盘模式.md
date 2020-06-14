---
title: Android 软键盘模式
date: 2018-08-22 11:06:40
type: "Android"
---

## 前言

在日常开发过程中会遇到很多关于软键盘的问题，比如 RN 底部按钮会被软键盘顶起、切换 Activity 后软键盘不自动消失等等，因此写这篇文章记录下常用的软键盘模式。<!--more-->

## 软键盘模式(WindowManager.LayoutParams)

* 首先要分清是前进还是后退 假设有 A、B、C 三个页面，A 启动 B，表示前进，C 返回 B 表示后退。

1. SOFT_INPUT_STATE_UNSPECIFIED 默认模式，系统会根据界面采取相应的软键盘的显示模式。
2. SOFT_INPUT_STATE_UNCHANGED 当这个 Activity 出现时，软键盘将一直保持在上一个 Activity 里的状态，无论是后退还是前进。
3. SOFT_INPUT_STATE_HIDDEN 前进到设置该模式的 Activity 时如果键盘已经显示会隐藏键盘，回退到该Activity 则软键盘显示保持不变。
4. SOFT_INPUT_STATE_ALWAYS_HIDDEN 前进或后退到该 Activity 如果软键盘已经显示都会关闭。
5. SOFT_INPUT_STATE_VISIBLE 当前进到设置该模式的 Activity 时会显示软键盘，回退到该 Activity 则软键盘显示保持不变。
6. SOFT_INPUT_STATE_ALWAYS_VISIBLE 当前进或后退到该 Activity 如果软键盘已经消失会显示。
* 下面是几种当软键盘弹出时是否需要调整 Activity 的视图。
7. SOFT_INPUT_ADJUST_UNSPECIFIED 未指定模式系统将根据情况使用下面的几种模式。
8. SOFT_INPUT_ADJUST_RESIZE 如果当前 Activity 有 focus 的输入框那么进入时就会弹出软键盘，并且当软键盘显示时会缩小 ContentView(id 为 android.R.id.content) 的高度，用以显示软键盘，注意该属性不能与SOFT_INPUT_ADJUST_PAN 一起使用。
9. SOFT_INPUT_ADJUST_PAN 如果当前 Activity 有 focus 的输入框进入时不会弹出软键盘，并且当软键盘显示时会把整个 ContentView 向上移动一段距离直到输入框能够显示出来(可能会出现短暂的底部黑屏)，注意该属性不能与 SOFT_INPUT_ADJUST_RESIZE 一起使用。
10. SOFT_INPUT_ADJUST_NOTHING 当软键盘显示时不缩小 ContentView 的高度，也不移动 ContentView，可能会导致输入框不可见。

* 上述几个 Mode 作用于滚动视图也是如此，设置成 SOFT_INPUT_ADJUST_NOTHING ，还是不改 变Activity 的视图只是弹出一个输入框。设置成 SOFT_INPUT_ADJUST_RESIZE，则会减少 ContentView的高度，滚动视图会向上滚动，直到 Focus 的输入框显示在输入框上面，进入 Activity 时如果有输入框 focus 也会自动弹出软键盘。设置成 SOFT_INPUT_ADJUST_PAN，可能会导致滚动视图的上边的 Item 不可见因为滚动视图向上移动出了屏幕。

## Tips:

1. 想要弹出 PopupWindow 的时候隐藏软键盘。
	```java
	window.setFocusable(true);
	window.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_STATE_ALWAYS_HIDDEN);
	```
2. 弹出的 PopupWindow 直接覆盖在软键盘上面。
	```java
	window.setFocusable(true);
	window.setInputMethodMode(PopupWindow.INPUT_METHOD_NOT_NEEDED);
	```