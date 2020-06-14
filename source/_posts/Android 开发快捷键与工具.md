---
title: Android 开发快捷键与工具
date: 2018-03-14 13:11:26
type: "Android"
---

### 前言

本文主要是参考自 《Android 群英传：神兵利器》，总结对 Android 开发有用的开发快捷键及工具。所有快捷键全部基于 mac 操作系统。<!-- more --> 

### 窗口切换

* command + ~ 可以在一个应用程序中切换窗口（切换多个 Android Studio 窗口、终端，对 Chrome 浏览器无效）。
* command + w 关闭一个窗口（比如关闭一个浏览器项或者关闭 Android Studio 中一个打开的文件）。
* command + q 退出应用程序 (关闭Android Studio等等)。
* command + option + esc 强制退出应用程序，在 ANR 时使用。
* command + n 新建一个窗口（再打开一个浏览器窗口、终端窗口）。

### 截图

* command + shift + 4 类似 QQ 的截图。
* command + shift + 4 + space 截取当前屏幕。

### 编辑

* command + ← / →  可以使光标移动到行首/行尾。
* option + ← / →  可以使光标在行内按单词移动。
* command + ↑ / ↓ 可以把光标定位到首部/尾部（ Chorme 可以，Android Studio 中用于选择文件）。
* command + delete 删除一行。

### 终端

* control + a 终端中将光标移动到文本的首部。
* control + e 终端中将光标移动到文本的尾部。
* open . 可以在 finder 中打开该目录。
* \> filename 可以把输出内容写到该文件中，创建文本文件也可以通过这个方式。

### Git

* 在多人开发中当想要 push 代码时发现别人已经有了提交，Git 会提示使用 git pull，如果这样操作会留下一个 mergeHistory，使用 git pull –rebase 可以避免这种情况。
* 当一个分支中有内容修改并没有提交，是无法切换分支的，这时候可以使用 git stash 保存，待下次处理完后再回来 git stash pop。
* git commit -am 添加并且提交。
* git pull == git fetch + merge local。

### Android Studio

* option + shift 可以实现选中多个代码块，会产生多个光标，可以同时进行输入、删除。
* command + shift + a 可以快速打开 Android Studio 的功能，比如输入 Preferences 可以打开Android Studio 的偏好设置。
* command + e 可以显示最近打开的文件。
* command + y 可以显示出方法的定义信息。
* command + shift + e 可以显示最近编辑过的文件。会和搜狗输入法切换 English 输入模式冲突。
* **command + shift + u 大小写切换。**
* command + option + f 提取局部变量到成员变量。
* **command + f12 可以显示出代码的大纲，可以输入关键字进行搜索。**
* command + - / + 可以对代码进行折叠、展开。
* **command + b 等同于 command + 鼠标左键 可以快速进入该方法源码实现。**
* command + p 可以查看方法的参数类型。
* F1 显示参数类型及注释比上面那个详细。
* **control + tab 可以用于切换 tab，在开发中操作多个类时进行切换。**
* **option / command + shift + ↑ / ↓ 可以向上/向下移动整行代码。**
* option + F7 可以查看当前方法在那被调用。
* **F3 可以给当前代码添加书签或者删除书签，然后通过 command + F3  可以调出书签进行查看，该方法可以用于记录代码中的关键点，查阅源码时挺有用。**
* View - Appearance - Enter Presentation 打开演示模式，该模式可以把当前屏幕的几乎所有空间都用来显示代码，代码大小会变大好多。

### 断点调试

* 条件断点，打了断点以后点击右键在弹出的窗口进行设置。

    ![条件断点](条件断点.png)

* 异常断点，打了断点以后运行过程中出现指定异常就会停止，同时也可以设置条件，以及从哪个断点开始检测，进入方式为 run - View Breakpoints。

    ![异常断点](异常断点.png)

* 日志断点，可在不重新运行代码的情况下输出日志。

    ![日志断点](日志断点.png)

* 临时断点，断点执行一次后就自动移除，勾选上面界面的 Remove once hit，或者使用快捷键 command + option + shift + F8 一点都不快捷！难按。

    