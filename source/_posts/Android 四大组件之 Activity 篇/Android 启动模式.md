---
title: Android 启动模式
date: 2018-04-11 13:25:13
type: "Android"
---

## 前言

昨天在复习总结 Activity 相关知识的时候，无意中看到了启动模式，于是就想把启动模式总结下。那么启动模式有什么用？就是指定 Activity 该怎么运行，比如决定新开的 Activity 运行在调用方的任务栈中还是运行于一个新的任务栈等等。<!-- more --> 

## 四种启动模式

下面所讲述的启动模式都是不加 Intent 的 flag 时的情形。

### standard（默认值）

该启动模式的 Activity 可以位于所有任务栈，每次启动都会新创建一个 Activity 实例，taskAffinity 属性对该模式无影响，新启动的 Activity 始终与原有 Activity 同属于一个栈（前提是原有 Activity 的启动模式不是 singleInstance）。

- **例一** 当应用程序 A 的 Activity A1  启动应用程序 B 的  Activity B1（standard），B1 会和 A1同属于一个栈（前提是 A1 的启动模式不是 singleInstance ）。
- **例二** 当应用程序 A 的 Activity A1（standard） 不停的启动自己，那么栈中 A1 实例就会越来越多。

### singleTop

栈顶复用模式，该启动模式与 standard 的区别仅仅在于如果待启动的 Activity 已经处于目标栈顶了那么将不会再次创建实例，而只是调用该 Activity 的 ```onNewIntent``` 回调（其实会先调用 `onPause` 然后才是 `onNewIntent` 最后 `onResume` ）。

* **例一** 当应用程序 A 的 Activity A1(singleTop) 不停的启动自己，栈中还是只会有一个实例只是 `onNewIntent` 不断的调用。

### singleTask

该启动模式的 Activity 整个系统的所有任务栈最多只能拥有一个实例，并且其启动后所在的任务栈是固定的，固定为特定 taskAffinity 的任务栈，如果任务栈中已经拥有了一个该 Activity 的实例那么不会再创建实例而是将该实例上面的所有 Activity 都关闭掉，然后调用该 Activity 的`onNewIntent`。 

* **例一** 当应用程序 A 的 Activity A1 启动应用程序 B 的 Activity B1(singTask) ，B1 **不和 A1 同属于**一个栈（特别的当 A1 与 A2 的 taskAffinity 一致时会运行在同一个栈中）。
* **例二** 应用程序 A 有 A1(singleTask)、A2 两个Activity，A1 启动 A2，然后 A2 启动A1，这时候会发现任务栈中只有A1了。

### singleInstance

该启动模式与 singleTask 的唯一区别就是其会运行在一个单独的栈中，并且该栈中不允许其它Activity 进入。

### 返回页面问题 

点击返回键，系统会关闭当前处于前台的任务栈栈顶的 Activity 如果处于前台的任务栈中的 Activity 都被关闭了就会将最近被切换到后台的返回栈切换到前台以此循环。

* **例一** 应用程序 A 有 A1、A2（singleTask）两个 Activity，应用程序 B 有 B1 一个 Activity，A1启动 A2 然后切换回后台，启动 B1，B1 再启动 A2，这时候点击返回键会返回哪个页面？答案是 A1，因为当 B1 启动了 A2 时系统判定特定 taskAffinity 的任务栈中 A2 实例是否存在，如果存在就把包括 A1、A2 的任务栈整个切换回前台，所以点击返回键出现的是 A1。

### 通过设置常用 Flag 启动 Activity

#### `FLAG_ACTIVITY_NEW_TASK`

该 Flag 有以下两个作用

1. 第一个作用是使非 Activity 的 Context 可以启动 Activity 比如在 Service 里面启动一个 Activity 就得设置这个 Flag。
2. 第二个作用是将启动模式为 standard、singleTop 的 Activity 运行在特定 taskAffinity 的任务栈中。**注意当设置了该 Flag 以后调用 startActivityForResult 将会立即返回**（启动启动模式为 singleTask 和 singleInstance 的 Activity 自带该 Flag）。

* **例一** 在不考虑修改了 taskAffinity 的情况下，当应用程序 A 的 Activity A1（standard）通过添加 `FLAG_ACTIVITY_NEW_TASK` 后启动自己会发现添加了该 Flag 与没加一样，还是会每次创建一个 A1 的实例压入栈，原因是添加了该 Flag 后系统会查询待启动的 Activity 也就是 A1 指定的taskAffinity 栈是否存在，如不存在就创建该栈然后创建一个实例压入栈，如果存在会创建一个实例压入栈。

    思考：如果在本例中 A1 的启动模式为 singleTop、singleTask、singInstance 又会怎么样？首先对 singleTask 和 singleInstance 无影响因为它们默认就设置了该 Flag，对于 singleTop 在本例中也相当于没设置该 Flag，点击启动不会创建新的实例只是会调用 `onNewIntent` 。

* **例二** 在不考虑修改了 taskAffinity 的情况下，当应用程序 A 的 Activity A1 通过添加 `FLAG_ACTIVITY_NEW_TASK`  启动应用程序 B 的 Activity B1(standard) ，一共会有几个任务栈？答案是 2 个，如果没设置这个 Flag 那么结果是 1 个，设置了以后系统发现 B1 指定的taskAffinity 的栈还没启动就会启动然后创建 B1 将其压入栈。

    思考：如果在本例中 B1 的启动模式为 singleTop、singleTask、singInstance 又会怎么样？首先对 singleTask 和 singleInstance 无影响因为它们默认就设置了该 Flag，对于 singleTop 而言任务栈也会变成两个其原理与 standard 一致。

###  `FLAG_ACTIVITY_SINGLE_TOP`

该 Flag 只对目标 Activity 启动模式为 standard 起作用，使其产生的行为与 singleTop 一致。

####  `FLAG_ACTIVITY_CLEAR_TOP`

对于启动模式为 standard 的 Activity 来说，如果启动时发现在目标栈中已经有该 Activity 的实例了，那么会将该 Activity 及其上面的所有 Activity 全部销毁，然后重建该 Activity，对于启动模式是singTop 或者同时设置了 `FLAG_ACTIVITY_CLEAR_TOP `与 `FLAG_ACTIVITY_SINGLE_TOP `来说会将上面的 Activity 销毁然后回调该 Activity 的 `onNewIntent` （并不会把自己销毁再重建）。

### 启动模式的应用场景

* singleTop 比如一个 App 收到若干个通知每个通知都要打开同一个页面查看信息，这时候如果用户看完一个再点击另一个就不会再打开一个该界面而只会刷新数据。
* singleTask 一般用于应用程序的首页，比如浏览器首页。
* singleInstance 应用于社交 App 的分享界面（微博、QQ、微信等），注意这里有个问题，当 A应用程序打开分享页面然后输入了部分内容后然后切到后台，打开 B 应用程序再次打开分享页面，这样由于分享页面是 singleInstance 因此会被复用，那么编辑过的内容也会被保留，如果不需要保留我们可以监听 `onNewIntent` ，在该回调方法中清除编辑过的内容。

