---
title: LeakCanary2-LeakSentry 源码分析
date: 2019-06-18 14:12:10
type: "Android"
---



## 前言

LeakCanary 是 Square 公司为 Android 开发者提供的一个自动检测内存泄漏的工具，在4月23日推出了2.0预览版，更新内容见 [Github](https://github.com/square/leakcanary)，其中新增了一个 LeakSentry 库，该库作为 LeakCanary 的开关，可以实时查看那些被观察的对象是否可能内存泄露，并且可以独立引入使用，下面从源码角度来分析下该库，本文基于 2.0-alpha-2。<!-- more -->

## 源码分析

### LeakSentryInstaller.onCreate

首先查看其 manifest 文件，发现其中注册了一个名叫 LeakSentryInstaller 的 ContentProvider，因此其onCreate 方法会在 Application.onCreate 之前得到执行，原因如下：

```java
// ActivityThread.java
private void handleBindApplication() {
    ...
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    if (!ArrayUtils.isEmpty(data.providers)) {
        installContentProviders(app, data.providers);
    }
    mInstrumentation.callApplicationOnCreate(app);
}
private void installContentProviders(...) {
    installProvider(...);
}
private IActivityManager.ContentProviderHolder installProvider(...) {
    ContentProvider localProvider = null;
    localProvider = (ContentProvider)cl.loadClass(info.name).newInstance();
    localProvider.attachInfo(c, info);
}
// ContentProvider.java
public void attachInfo(Context context, ProviderInfo info) {
    attachInfo(context, info, false);
}
private void attachInfo(...) {
    ...
    ContentProvider.this.onCreate();
}
```

搞清楚了入口以后，看看 LeakSentryInstaller.onCreate 都干了些什么。

```kotlin
// LeakSentryInstaller.kt
override fun onCreate(): Boolean {
    CanaryLog.logger = DefaultCanaryLog()
    val application = context!!.applicationContext as Application
    InternalLeakSentry.install(application)
    return true
}
internal object InternalLeakSentry {
    init {
        listener = try {
            val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
            leakCanaryListener.getDeclaredField("INSTANCE").get(null) as LeakSentryListener
        } catch (ignored: Throwable) {
            LeakSentryListener.None
        }
    }
    fun install(application: Application) {
        ...
        InternalLeakSentry.application = application
        // 这里要使用Lambda包装的原因是，想要在使用的时候才去初始化config
        val configProvider = { LeakSentry.config }
        ActivityDestroyWatcher.install(application, refWatcher, configProvider)
        FragmentDestroyWatcher.install(application, refWatcher, configProvider)
        listener.onLeakSentryInstalled(application)
    }
}
object LeakSentry {
    data class Config(
            val enabled: Boolean = InternalLeakSentry.isDebuggableBuild,
            val watchActivities: Boolean = true,
            val watchFragments: Boolean = true,
            val watchFragmentViews: Boolean = true,
            val watchDurationMillis: Long = TimeUnit.SECONDS.toMillis(5))
    @Volatile
    var config: Config = if (isInstalled) Config() else Config(enabled = false)
    val refWatcher
        get() = InternalLeakSentry.refWatcher
    val isInstalled
        get() = InternalLeakSentry.isInstalled
    fun manualInstall(application: Application) = InternalLeakSentry.install(application)
}
```

onCreate 内部调用了 InternalLeakSentry.install，在其 init 代码块设置了监听器当 LeakCanary 被依赖时会设置成 InternalLeakCanary 实例，不然就是一个空实现，install 内部主要就是调用了两个 Watch.install。

### ActivityDestroyWatcher.install

代码如下：

```kotlin
// ActivityDestroyWatcher.kt
internal class ActivityDestroyWatcher private constructor(
        private val refWatcher: RefWatcher,
        private val configProvider: () -> Config) {
    private val lifecycleCallbacks = object : ActivityLifecycleCallbacksAdapter() {
        override fun onActivityDestroyed(activity: Activity) {
            if (configProvider().watchActivities) {
                refWatcher.watch(activity)
            }
        }
    }
    companion object {
        fun install(
                application: Application,
                refWatcher: RefWatcher,
                configProvider: () -> Config) {
            val activityDestroyWatcher =
                    ActivityDestroyWatcher(refWatcher, configProvider)
            	                    application.registerActivityLifecycleCallbacks(activityDestroyWatcher.lifecycleCallbacks)
        }
    }
}
```

内部注册了一个 Activity 生命周期回调，当 onDestory 事件来临时由于默认的 config.watchActivities = true ，因此 refWatcher.watch 将被调用。

```kotlin
class RefWatcher constructor(
        private val clock: Clock,
        private val checkRetainedExecutor: Executor,
        private val onReferenceRetained: () -> Unit,
        private val isEnabled: () -> Boolean = { true }) {

    private val watchedReferences = mutableMapOf<String, KeyedWeakReference>()
    private val retainedReferences = mutableMapOf<String, KeyedWeakReference>()
    private val queue = ReferenceQueue<Any>()

    @Synchronized
    fun watch(watchedReference: Any) {
        watch(watchedReference, "")
    }
    @Synchronized
    fun watch(
            watchedReference: Any,
            referenceName: String) {
        if (!isEnabled()) {
            return
        }
        removeWeaklyReachableReferences()
        val key = UUID.randomUUID()
                .toString()
        val watchUptimeMillis = clock.uptimeMillis()
        val reference =
                KeyedWeakReference(watchedReference, key, referenceName, watchUptimeMillis, queue)
        watchedReferences[key] = reference
        checkRetainedExecutor.execute {
            moveToRetained(key)
        }
    }
    @Synchronized
    private fun moveToRetained(key: String) {
        removeWeaklyReachableReferences()
        val retainedRef = watchedReferences.remove(key)
        if (retainedRef != null) {
            retainedReferences[key] = retainedRef
            onReferenceRetained()
        }
    }
    @Synchronized
    fun removeRetainedKeys(keysToRemove: Set<String>) {
        retainedReferences.keys.removeAll(keysToRemove)
    }
    @Synchronized
    fun clearWatchedReferences() {
        watchedReferences.clear()
        retainedReferences.clear()
    }
    private fun removeWeaklyReachableReferences() {
        var ref: KeyedWeakReference?
        do {
            ref = queue.poll() as KeyedWeakReference?
            if (ref != null) {
                val removedRef = watchedReferences.remove(ref.key)
                if (removedRef == null) {
                    retainedReferences.remove(ref.key)
                }
            }
        } while (ref != null)
    }
}
```

watch 方法主要分为如下几步：
1. 当 isEnabled 被设置成 false （release 包），那么 watch 将会直接返回，该值取自于 config 。
2. 移除掉那些已经在队列里面的弱引用，一开始肯定没有先跳过。
3. 创建一个 KeyedWeakReference 实例将观察对象这里是 onDestroy 后的 Activity 实例，传入 queue，当被观察的对象被回收掉以后会将弱引用对象放入该队列中去。
4. 将弱引用对象放入 watchedReferences 中。
5. 延迟一定时间后(默认为5秒，取自于config)调用 moveToRetained。

moveToRetained 方法主要分为如下几步：

1. 首先调用了 removeWeaklyReachableReferences ，该方法将那些已经放到队列中的弱引用对象从 watchedReferences 中移除(存在队列中就表示该对象已经被回收了不可能发生内存泄露)，后面的retainedReferences.remove 是为了将上次保留的对象删除(因为本次它已经被回收了)。
2. 接着如果 retainedRef!=null，那么就说明当前对象还没有被回收，就加入到 retainedReferences 中，并回调listener.onReferenceRetained。

方法主要是统计哪些应该要被回收的对象，但是却还没有被回收。

注：retainedReferences 保存经过检查还没被回收的对象，watchedReference 保存正在检测的对象。

### FragmentDestroyWatcher.install

代码如下：

```kotlin
internal interface FragmentDestroyWatcher {
    fun watchFragments(activity: Activity)
    companion object {
        private const val SUPPORT_FRAGMENT_CLASS_NAME = "androidx.fragment.app.Fragment"
        fun install(
                application: Application,
                refWatcher: RefWatcher,
                configProvider: () -> LeakSentry.Config) {
            val fragmentDestroyWatchers = mutableListOf<FragmentDestroyWatcher>()
            if (SDK_INT >= O) {
                fragmentDestroyWatchers.add(AndroidOFragmentDestroyWatcher(refWatcher, configProvider))
            }
            if (classAvailable(SUPPORT_FRAGMENT_CLASS_NAME)) {
                fragmentDestroyWatchers.add(SupportFragmentDestroyWatcher(refWatcher, configProvider))
            }
            if (fragmentDestroyWatchers.size == 0) {
                return
            }
            application.registerActivityLifecycleCallbacks(object : ActivityLifecycleCallbacksAdapter() {
                override fun onActivityCreated(
                        activity: Activity,
                        savedInstanceState: Bundle?) {
                    for (watcher in fragmentDestroyWatchers) {
                        watcher.watchFragments(activity)
                    }
                }
            })
        }
        private fun classAvailable(className: String): Boolean {
            return try {
                Class.forName(className)
                true
            } catch (e: ClassNotFoundException) {
                false
            }
        }
    }
}
```

首先 FragmentDestroyWatcher 只支持 Android8 及以上或者是 AndroidX ，还是监听了 Activity 生命周期不过这次监听 onCreate，先看看 AndroidOFragmentDestroyWatcher.watchFragments。

#### AndroidOFragmentDestroyWatcher.watchFragments

代码如下：

```kotlin
internal class AndroidOFragmentDestroyWatcher(
        private val refWatcher: RefWatcher,
        private val configProvider: () -> Config
) : FragmentDestroyWatcher {
    private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {
        override fun onFragmentViewDestroyed(
                fm: FragmentManager,
                fragment: Fragment) {
            val view = fragment.view
            if (view != null && configProvider().watchFragmentViews) {
                refWatcher.watch(view)
            }
        }
        override fun onFragmentDestroyed(
                fm: FragmentManager,
                fragment: Fragment) {
            if (configProvider().watchFragments) {
                refWatcher.watch(fragment)
            }
        }
    }
    override fun watchFragments(activity: Activity) {
        val fragmentManager = activity.fragmentManager
        fragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
    }
}
```

内部注册了 Fragment 声明周期回调，由于该方法只有在 API26 及以上才有，因此前面判断了 Android O，接着当 Fragment 被销毁后会分别调用 refWatcher.watch(view)、refWatcher.watch(fragment)，这里面的逻辑上文已经说过了。

### SupportFragmentDestroyWatcher#watchFragments

代码如下：

```kotlin
internal class SupportFragmentDestroyWatcher(
        private val refWatcher: RefWatcher,
        private val configProvider: () -> Config) : FragmentDestroyWatcher {
    private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {
        override fun onFragmentViewDestroyed(
                fm: FragmentManager,
                fragment: Fragment) {
            val view = fragment.view
            if (view != null && configProvider().watchFragmentViews) {
                refWatcher.watch(view)
            }
        }
        override fun onFragmentDestroyed(
                fm: FragmentManager,
                fragment: Fragment) {
            if (configProvider().watchFragments) {
                refWatcher.watch(fragment)
            }
        }
    }
    override fun watchFragments(activity: Activity) {
        if (activity is FragmentActivity) {
            val supportFragmentManager = activity.supportFragmentManager
            supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
        }
    }
}
```

内部逻辑与 AndroidOFragmentDestroyWatche r一样，只是获取的是 supportFragmentManager。

## 总结

根据源码可以看出 LeakSentry 内部使用弱引用保存被观察对象，在指定时间后如果还未被回收就可以认为该对象可能发生内存泄露，其默认会自动观察销毁的 Activity、fragment 及其内部 View，同时也可以手动使用 `LeakSentry.refWatcher.watch(obj)` 来观察任意不再需要的对象，然后在代码中通过调用 `LeakSentry.refWatcher.retainedKeys.size` 查看还有几个被观察对象还未被释放。

