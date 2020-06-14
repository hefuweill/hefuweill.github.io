---
title: RxJava2 线程切换原理
date: 2019-05-15 12:35:32
type: "Android"
---

## 前言

RxJava 的一个重要优点就在于可以方便的切换线程，所以就想从源码的角度探索下其切换线程的原理。<!-- more -->

## ObserveOn

ObserveOn 用于切换下游执行线程，可以多次调用，每调用一次会切换一次，先来看一个例子。

```java
fun threadName(desc: String) {
    println("$desc ${Thread.currentThread().name}")
}
fun main() {
    Observable.create<Int> {
        threadName("subscribe")
        it.onNext(1)
        it.onNext(2)
        it.onComplete()
    }.observeOn(Schedulers.io())
        .subscribe(object : Observer<Int> {
            override fun onComplete() {
                threadName("onComplete")
            }
            override fun onSubscribe(d: Disposable) {
                threadName("onSubscribe")
            }
            override fun onError(e: Throwable) {
                threadName("onError")
            }
            override fun onNext(t: Int) {
                threadName("onNext")
            }
        })
}
```

输出结果

```
onSubscribe main
subscribe main
onNext RxCachedThreadScheduler-1
onNext RxCachedThreadScheduler-1
onComplete RxCachedThreadScheduler-1
```

说好的 `observeOn` 切换下游执行线程，怎么 `onSubscribe` 方法会在主线程中调用？原因是 `observeOn` 方法生成的 ObserveOnObserver 实例并不会对 `onSubscribe` 事件做切换线程的操作，这个等下看了源码就理解了。那么 `observeOn` 是怎么把下游的 `onNext`、`onComplete `切换到子线程执行的呢？来看看 `observeOn` 的源码实现。

```java
// Observable.java
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError) {
    return observeOn(scheduler, delayError, bufferSize());
}
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```

`observeOn  `方法调用后会返回一个 ObservableObserveOn 实例，经过上篇文章的分析主要关注其  `subscribeActual    ` 方法就行。

```java
protected void subscribeActual(Observer<? super T> observer) {
    Scheduler.Worker w = scheduler.createWorker();
    source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
}
```

包装了一下下游 Observer，可以猜测这个 Observer 内部会将 `onNext` 等事件转到其它线程进行执行。

```java
public void onSubscribe(Disposable d) {
    ...
    downstream.onSubscribe(this);
}
public void onNext(T t) {
    queue.offer(t);
    schedule();
}
public void onError(Throwable t) {
    error = t;
    schedule();
}
public void onComplete() {
    schedule();
}
void schedule() {
    worker.schedule(this);
}
```

可以看到 `onSubscribe` 直接在当前线程执行了没有进行线程切换，`onNext`、`onError`、`onComplete ` 则是都调用了 `schedule` 方法。

```java
public Disposable schedule(@NonNull Runnable run) {
    return schedule(run, 0L, TimeUnit.NANOSECONDS);
}
public abstract Disposable schedule(@NonNull Runnable run, long delay, @NonNull TimeUnit unit);
// EventLoopWorker.java
public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
    return threadWorker.scheduleActual(action, delayTime, unit, tasks);
}
// NewThreadWorker.java
public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
    Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
    ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);
    Future<?> f;
    try {
        if (delayTime <= 0) {
            f = executor.submit((Callable<Object>)sr);
        } else {
            f = executor.schedule((Callable<Object>)sr, delayTime, unit);
        }
        sr.setFuture(f);
    } catch (RejectedExecutionException ex) {
        if (parent != null) {
            parent.remove(sr);
        }
        RxJavaPlugins.onError(ex);
    }
    return sr;
}
```

executor 是一个 SchedulerThreadPoolExecutor 实例，最终都会在线程池中运行那个 Runnable 实例也就是ObserveOnObserver 实例，所以就看看其 `run` 方法。

```java
public void run() {
    if (outputFused) {
        drainFused();
    } else {
        drainNormal();
    }
}
void drainNormal() {
    for (;;) {
        for (;;) {
            try {
                v = q.poll();
            } catch (Throwable ex) {
                a.onError(ex);
                return;
            }
            a.onNext(v);
        }
    }
}
void drainFused() {
    for (;;) {
        if (d) {
            disposed = true;
            ex = error;
            if (ex != null) {
                downstream.onError(ex);
            } else {
                downstream.onComplete();
            }
            worker.dispose();
            return;
        }
    }
}
```

可以看到 `onNext`、`onError`、`onComplete `都在这个方法里面调用，因此这些方法就运行在了线程池中了，这样就成功的切换了线程，那么多次调用 observeOn 有效果吗？多次调用其实就是在一个线程池中的某个线程中再次开启了一个线程，所以是有效果的。接着看看 `subscribeOn` 这个方法。

## subscribeOn

subscribeOn 用于上游执行线程，并且多次调用只有第一次会生效，先来看一个例子。

```kotlin
fun main() {
    Observable.create<Int> {
        threadName("subscribe")
        it.onNext(1)
        it.onNext(2)
        it.onComplete()
    }.subscribeOn(Schedulers.io())
    .subscribe(object : Observer<Int> {
        override fun onComplete() {
            threadName("onComplete")
        }
        override fun onSubscribe(d: Disposable) {
            threadName("onSubscribe")
        }
        override fun onError(e: Throwable) {
            threadName("onError")
        }
        override fun onNext(t: Int) {
            threadName("onNext")
        }
    })
}
```
输出结果

```
onSubscribe main
subscribe RxCachedThreadScheduler-1
onNext RxCachedThreadScheduler-1
onNext RxCachedThreadScheduler-1
onComplete RxCachedThreadScheduler-1
```

咦！不是说 subscribeOn 切换的只是上游的执行线程，为什么 `onNext`、`onComplete `也会在子线程中执行？其实答案很简单该段代码中没有调用 observeOn 所以下游执行线程并没有发生改变，因此上游在子线程中发送一个 `onNext` 事件过来，下游的 `onNext` 方法自然也会在子线程中执行，那么 `subscribeOn` 内部到底做了什么才会导致上游会在子线程中执行呢，来看看其源码实现。

```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```

创建了一个 `ObservableSubscribeOn` 实例并将 `Scheduler` 实例传入，接着看看其 `subscribeActual` 实现。

```java
// ObservableSubscribeOn.java
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);
    observer.onSubscribe(parent);
    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```

这里直接回调了下游 `Observer` 实例的 `onSubscribe` 方法，接着执行 `scheduleDirect`，继续跟进。

```java
// Schedule.java
public Disposable scheduleDirect(@NonNull Runnable run) {
    return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    // createWorker是抽象方法，IoScheduler会返回一个EventLoopWorker实例
    final Worker w = createWorker();
    DisposeTask task = new DisposeTask(decoratedRun, w);
    w.schedule(task, delay, unit);
    return task;
}
public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
    return threadWorker.scheduleActual(action, delayTime, unit, tasks);
}
```

调用到了 `scheduleActual` 方法，这个方法上面在分析 `observeOn` 时已经分析过，内部会在线程池中执行 `action` 这个 `Runnable` 实例，那么主要看看 `SubscribeTask`的`run `方法。

```java
// SubscribeTask.java
public void run() {
    source.subscribe(parent);
}
```

里面的逻辑很简单就是在线程池中执行上游的 `subscribe` 方法，因此上游的所有事件都将在该线程池中执行。那么为什么说 `subscribeOn` 只能生效一次呢？其实真正的来说 `subscribeOn` 也可以生效多次，只不过最上游发送事件的线程是由第一次 `subscribeOn` 调用时确定的，举个例子。

```kotlin
fun main() {
    Observable.create<Int> {
        threadName("subscribe")
        it.onNext(1)
        it.onNext(2)
        it.onComplete()
    }.subscribeOn(Schedulers.io())
        .map { threadName("map"); it + 1 }
        .subscribeOn(Schedulers.computation())
        .subscribe(object : Observer<Int> {
            override fun onComplete() {
                threadName("onComplete")
            }
            override fun onSubscribe(d: Disposable) {
                threadName("onSubscribe")
            }
            override fun onError(e: Throwable) {
                threadName("onError")
            }
            override fun onNext(t: Int) {
                threadName("onNext")
            }
        })
    // 主线程睡眠下，防止RxJava生成的daemon线程自动退出
    Thread.sleep(200)
}
```

输出结果

```
onSubscribe main
subscribe RxCachedThreadScheduler-1
map RxCachedThreadScheduler-1
onNext RxCachedThreadScheduler-1
map RxCachedThreadScheduler-1
onNext RxCachedThreadScheduler-1
onComplete RxCachedThreadScheduler-1
```

在这里例子中调用了两次 `subscribeOn` 但是看起来只有第一次才生效，其实 `map` 方法生成的 `ObservableMap` 实例的 `subscribe` 方法是在计算线程池中执行的，下面来看一下 `observeOn` 和 `subscribeOn` 进行组合的情况。

## observeOn、subscribeOn 组合

假设有这么一个需求: 先注册然后登录，需要满足：1. 注册成功后要能更新UI，2. 注册失败将不再进行登录，3. 登录成功或者失败也需要更新UI。由于网络请求不能在主线程执行，因此我们就需要用到线程切换，下面是示例代码。

```kotlin
private var disposable: Disposable? = null
private fun registerAndLogin(listener: Listener) {
    getRegisterObservable()
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .doOnNext {
                threadName("doOnNext")
                if (!it.success) {
                    listener.registerFail()
                    disposable?.dispose()
                } else {
                    listener.registerSuccess()
                }
            }
            .observeOn(Schedulers.io())
            .flatMap {
                getLoginObservable()
            }
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(object : Observer<LoginModel> {
                override fun onComplete() {
                    threadName("onComplete")
                }
                override fun onSubscribe(d: Disposable) {
                    disposable = d
                    threadName("onSubscribe")
                }
                override fun onNext(model: LoginModel) {
                    if (model.success) listener.loginSuccess(model.token)
                    else listener.loginFail()
                    threadName("onNext")
                }
                override fun onError(e: Throwable) {}
            })
}
private fun getRegisterObservable(): Observable<RegisterModel> {
    return Observable.create {
        // 模拟网络耗时
        Thread.sleep(500)
        threadName("register")
        it.onNext(RegisterModel(true))
        it.onComplete()
    }
}
private fun getLoginObservable(): Observable<LoginModel> {
    return Observable.create {
        // 模拟网络耗时
        Thread.sleep(500)
        threadName("login")
        it.onNext(LoginModel(true, "token"))
        it.onComplete()
    }
}
data class RegisterModel(val success: Boolean)
data class LoginModel(val success: Boolean, val token: String)
interface Listener {
    fun registerSuccess()
    fun registerFail()
    fun loginSuccess(token: String)
    fun loginFail()
}
private fun threadName(desc: String) {
    Log.d("Thread", "$desc ${Thread.currentThread().name}")
}
```

输出结果如下

```
onSubscribe main
register RxCachedThreadScheduler-1
doOnNext main
login RxCachedThreadScheduler-2
onNext main
onComplete main
```

这个输出完全满足了我们的需求，网络请求在子线程，UI更新在主线程，现在来分析下为什么会是这个结果。

* 首先 `subscribeOn` 这个方法在这条链上哪个地方调用都没关系其并不会影响结果，因为其只是决定了注册操作所在的线程。
* 第一次更新UI是在 `doOnNext` 中，我们知道注册操作是在子线程，所以我们这里要使用 `observeOn` 将线程切换到主线程。
* 当UI更新完毕后我们要进行登录操作，网络操作需要在子线程，所以我们这里要使用 `observeOn` 将线程再次切换到子线程。
* 当登录完成后我们又需要更新UI，所以我们这里要使用 `observeOn` 将线程切换到主线程。

