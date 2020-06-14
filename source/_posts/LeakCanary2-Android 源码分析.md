---
title: LeakCanary2-Android 源码分析
date: 2019-06-25 11:08:12
type: "Android"
---



## 前言

上篇文章分析了 LeakSentry 的源码，本文在此基础上来分析下 LeakCanary 的核心库——Leakcanary-android的源码。<!--more-->

## 原理

LeakCanary 的原理为当 Activity、Fragment 销毁的时候会调用 RefWatcher.watch 来观察这些对象，如果这些对象得不到释放那么就会一直保存在 retainedReferences 中，当 App 切到后台一定时间后或者保留的引用超过了临界值，那么就会手动的触发一个 GC，如果触发后条件仍然满足，那么就会去生成内存堆转储快照，然后开启服务去分析该文件，找出泄露的对象，并给出引用链。

## 源码分析

首先根据上篇文章我们知道在 InternalLeakSentry.init 代码块中会将 InternalLeakCanary 实例作为 LeakSentry 的监听器，在 InternalLeakSentry.install 的最后回调了InternalLeakCanary.onLeakSentryInstalled。

### InternalLeakCanary.onLeakSentryInstalled

代码如下：

```kotlin
override fun onLeakSentryInstalled(application: Application) {
    this.application = application
    val heapDumper = AndroidHeapDumper(application, leakDirectoryProvider)
    // GC触发器
    val gcTrigger = GcTrigger.Default
    val configProvider = { LeakCanary.config }
    val handlerThread = HandlerThread(HeapDumpTrigger.LEAK_CANARY_THREAD_NAME)
    handlerThread.start()
    val backgroundHandler = Handler(handlerThread.looper)
    heapDumpTrigger = HeapDumpTrigger(
            application, backgroundHandler, LeakSentry.refWatcher, gcTrigger, heapDumper, configProvider
    )
    application.registerVisibilityListener { applicationVisible ->
        this.applicationVisible = applicationVisible
        heapDumpTrigger.onApplicationVisibilityChanged(applicationVisible)
    }
    addDynamicShortcut(application)
}
```

方法内部主要构造了一个 HeapDumpTrigger 实例，然后调用了 registerVisibilityListener 这个扩展方法。

```kotlin
internal fun Application.registerVisibilityListener(listener: (Boolean) -> Unit) {
    registerActivityLifecycleCallbacks(VisibilityTracker(listener))
}
internal class VisibilityTracker(private val listener: (Boolean) -> Unit) :
        ActivityLifecycleCallbacksAdapter() {
    private var startedActivityCount = 0
    private var hasVisibleActivities: Boolean = false
    override fun onActivityStarted(activity: Activity) {
        startedActivityCount++
        if (!hasVisibleActivities && startedActivityCount == 1) {
            hasVisibleActivities = true
            listener.invoke(true)
        }
    }
    override fun onActivityStopped(activity: Activity) {
        if (startedActivityCount > 0) {
            startedActivityCount--
        }
        if (hasVisibleActivities && startedActivityCount == 0 && !activity.isChangingConfigurations) {
            hasVisibleActivities = false
            listener.invoke(false)
        }
    }
}
```

方法内部通过监听 Activity 声明周期，每次 onStart 的时候将可见 Activity 数量加1，如果可见数量为 1，并且原来是不可见的，那么就认为 App 从不可见转变到了可见。每次 onStop 的时候将可见 Activity 数量减1，如果可见数量变成了 0，并且原来是可见的，并且不是因为配置改变导致的(配置改变会立马又再次启动该 Activity)，那么就认为 App 从可见转变到了不可见。再看 onLeakSentryInstalled，在可见状态发生变化后会调用 onApplicationVisibilityChanged。

### heapDumpTrigger.onApplicationVisibilityChanged

代码如下：

```kotlin
fun onApplicationVisibilityChanged(applicationVisible: Boolean) {
    if (applicationVisible) {
        applicationInvisibleAt = -1L
    } else {
        applicationInvisibleAt = SystemClock.uptimeMillis()
        scheduleRetainedInstanceCheck("app became invisible", LeakSentry.config.watchDurationMillis)
    }
}
```

主要看看从可见变成不可见时的情况(App被切换到后台)，记录下应用程序开始不可见的时间。

```kotlin
private fun scheduleRetainedInstanceCheck(reason: String, delayMillis: Long) {
    if (checkScheduled) {
        return
    }
    checkScheduled = true
    backgroundHandler.postDelayed({
        checkScheduled = false
        checkRetainedInstances(reason)
    }, delayMillis)
}
```
在5秒(默认)后，会在子线程中执行 checkRetainedInstances 。

### heapDumpTrigger.checkRetainedInstances

代码如下：

```kotlin
private fun checkRetainedInstances(reason: String) {
    val config = configProvider()
    // 如果不需要分析堆转储文件就直接返回
    if (!config.dumpHeap) {
        return
    }
    // 获取到还没被释放的被观察者对象数量
    var retainedKeys = refWatcher.retainedKeys
    // 如果保留的对象数量等于0，或者保留的对象数量小于临界值并且App可见或者不可见但是不可见时间少于5秒就返回true，
    // 否则(保留对象数量大于临界值，或者不可见时间超过了5秒)就返回false
    if (checkRetainedCount(retainedKeys, config.retainedVisibleThreshold)) return
    // 如果配置了在debug时不转储堆文件，但是现在正在debug，那么显示一个通知，然后等20秒再试
    if (!config.dumpHeapWhenDebugging && DebuggerControl.isDebuggerAttached) {
        showRetainedCountWithDebuggerAttached(retainedKeys.size)
        scheduleRetainedInstanceCheck("debugger was attached", WAIT_FOR_DEBUG_MILLIS)
        return
    }
    // 运行一下GC，将那些只有弱引用引用着的对象回收
    gcTrigger.runGc()
    // 运行GC后，还保留的对象数量(这些对象极有可能发生泄漏)
    retainedKeys = refWatcher.retainedKeys
    // 如果不可见超过5秒还是会继续执行，否则运行了GC后发现保留对象数量已经少于临界值了那么就直接返回
    if (checkRetainedCount(retainedKeys, config.retainedVisibleThreshold)) return
    // 保存下retainedKeys，便于与后续进行比对
    HeapDumpMemoryStore.setRetainedKeysForHeapDump(retainedKeys)
    HeapDumpMemoryStore.heapDumpUptimeMillis = SystemClock.uptimeMillis()
    dismissNotification()
    // 生成堆转储快照
    val heapDumpFile = heapDumper.dumpHeap()
    if (heapDumpFile == null) {
        scheduleRetainedInstanceCheck("failed to dump heap", WAIT_AFTER_DUMP_FAILED_MILLIS)
        showRetainedCountWithHeapDumpFailed(retainedKeys.size)
        return
    }
    refWatcher.removeRetainedKeys(retainedKeys)
    // 开启服务分析堆转储文件
    HeapAnalyzerService.runAnalysis(application, heapDumpFile)
}
```

主要来看下 heapDumper.dumpHeap 和 HeapAnalyzerService.runAnalysis，heapDumper 是一个AndroidHeapDumper 实例。

```kotlin
override fun dumpHeap(): File? {
    // 内部就是创建了一个File并且保证.hprof文件不超过7个
    val heapDumpFile = leakDirectoryProvider.newHeapDumpFile() ?: return null
    val waitingForToast = FutureResult<Toast?>()
    // 展示一个toast表示正在生成堆转储文件
    showToast(waitingForToast)
    if (!waitingForToast.wait(5, SECONDS)) {
        return null
    }
    val notificationManager =
            context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
    if (Notifications.canShowNotification) {
        val dumpingHeap = context.getString(R.string.leak_canary_notification_dumping)
        val builder = Notification.Builder(context)
                .setContentTitle(dumpingHeap)
        val notification = Notifications.buildNotification(context, builder, LEAKCANARY_LOW)
        notificationManager.notify(R.id.leak_canary_notification_dumping_heap, notification)
    }
    val toast = waitingForToast.get()
    return try {
        // 获取堆转储快照写入heapDumpFile
        Debug.dumpHprofData(heapDumpFile.absolutePath)
        if (heapDumpFile.length() == 0L) {
            CanaryLog.d("Dumped heap file is 0 byte length")
            null
        } else {
            heapDumpFile
        }
    } catch (e: Exception) {
        null
    } finally {
        cancelToast(toast)
        notificationManager.cancel(R.id.leak_canary_notification_dumping_heap)
    }
}
```

成功生成堆转储文件后会调用 HeapAnalyzerService.runAnalysis 进行分析

```kotlin
internal class HeapAnalyzerService : ForegroundService(...), AnalyzerProgressListener {
    
    override fun onHandleIntentInForeground(intent: Intent?) {
        if (intent == null) {
            CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.")
            return
        }
        // 因为与应用进程运行在一起所以为了不影响App需要降低该子线程优先级
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)
        val heapDumpFile = intent.getSerializableExtra(HEAPDUMP_FILE_EXTRA) as File
        val heapAnalyzer = HeapAnalyzer(this)
        val config = LeakCanary.config
        val heapAnalysis =
                heapAnalyzer.checkForLeaks(
                        heapDumpFile, config.exclusionsFactory, config.computeRetainedHeapSize,
                        config.leakInspectors, config.labelers)
        // 分析完后，回调下，并且将堆转储文件删除
        try {
            config.analysisResultListener(application, heapAnalysis)
        } finally {
            heapAnalysis.heapDumpFile.delete()
        }
    }
    // 整个分析过程有15步之多，每走一步都会计算百分比更新下通知
    override fun onProgressUpdate(step: AnalyzerProgressListener.Step) {
        val percent = (100f * step.ordinal / AnalyzerProgressListener.Step.values().size).toInt()
        val lowercase = step.name.replace("_", " ")
                .toLowerCase()
        val message = lowercase.substring(0, 1).toUpperCase() + lowercase.substring(1)
        showForegroundNotification(100, percent, false, message)
    }
     companion object {
        private const val HEAPDUMP_FILE_EXTRA = "HEAPDUMP_FILE_EXTRA"
        fun runAnalysis(
                context: Context,
                heapDumpFile: File) {
            val intent = Intent(context, HeapAnalyzerService::class.java)
            intent.putExtra(HEAPDUMP_FILE_EXTRA, heapDumpFile)
            ContextCompat.startForegroundService(context, intent)
        }
    }
}
```

可以看到主要的逻辑都在 heapAnalyzer.checkForLeaks 中了。

```kotlin
fun checkForLeaks(*): HeapAnalysis {
    val analysisStartNanoTime = System.nanoTime()
    listener.onProgressUpdate(READING_HEAP_DUMP_FILE)
    try {
        HprofParser.open(heapDumpFile)
                .use { parser ->
                    listener.onProgressUpdate(SCANNING_HEAP_DUMP)
                    // 找出所有的keyedWeakReference引用
                    val (gcRootIds, keyedWeakReferenceInstances, cleaners) = scan(
                            parser, computeRetainedHeapSize
                    )
                    val analysisResults = mutableMapOf<String, RetainedInstance>()
                    listener.onProgressUpdate(FINDING_WATCHED_REFERENCES)
                    // 获取在堆转储文件生成前保存在store里面的内容，为什么不直接调用?
                    // 可能考虑现在时刻数据已经发生变化
                    val (retainedKeys, heapDumpUptimeMillis) = readHeapDumpMemoryStore(parser)
                    // 找出发生泄露的引用
                    val leakingWeakRefs =
                            findLeakingReferences(
                                    parser, retainedKeys, analysisResults, keyedWeakReferenceInstances,
                                    heapDumpUptimeMillis
                            )
                     // 找出泄露引用的最短引用链
                    val (pathResults, dominatedInstances) =
                            findShortestPaths(
                                    parser, exclusionsFactory, leakingWeakRefs, gcRootIds,
                                    computeRetainedHeapSize
                            )
                    // 计算该泄露引用的大小 
                    val retainedSizes = if (computeRetainedHeapSize) {
                        computeRetainedSizes(parser, pathResults, dominatedInstances, cleaners)
                    } else {
                        null
                    }
                    buildLeakTraces(
                            reachabilityInspectors, labelers, pathResults, parser,
                            leakingWeakRefs, analysisResults, retainedSizes
                    )
                    addRemainingInstancesWithNoPath(parser, leakingWeakRefs, analysisResults)
                    return HeapAnalysisSuccess(
                            heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime),
                            analysisResults.values.toList()
                    )
                }
    } catch (exception: Throwable) {
        return HeapAnalysisFailure(
                heapDumpFile, System.currentTimeMillis(), since(analysisStartNanoTime),
                HeapAnalysisException(exception)
        )
    }
}
```

当分析堆转储文件完毕后会回到 onHandleIntentInForeground 继续执行 analysisResultListener。

### LeakCanary.config.analysisResultListener

代码如下：

```kotlin
object DefaultAnalysisResultListener : AnalysisResultListener {
    override fun invoke(
            application: Application,
            heapAnalysis: HeapAnalysis) {
    val movedHeapDump = renameHeapdump(heapAnalysis.heapDumpFile)
    val updatedHeapAnalysis = when (heapAnalysis) {
        is HeapAnalysisFailure -> heapAnalysis.copy(heapDumpFile = movedHeapDump)
        is HeapAnalysisSuccess -> heapAnalysis.copy(heapDumpFile = movedHeapDump)
    }
    val (id, groupProjections) = LeaksDbHelper(application)
            .writableDatabase.use { db ->
        val id = HeapAnalysisTable.insert(db, updatedHeapAnalysis)
        id to LeakingInstanceTable.retrieveAllByHeapAnalysisId(db, id)
    }
    val (contentTitle, screenToShow) = ...
    val pendingIntent = LeakActivity.createPendingIntent(
            application, arrayListOf(GroupListScreen(), HeapAnalysisListScreen(), screenToShow)
    )
    val contentText = application.getString(R.string.leak_canary_notification_message)
    Notifications.showNotification(
            application, contentTitle, contentText, pendingIntent,
            R.id.leak_canary_notification_analysis_result,
            LEAKCANARY_RESULT
    )
}
```

方法内部主要就是保存数据到数据库，显示一个 Notification，点击后跳转到 LeakActivity 显示 Screens 的内容。接着上篇文章最后还说到了当 RefWatcher 内部往 retainedReferences 内部添加一个对象时都会回调 onReferenceRetained，这做的事情跟可见度从可见变成不可见的事情一样。

## 总结

LeakSentry 默认使用 RefWatcher 观察将要被销毁的 Activity、Fragment(用户也可自定义观察对象)，在被观察5秒后如果对象还没被回收，并且保留的对象数量大于临界值或者应用不可见超过5秒那么就会手动运行一下 GC，运行后如果还满足那么会生成内存堆转储快照，然后开启一个前台服务( IntentService )去解析该快照，然后将解析结果与原先被观察但还没回收的对象进行对比，找出真正泄露的对象，比给出最短引用链，当 App 从可见到不可见时也同样会触发该逻辑。