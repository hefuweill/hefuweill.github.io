---
title: Android 应用启动源码分析
date: 2018-04-02 09:11:26
type: "Android"
---

## 前言

学习过程中一直很好奇，Application 实例是在哪创建的？Activity 实例是在哪创建的？Activity 的生命周期又是在哪调用的？Activity 内部界面绘制究竟在哪个时机，为什么 onCreate、onStart、onResume 都取不到 View 的宽高？应用程序启动时的白屏是怎么回事，怎么才能不显示？为了解决这些问题，下载编译了 AOSP 源码，阅读了这部分源码，本文基于 Android 7.1。<!-- more -->

## 准备

下载 AOSP 源码参考[官网]("http://source.android.com/source/downloading.html")，在线阅读源码使用[AndroidXRef]("http://androidxref.com/") 。

## Launcher

当用户点击桌面上应用图标时会执行 Launcher.onClick 方法，省略后代码如下：

```java
// packages/apps/Launcher3/src/android/launcher3/Launcher.java
public void onClick(View v) {
    Object tag = v.getTag();
    if (tag instanceof AppInfo) {
        startAppShortcutOrInfoActivity(v);
    }
}
void startAppShortcutOrInfoActivity(View v) {
    Object tag = v.getTag();
    if (tag instanceof AppInfo) {
        intent = ((AppInfo) tag).intent;
    }
    startActivitySafely(v, intent, tag);
}
public boolean startActivitySafely(View v, Intent intent, Object tag) {
    boolean success = false;
    try {
        success = startActivity(v, intent, tag);
    } catch (ActivityNotFoundException e) {
        Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
    }
    return success;
}
private boolean startActivity(View v, Intent intent, Object tag) {
	intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    startActivity(intent, optsBundle);
}
```

可以看到最终 Launcher 也是使用 Activity.startActivity 启动应用程序，这里有个疑问，这个 AppInfo 和其内部的 Intent 从哪来的？其实在最初 Launcher 加载所有 App 信息时就会构建 AppInfo 实例，内部会执行如下代码：

```java
public AppInfo(Context context, LauncherActivityInfoCompat info, UserHandleCompat user,
            IconCache iconCache, boolean quietModeEnabled) {
    intent = makeLaunchIntent(context, info, user);
}
public static Intent makeLaunchIntent(Context context, LauncherActivityInfoCompat info,
            UserHandleCompat user) {
    long serialNumber = UserManagerCompat.getInstance(context).getSerialNumberForUser(user);
    return new Intent(Intent.ACTION_MAIN)
        .addCategory(Intent.CATEGORY_LAUNCHER)
        .setComponent(info.getComponentName())
        .setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED)
        .putExtra(EXTRA_PROFILE, serialNumber);
}
```

设置了 Action 为 Intent.ACTION_MAIN，Category 为 Intent.CATEGORY_LAUNCHER ，这也就是为什么 App 的首个 Activity 必须要配置  android.intent.action.MAIN 、android.intent.category.LAUNCHER 的原因，同时还设置了 Intent.FLAG_ACTIVITY_NEW_TASK 这使得新开启的 Activity 不跟 Launcher 运行在同一个栈中。继续看 Activity.startActivity。

```java
// frameworks/base/core/java/android/app/Activity.java
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
    if (mParent == null) {
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
            this, mMainThread.getApplicationThread(), mToken, this,
            intent, requestCode, options);
    }
}
```

使用 Instrumentation 启动 Activity。 注：该类负责管理应用程序与系统的交互。

```java
// frameworks/base/core/java/android/app/Instrumentation.java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    try {
        intent.prepareToLeaveProcess(who); // 这个主要是改变下Bundle的flag
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                           intent.resolveTypeIfNeeded(who.getContentResolver()),
                           token, target != null ? target.mEmbeddedID : null,
                           requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
// frameworks/base/core/java/android/app/ActivityManagerNative.java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        IActivityManager am = asInterface(b);
        return am;
    }
};
static public IActivityManager getDefault() {
    return gDefault.get();
}
```

由此可以看到最终会执行 IActivityManager.startActivity ，先从 ServiceManager 中获取 IBinder 实例，然后将其转换为 IActivityManager ，通信方式同 AIDL 就不展开了。

ServiceManager 是客户端获取系统服务的桥梁，SystemServer 进程通过 ServiceManager.addService 添加了很多服务，ActivityManagerService 就是其中之一，详细添加地方在 SystemServer.startBootstrapServices 方法中，添加后客户端程序就能使用 ServiceManager.getService 获取 SystemServer 进程中指定服务对应的 BinderProxy。至于 ServiceManager 的具体实现，由于是 Native 的暂时就先不了解了。

## SystemServer

当客户端调用 IActivityManager.startActivity 后服务端就执行了 ActivityManagerService.startActivity 。

### AMS.startActivity

代码如下：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public final int startActivity(...) {
    return startActivityAsUser(...);
}
public final int startActivityAsUser(...) {
    return mActivityStarter.startActivityMayWait(...);
}
// frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java
final int startActivityMayWait(...) {
    boolean componentSpecified = intent.getComponent() != null;
    final Intent ephemeralIntent = new Intent(intent);
    intent = new Intent(intent);
    // 从 PackageManagerService 中查询所有符合 Intent 条件的组件信息，并根据优先级、用户偏爱选择最佳组件，如果确定不了最佳的那么返回一个让用户选择的组件。
    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId);
    // 从 ResolveInfo 从取出 ActivityInfo，以及为 Intent 设置 Component 。
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
    int res = startActivityLocked(...);
    return res;
}
```

继续看 startActivityLocked 方法。

### AST.startActivityLocked

代码如下：

```java
final int startActivityLocked(...) {
    int err = ActivityManager.START_SUCCESS;
    ProcessRecord callerApp = null;
    if (caller != null) {
        // 获取调用者的 ProcessRecond ，如果获取不到那么就将 err 置为 ActivityManager.START_PERMISSION_DENIED 。
        callerApp = mService.getRecordForAppLocked(caller);
        if (callerApp != null) {
            callingPid = callerApp.pid;
            callingUid = callerApp.info.uid;
        } else {
            err = ActivityManager.START_PERMISSION_DENIED;
        }
    }
    ActivityRecord sourceRecord = null;
    ActivityRecord resultRecord = null;
    if (resultTo != null) {
        // 根据启动方 Activity 的 token，获取其对应 ActivityRecord 信息，如果 requestCode 小于 0 则忽略，不然将 resultRecord 设置为启动方 ActivityRecord 信息。
        sourceRecord = mSupervisor.isInAnyStackLocked(resultTo);
        if (sourceRecord != null) {
            if (requestCode >= 0 && !sourceRecord.finishing) {
                resultRecord = sourceRecord;
            }
        }
    }
    final int launchFlags = intent.getFlags();
    if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
         // 如果启动方设置了 Intent.FLAG_ACTIVITY_FORWARD_RESULT ，那么会把启动后页面的返回结果给启动方的启动方，如 A、B、C 三个界面，A 通过 startActivity 启动 B，B 通过 startActivity 带上该 flag 启动 C，那么 B 会将 C 返回的。
    }
    // 由于在 startActivityMayWait 中，如果有 aInfo ，则会给 Intent 设置 Component，如果 Intent 无 Component 那么可以认定为找不到 Intent 指定的 Activity。
    if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
        err = ActivityManager.START_INTENT_NOT_RESOLVED;
    }
    // 如果 Component 不为空，可能原因是无 aInfo 用户自身设置了 Component 或者有 aInfo 上个方法给设置的，对于前者，可以认定用户通过 Intent 显示指定的 Activity 不存在。
    if (err == ActivityManager.START_SUCCESS && aInfo == null) { // 5
        err = ActivityManager.START_CLASS_NOT_FOUND;
    }
    // 这里由于不是 startActivityForResult 启动的，因此 resultStack 为 null 。
    final ActivityStack resultStack = resultRecord == null ? null : resultRecord.task.stack;
    if (err != START_SUCCESS) {
        // 返回错误给启动客户端。
        return err;
    }
    ActivityRecord r = new ActivityRecord(...);
    if (outActivity != null) {
        outActivity[0] = r;
    }
    err = startActivityUnchecked(...);
    return err;
}
```

这个方法主要内容还是处理错误情况，接着看 startActivityUnchecked。

### AST.startActivityUnchecked

代码如下：

```java
private int startActivityUnchecked(...) {
    // 设置初始状态，解析LaunchMode。
    setInitialState(...);
    // 给某些情况添加FLAG_ACTIVITY_NEW_TASK。
    computeLaunchingTaskFlags();
	// 判断是否要将新启动的Activity放入原有的栈中，如果返回非null表示放入原有栈。
    mReusedActivity = getReusableIntentActivity();
    if (mReusedActivity != null) {
        if (mStartActivity.task == null) {
            mStartActivity.task = mReusedActivity.task;
        }
        if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
            || mLaunchSingleInstance || mLaunchSingleTask) {
            // 如果拥有FLAG_ACTIVITY_CLEAR_TOP或者启动模式为singleTask、singleTop，则清除该Activity以上的所有Activity。
            final ActivityRecord top = mReusedActivity.task.performClearTaskForReuseLocked(
                mStartActivity, mLaunchFlags);
            // 表示找到了可以复用的Activity
            if (top != null) {
                if (top.frontOfTask) {
                    // 复用Activity处于栈底，重新设置task的intent属性。
                    top.task.setIntent(mStartActivity);
                }
                // 调用Activity的onNewIntent方法。
                top.deliverNewIntentLocked(mCallingUid, mStartActivity.intent,
                                           mStartActivity.launchedFromPackage);
            }
        }
        // 如果目标task已经存在了，那么切换task到前台。
        mReusedActivity = setTargetStackAndMoveToFrontIfNeeded(mReusedActivity);
    }
	...
    mTargetStack.startActivityLocked(mStartActivity, newTask, mKeepCurTransition, mOptions);
    mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
    return START_SUCCESS;
}

```

#### AST.setInitialState

代码如下：

```java
private void setInitialState(ActivityRecord r, ActivityOptions options, TaskRecord inTask,
            boolean doResume, int startFlags, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor) {
    reset();
    mStartActivity = r;
    mIntent = r.intent;
    mOptions = options;
    mCallingUid = r.launchedFromUid;
    mSourceRecord = sourceRecord;
    mVoiceSession = voiceSession;
    mVoiceInteractor = voiceInteractor;
    mLaunchBounds = getOverrideBounds(r, options, inTask);
    mLaunchSingleTop = r.launchMode == LAUNCH_SINGLE_TOP;
    mLaunchSingleInstance = r.launchMode == LAUNCH_SINGLE_INSTANCE;
    mLaunchSingleTask = r.launchMode == LAUNCH_SINGLE_TASK;
    mLaunchFlags = adjustLaunchFlagsToDocumentMode(
        r, mLaunchSingleInstance, mLaunchSingleTask, mIntent.getFlags());
    mLaunchTaskBehind = r.mLaunchTaskBehind
        && !mLaunchSingleTask && !mLaunchSingleInstance
        && (mLaunchFlags & FLAG_ACTIVITY_NEW_DOCUMENT) != 0;
    sendNewTaskResultRequestIfNeeded();
    ...
}
private void sendNewTaskResultRequestIfNeeded() {
    if (mStartActivity.resultTo != null && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0
        && mStartActivity.resultTo.task.stack != null) {
        Slog.w(TAG, "Activity is launching as a new task, so cancelling activity result.");
        mStartActivity.resultTo.task.stack.sendActivityResultLocked(...);
        mStartActivity.resultTo = null;
    }
}
```

这个方法需要注意，**如果 Intent 里面设置了 Intent_FLAG_ACTIVITY_NEW_TASK ，那么使用 startActivityForResult 启动 Activity 会立即收到 onActivityResult **。

#### AST.computeLaunchingTaskFlags

代码如下：

```java
private void computeLaunchingTaskFlags() {
	// 使用非Activity的Context启动Activity，并指定了目标栈，从Launcher启动不会进入。
    if (mSourceRecord == null && mInTask != null && mInTask.stack != null) {
        ...
    } 
    if (mInTask == null) {
        if (mSourceRecord == null) {
            // 使用非Activity的Context启动Activity，强制加上FLAG_ACTIVITY_NEW_TASK。
            if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0 && mInTask == null) {
                Slog.w(TAG, "startActivity called from non-Activity context; forcing " +
                       "Intent.FLAG_ACTIVITY_NEW_TASK for: " + mIntent);
                mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            }
        } else if (mSourceRecord.launchMode == LAUNCH_SINGLE_INSTANCE) {
            // 如果当前当前Activity的启动模式为singleInstance，那么不允许其它Activity入栈，也加上FLAG_ACTIVITY_NEW_TASK。
            mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
        } else if (mLaunchSingleInstance || mLaunchSingleTask) {
            // 如果目标Activity启动模式是singleInstance或者singleTask，也加上FLAG_ACTIVITY_NEW_TASK。
            mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
        }
    }
}
```

startActivityUnchecked 内部主要处理的还是启动模式的处理，接着执行 ActivityStack.startActivityLocked 。

#### ASK.startActivityLocked 

代码如下：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityStack.java
final void startActivityLocked(ActivityRecord r, boolean newTask, boolean keepCurTransition,
            ActivityOptions options) {
    ...
    if (!isHomeStack() || numActivities() > 0) {
        // 需要展示 previewWindow，如果切换到一个新 task 或者目标 Activity 进程还没运行。
        boolean showStartingIcon = newTask;
        ProcessRecord proc = r.app;
        if (proc == null) {
            proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
        }
        // 目标 Activity 进程没有启动。
        if (proc == null || proc.thread == null) {
            showStartingIcon = true;
        }
        if (r.mLaunchTaskBehind) {
            ...
        } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
            ActivityRecord prev = r.task.topRunningActivityWithStartingWindowLocked();
            // 展示startingWindow也就是俗称的白屏。
            r.showStartingWindow(prev, showStartingIcon);
        }
    } else {
        ...
    }
}
```

阅读到这里就找到了**显示白屏**的地方，继续往下看，看看是否有地方可以不显示白屏。

##### ARD.showStartingWindow

代码如下：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
void showStartingWindow(ActivityRecord prev, boolean createIfNeeded) {
    final boolean shown = service.mWindowManager.setAppStartingWindow();
}     
```

这里的 mWindowManager 其实现是 WindowManagerService。

##### WMS.setAppStartingWindow

代码如下：

```java
// frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public boolean setAppStartingWindow(IBinder token, String pkg,
            int theme, CompatibilityInfo compatInfo,
            CharSequence nonLocalizedLabel, int labelRes, int icon, int logo,
            int windowFlags, IBinder transferFrom, boolean createIfNeeded) {
    synchronized(mWindowMap) {
		...
        if (theme != 0) {
            AttributeCache.Entry ent = AttributeCache.instance().get(pkg, theme,                                                                com.android.internal.R.styleable.Window, mCurrentUserId);
            final boolean windowIsTranslucent = ent.array.getBoolean(
                com.android.internal.R.styleable.Window_windowIsTranslucent, false);
            final boolean windowIsFloating = ent.array.getBoolean(
                com.android.internal.R.styleable.Window_windowIsFloating, false);
            final boolean windowShowWallpaper = ent.array.getBoolean(
                com.android.internal.R.styleable.Window_windowShowWallpaper, false);
            final boolean windowDisableStarting = ent.array.getBoolean(
                com.android.internal.R.styleable.Window_windowDisablePreview, false);
            if (DEBUG_STARTING_WINDOW) Slog.v(TAG_WM, "Translucent=" + windowIsTranslucent
                                              + " Floating=" + windowIsFloating
                                              + " ShowWallpaper=" + windowShowWallpaper);
            if (windowIsTranslucent) {
                return false;
            }
            if (windowIsFloating || windowDisableStarting) {
                return false;
            }
        }
        Message m = mH.obtainMessage(H.ADD_STARTING, wtoken);
        // 发消息去添加PhoneWindow了
        mH.sendMessageAtFrontOfQueue(m);
    }
    return true;
}
```

通过阅读源码可以发现，只要 **Window_windowIsTranslucent、Window_windowIsFloating、Window_windowDisablePreview 三个有一个为 true 就不会显示白屏了**，不过虽然不显示白屏了，但是进程启动需要时间，因此会停留在桌面一段时间。

回到主线，继续执行 ActivityStart.startActivityUnchecked。

#### ASS.resumeFocusedStackTopActivityLocked

代码如下：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java
boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
}
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    return resumeTopActivityInnerLocked(prev, options);
}
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    // 执行上一个Activity.onPause
    if (mResumedActivity != null) {
        if (DEBUG_STATES) Slog.d(TAG_STATES,
                "resumeTopActivityLocked: Pausing " + mResumedActivity);
        pausing |= startPausingLocked(userLeaving, false, next, dontWaitForPause);
    }
	mStackSupervisor.startSpecificActivityLocked(next, true, false);
}
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
    // 判断将要启动的Activity进程是否存在
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                                                        r.info.applicationInfo.uid, true);

    r.task.stack.setLaunchTime(r);
    // 进程已经存在，并且ApplicationThread实例也在（表示可以正常通行）
    if (app != null && app.thread != null) {
        try {
            // 直接启动 Activity
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                   + r.intent.getComponent().flattenToShortString(), e);
        }
    }
	// 启动进程
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                                "activity", r.intent.getComponent(), false, false, true);
}
```

这里主要看看进程不存在的情况：

##### AMS.startProcessLocked

代码如下：

```java
final void startProcessLocked(...) {
    Process.ProcessStartResult startResult = Process.start(...);
}
// frameworks/base/services/core/java/android.os.Process
public static final ProcessStartResult start(...) {
    return startViaZygote(...);
}
private static ProcessStartResult startViaZygote(...)
                                  throws ZygoteStartFailedEx {
    return zygoteSendArgsAndGetResult(...);
}
private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
    try {
        final BufferedWriter writer = zygoteState.writer;
        final DataInputStream inputStream = zygoteState.inputStream;
        writer.write(Integer.toString(args.size()));
        writer.newLine();
        for (int i = 0; i < sz; i++) {
            String arg = args.get(i);
            writer.write(arg);
            writer.newLine();
        }
        writer.flush();
        ProcessStartResult result = new ProcessStartResult();
        result.pid = inputStream.readInt();
        result.usingWrapper = inputStream.readBoolean();
        if (result.pid < 0) {
            throw new ZygoteStartFailedEx("fork() failed");
        }
        return result;
    } catch (IOException ex) {
        zygoteState.close();
        throw new ZygoteStartFailedEx(ex);
    }
}
```

经过一系列流程，最终通过 Socket 发送消息给 Zygote 进程，通知其 fork 进程，然后将 pid 返回。Zygote 进程是在 init 进程启动后通过 fork 自身创建的，其主要代码在 ZygoteInit 类中，主要负责预加载类和资源（比如：Activity、Context）然后 fork 自身创建 SystemServer 进程，接着自身进入循环等待处理 Socket 消息。

## Zygote 

Zygote 创建应用程序进程代码在 ZygoteConnection.processOneCommand。

### ZC.processOneCommand

代码如下：

```java
Runnable processOneCommand(ZygoteServer zygoteServer) {
    String args[];
    Arguments parsedArgs = null;
    FileDescriptor[] descriptors;
    try {
        args = readArgumentList(); //读取socket客户端发送过来的参数列表
        descriptors = mSocket.getAncillaryFileDescriptors();
    } catch (IOException ex) {
        throw new IllegalStateException("IOException on command socket", ex);
    }
    pid = Zygote.forkAndSpecialize(...); // fork自身
    if (pid == 0) { // 1
        // 子进程
        zygoteServer.setForkChild();
        zygoteServer.closeServerSocket(); //关闭除了子进程的socket
        return handleChildProc(...);
    } else {
        // 父进程
        handleParentProc(...);
        return null;
    }  
}
```

## App

在 processOneCommand 进入到代码一时，已经是在 App 进程执行了，接着调用 handleChildProc。

### ZC.handleChildProc

代码如下：

```java
private Runnable handleChildProc(Arguments parsedArgs, FileDescriptor[] descriptors,
            FileDescriptor pipeFd, boolean isZygote) {
    closeSocket(); // 关闭LocalSocket
    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName); // 设置进程名
    }
    RuntimeInit.zygoteInit(...);
}
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) throws ZygoteInit.MethodAndArgsCaller {
    commonInit(); // 设置线程异常处理器，UserAgent 等
    nativeZygoteInit();
    applicationInit(targetSdkVersion, argv, classLoader);
}
private static final void commonInit() {
    // 这也就是Android子线程抛出异常也会crash的原因。
    Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());
    String userAgent = getDefaultUserAgent();
    System.setProperty("http.agent", userAgent);
}
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
    invokeStaticMain(args.startClass, args.startArgs, classLoader);
}
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
    Class<?> cl = Class.forName(className, true, classLoader);
    Method m = cl.getMethod("main", new Class[] { String[].class });
    int modifiers = m.getModifiers();
    // 这里抛出异常，会回到ZygoteInit.main，这样做是为了清理栈帧。
    throw new ZygoteInit.MethodAndArgsCaller(m, argv);
}
public static class MethodAndArgsCaller extends Exception implements Runnable {
    public void run() {
        try {
            //ASM传递参数为android.app.ActivityThread，因此执行其main方法
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            ...
        } catch (InvocationTargetException ex) {
            ...
        }
    }
}
```

接下来代码就执行到了 ActivityThread.main 。

### AT.main

代码如下：

```java
public static void main(String[] args) {
    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    Looper.loop();
}
```

首先 Looper.prepareMainLoop 创建了主线程的 Looper 实例，Loop.loop 死循环从消息队列中取出消息，那么是那里给主线程发消息呢，看来主要逻辑还是在 attach 方法中了。

### AT.attach

代码如下：

```java
private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                UserHandle.myUserId());
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        try {
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    } else {
        // 设置了ddms里面看到的系统进程名和进程号
        // 创建了系统服务的context
        android.ddm.DdmHandleAppName.setAppName("system_process",
                                                UserHandle.myUserId());
        try {
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
            ContextImpl context = ContextImpl.createAppContext(
                this, getSystemContext().mPackageInfo);
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
        } catch (Exception e) {
            throw new RuntimeException(
                "Unable to instantiate Application():" + e.toString(), e);
        }
    }
}
```

attach 内部其实主要工作也只是调用了 AMS.attachApplication。注意传入一个 ApplicationThread 实例，先来看看该类。

```java
private class ApplicationThread extends ApplicationThreadNative {}
public abstract class ApplicationThreadNative extends Binder implements IApplicationThread {}
```

ApplicationThreadNative 其实可以看做 AIDL 里面的 Stub 类， 因此将 ApplicationThread 实例传递给 AMS，AMS 就可以远程调用该实例的方法。

### AMS.attachApplication

代码如下：

```java
 public final void attachApplication(IApplicationThread thread) {
     synchronized (this) {
         int callingPid = Binder.getCallingPid();
         final long origId = Binder.clearCallingIdentity();
         attachApplicationLocked(thread, callingPid);
         Binder.restoreCallingIdentity(origId);
     }
 }
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {
    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    thread.bindApplication(...); // 通知ActivityThread创建Application对象
    if (normalMode) {
        try {
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
        }
    }
    return true;
}
```

方法内部首先 IPC 调用 ApplicationThread.bindApplication 。

```java
public final void bindApplication(...) {
    if (services != null) {
        // 先缓存部分服务，减少后续调用native方法获取的次数。
        ServiceManager.initServiceCache(services);
    }
    ...
    sendMessage(H.BIND_APPLICATION, data);
}
```

其内部只是发了个消息，**这时候消息是没办法处理的因为 Looper.loop 还没调用**，于是继续执行 ActivityStackSupervisor.attachApplicationLocked。

```java
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
    final String processName = app.processName;
    boolean didSomething = false;
    // 找到当前前台栈，如果栈顶Activity属于当前新建的进程，那么执行realStartActivityLocked启动该Activity。
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = stacks.get(stackNdx);
            if (!isFocusedStack(stack)) {
                continue;
            }
            ActivityRecord hr = stack.topRunningActivityLocked();
            if (hr != null) {
                if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                    && processName.equals(hr.processName)) {
                    try {
                        if (realStartActivityLocked(hr, app, true, true)) {
                            didSomething = true;
                        }
                    } catch (RemoteException e) {
                        Slog.w(TAG, "Exception in new application when starting activity "
                               + hr.intent.getComponent().flattenToShortString(), e);
                        throw e;
                    }
                }
            }
        }
    }
    return didSomething;
}
```

执行到这里其实可以与不创建进程的情况并起来了，最终在进程存在后，都是通过调用 ActivityStackSupervisor.realStartActivityLocked 启动 Activity。

```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException 
    ...
    app.thread.scheduleLaunchActivity(...);
    return true;
}
public final void scheduleLaunchActivity(...);
	...
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

scheduleLaunchActivity 与 bindApplication 一样都是发了个消息返回了，现在 AMS 执行完成了，ActivityThread 继续执行 Looper.loop，因此会先执行 BIND_APPLICATION 消息。

### AT.handleBindApplication

在 H 类中 BIND_APPLICATION 消息最终调用 handleBindApplication 代码如下：

```java
private void handleBindApplication(AppBindData data) {
    mBoundApplication = data;
    mConfiguration = new Configuration(data.config);
    mCompatConfiguration = new Configuration(data.config);
	// 在等待调试器前设置进程名
    Process.setArgV0(data.processName);
    android.ddm.DdmHandleAppName.setAppName(data.processName,
                                            UserHandle.myUserId());
	// 重置时区
    TimeZone.setDefault(null);
    LocaleList.setDefault(data.config.getLocales());
    // 获取 LoadApk
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
    // 设置时间格式
    final boolean is24Hr = "24".equals(mCoreSettings.getString(Settings.System.TIME_12_24));
    DateFormat.set24HourTimePref(is24Hr);
    // Android 7.0 以上不允许使用 file:// Uri
    if (data.appInfo.targetSdkVersion >= Build.VERSION_CODES.N) {
        StrictMode.enableDeathOnFileUriExposure();
    }
    if (data.debugMode != IApplicationThread.DEBUG_OFF) {
        Debug.changeDebugPort(8100);
        if (data.debugMode == IApplicationThread.DEBUG_WAIT) {
            IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                // 通知 AMS 弹出等待调试器弹框，也就是说在这之前的代码是没法debug的。
                mgr.showWaitingForDebugger(mAppThread, true);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            // 循环等待调试器 attach 。
            Debug.waitForDebugger();
            try {
                // 通知 AMS 关闭等待调试器弹框。
                mgr.showWaitingForDebugger(mAppThread, false);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }

        }
    }
    // 初始化默认的网络代理
    final IBinder b = ServiceManager.getService(Context.CONNECTIVITY_SERVICE);
    if (b != null) {
        final IConnectivityManager service = IConnectivityManager.Stub.asInterface(b);
        try {
            final ProxyInfo proxyInfo = service.getProxyForNetwork(null);
            Proxy.setHttpProxySystemProperty(proxyInfo);
        } catch (RemoteException e) {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            throw e.rethrowFromSystemServer();
        }
    }
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    // 创建 Instrumetation
    mInstrumentation = new Instrumentation();
    try {
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;
        if (!data.restrictedBackupMode) {
            if (!ArrayUtils.isEmpty(data.providers)) {
                // 创建ContentProvider实例，并调用其onCreate。
                installContentProviders(app, data.providers);
            }
        }
        try {
            // 回调Application.create.
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            ...
        }
    }
}
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
    // 如果当前进程已经有Application了那么直接返回
    if (mApplication != null) {
        return mApplication;
    }
    Application app = null;
    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }
    try {
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        // 内部通过返回创建Application实例
        app = mActivityThread.mInstrumentation.newApplication(
            cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        ...
    }
    mApplication = app;
    // 重写所有apk库中的R常量
    SparseArray<String> packageIdentifiers = getAssets().getAssignedPackageIdentifiers();
    final int N = packageIdentifiers.size();
    for (int i = 0; i < N; i++) {
        final int id = packageIdentifiers.keyAt(i);
        if (id == 0x01 || id == 0x7f) {
            continue;
        }
        rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
    }
    return app;
}
```

可以看到方法内部主要就是创建 Application 对象并且回调其 onCreate，注意 ContentProvider 的 onCreate，优先于 Application 的 onCreate。

### AT.handleLaunchActivity

在 H 类中 LAUNCH_ACTIVITY 消息最终调用 handleLaunchActivity 代码如下：

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    // 提前准备WMS的本地代理
    WindowManagerGlobal.initialize();
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        Bundle oldState = r.state;
        handleResumeActivity(r.token, false, r.isForward,
                             !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
        }
    } else {
        // 出错了，那么就直接关闭。
        try {
            ActivityManagerNative.getDefault()
                .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

handleLaunchActivity 内部主要执行了 performLaunchActivity 以及 handleResumeActivity。

#### AT.performLaunchActivity

代码如下：

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    Activity activity = null;
    try {
        // 反射创建Activity对象
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
    } catch (Exception e) {
        ...
    }
    try {
        // 如果当前进程没Application则创建不然返回
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        if (activity != null) {
            Context appContext = createBaseContextForActivity(r, activity);
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            // 执行activity.attach
            activity.attach(appContext, this, getInstrumentation(), r.token,
                            r.ident, app, r.intent, r.activityInfo, title, r.parent,
                            r.embeddedID, r.lastNonConfigurationInstances, config,
                            r.referrer, r.voiceInteractor, window);
            // 设置主题
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }
            activity.mCalled = false;
            if (r.isPersistable()) {
                ...
            } else {
                // 回调Activity.onCreate。
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            // onCreate内部必须要调用父类的onCreate
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
            r.stopped = true;
            if (!r.activity.mFinished) {
                // 执行Activity.onStart
                activity.performStart();
                r.stopped = false;
            }
            if (!r.activity.mFinished) {
                if (r.isPersistable()) {
                    ...
                } else if (r.state != null) {
                    // 执行Activity.onRestoreInstanceState
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            if (!r.activity.mFinished) {
                activity.mCalled = false;
                if (r.isPersistable()) {
                    ...
                } else {
                    // 执行Activity.onPostCreate
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                }
                // 同样必须要调用父类
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onPostCreate()");
                }
            }
        }
        r.paused = true;
        mActivities.put(r.token, r);
    } catch (SuperNotCalledException e) {
        throw e;
    } catch (Exception e) {
        ...
    }

    return activity;
}
```

方法内部主要创建了 Activity 实例并且调用其 attach、onCreate、onStart、onRestoreInstanceState(如果有)、onPostCreate。接着看 handleResumeActivity。

#### AT.handleResumeActivity

```java
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
	// 调用Activity.onResume
    r = performResumeActivity(token, clearHide, reason);
    if (r != null) {
        final Activity a = r.activity;
        r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            ...
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            }
        } else if (!willBeVisible) {
            ...
        }
    }
    ...
}
```

方法内部主要是调用了 Activity.onResume，接着调用 WindowManager.addView 执行 View 的三大流程。

## 总结

基本阅读完整个启动流程，前言那几个问题都可以解决了。

1. Application 实例是在哪创建的？Instrumetation.newApplication。
2. Activity 实例是在哪创建的？Instrumetaion.newActivity。
3. Activity 内部界面绘制究竟在哪个时机，为什么 onCreate、onStart、onResume 都取不到 View 的宽高？在 onResume 执行完毕后才开始，因此 onResume 其之前都是取不到的。
4. 应用程序启动时的白屏是怎么回事，怎么才能不显示？设置 Window_windowIsTranslucent、Window_windowIsFloating、Window_windowDisablePreview 三个有一个为 true 就不会显示白屏了。

注：挂载并运行模拟器命令

```java
sudo hdiutil attach ~/android.dmg.sparseimage -mountpoint /Volumes/android 
emulator -skin 720x1280
```

