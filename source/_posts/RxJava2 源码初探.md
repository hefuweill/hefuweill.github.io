---
title: RxJava2 源码初探
date: 2019-05-14 18:22:13
type: "Android"
---



## 前言

众所周知RxJava有许多优点比如强大的链式调用，方便的线程调度，但是我对其原理还是了解的太少了，因此打算阅读下源码，先从一个最基本的例子开始。注：[仓库地址](https://github.com/ReactiveX/RxJava) <!-- more -->

## 例子

以下代码只是为了示例，正常情况下不会这么写。

```java
fun main() {
    Observable.create {
        emitter: ObservableEmitter<Int> ->
        println("onSourceSubscribe")
        emitter.onNext(1)
        emitter.onNext(2)
        emitter.onNext(3)
        emitter.onComplete()
    }.subscribe(object : Observer<Int> {
        override fun onComplete() {
            println("onComplete")
        }
        override fun onSubscribe(d: Disposable) {
            println("onObserverSubscribe")
        }
        override fun onNext(t: Int) {
            println("onNext $t")
        }
        override fun onError(e: Throwable) {
            println("onError")
        } 
    })
}
```

输出结果:

```
onObserverSubscribe
onSourceSubscribe
onNext 1
onNext 2
onNext 3
onComplete
```

那么为什么会按这个顺序输出呢？从代码中也可以看出从始至终也只调用了 create、subscribe 两个方法，先来看看 create 的源码。

```kotlin
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```

可以看出当我们没有设置 onObservableAssembly 时其实就是直接创建了一个 ObservableCreate 实例返回，接着看看 subscribe 。

```java
public final void subscribe(Observer<? super T> observer) {
    subscribeActual(observer);
}
```

可以看到内部就是调用了 subscribeActual 方法，而这个方法是个抽象方法，ObservableCreate 实现了该方法。

```java
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);
    source.subscribe(parent);
}
```

内部主要就是先创建了一个 CreateEmitter 实例，然后调用 observer.onSubscribe 方法，最后再调用 source.subscribe 方法，这就解释了 onObserverSubscribe 和 onSourceSubscribe 的输出，而 source 的subscribe 方法又调用了三次 onNext 方法和一次 onComplete 方法，先看看 onNext。

```java
// CreateEmitter.java
public void onNext(T t) {
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    if (!isDisposed()) {
        observer.onNext(t);
    }
}
```

如果还没 dispose 那么直接就调用了 observer.onNext，这也就解释了 onNext 1、onNext 2、onNext 3 三个输出接着看 onComplete 。

```java
// CreateEmitter.java
public void onComplete() {
    if (!isDisposed()) {
        try {
            observer.onComplete();
        } finally {
            dispose();
        }
    }
}
```

如果还没 dispose 就直接调用 observer.onComplete ，这就解释了 onComplete 的输出，并且最后还会执行 dispose ，此外注意到 Observer还有一个 onError 回调，该方法可以通过调用 emitter.onError 手动触发。

```java
// CreateEmitter.java
public void onError(Throwable t) {
    if (!tryOnError(t)) {
        RxJavaPlugins.onError(t);
    }
}
public boolean tryOnError(Throwable t) {
    if (t == null) {
        t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
    }
    if (!isDisposed()) {
        try {
            observer.onError(t);
        } finally {
            dispose();
        }
        return true;
    }
    return false;
}
```

可以看到当还没被 dispose 就会调用到 observer.onError 方法，至此这个基本 demo 的源码已经分析完毕。
总结下上述代码其实就分为如下几步：

1. 创建 Observable 实例。
2. 调用 observable.subscribeActual 方法。
3. 调用 observer.onSubscribe 方法。
4. 调用 source.subscribe方法。
5. 上述的 subscribe 方法内部可以执行若干次 onNext ，最多一次 onError、onComplete。

下面来从源码的角度研究研究 RxJava 中的几个基本方法。

## 基本方法

首先从最基本的 map 方法开始。

### map

map 方法用于对上游事件进行一次转化，如下图所示：

![](/Users/hefuwei/GitHub/blog/source/_posts/RxJava2 源码初探/map.png)

示例代码如下所示：

```kotlin
fun main() {
    Observable.create {
        emitter: ObservableEmitter<Int> ->
        println("onSourceSubscribe")
        emitter.onNext(1)
        emitter.onNext(2)
        emitter.onNext(3)
        emitter.onComplete()
    }
    .map {
       it + 1
    }.subscribe(object : Observer<Int> {
        override fun onComplete() {
            println("onComplete")
        }
        override fun onSubscribe(d: Disposable) {
            println("onObserverSubscribe")
        }
        override fun onError(e: Throwable) {
            println("onError")
        }
        override fun onNext(t: Int) {
            println("onNext $t")
        }
    })
}
```

输出结果：

```
onObserverSubscribe
onSourceSubscribe
onNext 2
onNext 3
onNext 4
onComplete
```

很明显 map 方法会对所有的 next 的数据做一次变化这里是加1，接着看看 map 的源码实现：

```java
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```

内部创建了一个 ObservableMap 实例并将当前的 Observable 实例和 Function 实例传入，根据本文一开始的分析当调用 Observable.subscribe 方法其实会调用 subscribeActual 方法。

```java
// ObservableMap.java
public void subscribeActual(Observer<? super U> t) {
    source.subscribe(new MapObserver<T, U>(t, function));
}
```

创建了 MapObserver 实例将 Observer 实例进行包装然后调用 source.subscribe，这个 source 其实就是上一级 Observable 实例本例中对应 ObservableCreate 实例，接着根据上文的分析会调用该 MapObserver 实例的onNext 三次然后调用一次 onComplete 。

```java
// MapObserver.java
public void onNext(T t) {
    // 初始化的时候done为false，直到complete或者error变为false
    if (done) {
        return;
    }
    U v;
    try {
        v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
    } catch (Throwable ex) {
        fail(ex);
        return;
    }
    downstream.onNext(v);
}
```

可以看到内部调用了 mapper.apply 方法，接着将拿到的结果当做参数调用 downstream.onNext 方法，注意这里的 downStream 就是外界创建的一个 Observer 对象，因此 map 就是通过代理下游 Observer 实例完成数据转换，**使用 Kotlin 重写 map 方法来加深下理解**，代码如下：

```kotlin
fun <T, R> Observable<T>.newMap(converter: (T) -> R): Observable<R> {
    return object : Observable<R>() {
        override fun subscribeActual(observer: Observer<in R>?) {
            requireNotNull(observer)
            this@newMap.subscribe(object : Observer<T> {
                private var done = false
                override fun onSubscribe(d: Disposable) {
                    observer.onSubscribe(d)
                }
                override fun onNext(t: T) {
                    if (!done) {
                        observer.onNext(converter.invoke(t))
                    }
                }
                override fun onError(e: Throwable) {
                    observer.onError(e)
                    done = true
                }
                override fun onComplete() {
                    observer.onComplete()
                    done = true
                }
            })
        }
    }
}
```

newMap 新建一个 Observable 实例假设叫 mapObservable 返回，下游会调用 mapObservable.subscribe ，因此 mapObservable.subscribeActual 会被执行，在这里面新建一个 Observer 实例假设叫 mapObserver，再去调用上游 observable.subscribe 传入 mapObserver，这样从下到上一条线就串起来了，接着上游执行 onNext ，mapObserver.onNext 得到执行，将值进行转化后调用下游 observer.onNext ，onComplete、onError 也就类似。

### flapMap

flatMap 方法用于将上游的每一个 onNext 事件都转换成一个 Observable 实例，然后将这些个 Observable 事件合并起来传递给下游。

![](/Users/hefuwei/GitHub/blog/source/_posts/RxJava2 源码初探/flatMap.png)

```java
fun main() {
    Observable.create {
        emitter: ObservableEmitter<Int> ->
        println("onSourceSubscribe")
        emitter.onNext(1)
        emitter.onNext(2)
        emitter.onNext(3)
        emitter.onComplete()
    }
    .flatMap {
        Observable.just(it, it + 1)
    }
    .subscribe(object : Observer<Int> {
        override fun onComplete() {
            println("onComplete")
        }

        override fun onSubscribe(d: Disposable) {
            println("onObserverSubscribe")
        }

        override fun onError(e: Throwable) {
            println("onError")
        }

        override fun onNext(t: Int) {
            println("onNext $t")
        }
    })
}
```

输出结果: 

```
onObserverSubscribe
onSourceSubscribe
onNext 1
onNext 2
onNext 2
onNext 3
onNext 3
onNext 4
onComplete
```

很显然 flatMap 将每一个事件比如 1 转换成一个拥有1、2 两个事件的 Observable 实例，来看看其源码实现。

```java
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper) {
    return flatMap(mapper, false);
}
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper, boolean delayErrors) {
    return flatMap(mapper, delayErrors, Integer.MAX_VALUE);
}
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper, boolean delayErrors, int maxConcurrency) {
    return flatMap(mapper, delayErrors, maxConcurrency, bufferSize());
}
public final <R> Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper,
            boolean delayErrors, int maxConcurrency, int bufferSize) {
    return RxJavaPlugins.onAssembly(new ObservableFlatMap<T, R>(this, mapper, delayErrors, maxConcurrency, bufferSize));
}
```

默认 delayErrors 为 false 表示当一个事件出现异常就会停止整个事件序列，默认并发数为 Int 的最大值，默认缓存大小为 128，然后根据这些参数和当前 Observable 实例构建出一个 ObservableFlatMap 实例，看看其subscribeActual 方法。

```java
// ObservableFlatMap.java
public void subscribeActual(Observer<? super U> t) {
    source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));
}
```

内部又通过这些参数和下游的 Observer 实例构建了一个 MergeObserver 实例，接着看看其 onSubscribe 方法。

```java
// MergeObserver.java
public void onSubscribe(Disposable d) {
    // 只会回调一次下游的onSubscribe方法
    if (DisposableHelper.validate(this.upstream, d)) {
        this.upstream = d;
        downstream.onSubscribe(this);
    }
}
```

如果已经有上游了就不做任何处理不然进行上游的赋值，然后回调了下游也就是自定义的那个 Observer 的onSubscribe 方法，接着看看其 onNext 方法是怎么把一个输入源转化成一个 Observable 的。

```java
public void onNext(T t) {
    ObservableSource<? extends U> p;
    p = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper returned a null ObservableSource");
    subscribeInner(p);
}
```

先是调用了传入的 apply 方法将每个 onNext 数据源转化为 Observable 实例，接着调用 subscribeInner 方法。

```java
void subscribeInner(ObservableSource<? extends U> p) {
    for (;;) {
        if (p instanceof Callable) { // ObservableFromArray 不派生自 Callable
            ...
        } else {
            InnerObserver<T, U> inner = new InnerObserver<T, U>(this, uniqueId++);
            if (addInner(inner)) {
                p.subscribe(inner);
            }
            break;
        }
    }
}
boolean addInner(InnerObserver<T, U> inner) {
    for (;;) {
        InnerObserver<?, ?>[] a = observers.get();
        int n = a.length;
        InnerObserver<?, ?>[] b = new InnerObserver[n + 1];
        System.arraycopy(a, 0, b, 0, n);
        b[n] = inner;
        if (observers.compareAndSet(a, b)) {
            return true;
        }
    }
}
```

为每个 Observable 对象都创建了一个 InnerObserver 实例，然后将其放入到一个数组中去，最后调用 subscribe 方法进行订阅，由于 apply 方法返回了一个 ObservableFromArray 实例，所以看看其 subscribeActual 方法。

```java
// ObservableFromArray.java
public void subscribeActual(Observer<? super T> observer) {
    FromArrayDisposable<T> d = new FromArrayDisposable<T>(observer, array);
    observer.onSubscribe(d);
    if (d.fusionMode) {
        return;
    }
    d.run();
}
```

observer 指代 InnerObserver，看看其 onSubscribe 方法。

```java
public void onSubscribe(Disposable d) {
    // 第一次调用会返回true，d就是FromArrayDisposable实例其派生自QueueDisposable
    if (DisposableHelper.setOnce(this, d)) {
        if (d instanceof QueueDisposable) {
            QueueDisposable<U> qd = (QueueDisposable<U>) d;
            // 相当于传入了7，返回SYNC
            int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);
            if (m == QueueDisposable.SYNC) {
                fusionMode = m;
                queue = qd;
                done = true;
                parent.drain();
                return;
            }
            if (m == QueueDisposable.ASYNC) {
                fusionMode = m;
                queue = qd;
            }
        }
    }
}
//FromArrayDisposable.java
public int requestFusion(int mode) {
    // 很明显7 & 1 != 0
    if ((mode & SYNC) != 0) {
        fusionMode = true;
        return SYNC;
    }
    return NONE;
}
```

接着会调用到 MergeObserver.drain 方法。

```java
void drain() {
    // 只会执行一次，循环将所有事件取出
    if (getAndIncrement() == 0) {
        drainLoop();
    }
}
// 当取消了或者出现了错误并其dealyErrors为false时会将所有InnerObserver都dispose掉
boolean checkTerminate() {
    if (cancelled) {
        return true;
    }
    Throwable e = errors.get();
    if (!delayErrors && (e != null)) {
        disposeAll();
        e = errors.terminate();
        if (e != ExceptionHelper.TERMINATED) {
            downstream.onError(e);
        }
        return true;
    }
    return false;
}
void drainLoop() {
    // 这里的downstream就是外界自定义的Observer实例
    final Observer<? super U> child = this.downstream;
    for (;;) {
        ...
        for (int i = 0; i < n; i++) {
            if (checkTerminate()) {
                return;
            }
            InnerObserver<T, U> is = (InnerObserver<T, U>)inner[j];
            SimpleQueue<U> q = is.queue;
            if (q != null) {
                for (;;) {
                    U o;
                    try {
                        o = q.poll();
                    } catch (Throwable ex) {
                        ...
                        if (checkTerminate()) {
                            return;
                        }
                        continue sourceLoop;
                    }
                    // 取不到数据了，退出
                    if (o == null) {
                        break;
                    }
                    // 每取出一个便会调用下游的onNext方法
                    child.onNext(o);
                }
            }
            ...
        }
        ..
    }
}
```

可以看到内部调用了 mapper.apply 方法，接着将拿到的结果 v 当做参数调用 downstream.onNext 方法，注意这里的 downStream 就是外界创建的一个 Observer 对象。

### zip

zip 方法通过一个函数将多个 Observables 的发射物结合到一起，基于这个函数的结果为每个结合体发射单个数据项。如下图所示：

![](/Users/hefuwei/GitHub/blog/source/_posts/RxJava2 源码初探/zip.png)

下面是一个使用zip操作符的一个简单例子：

```java
fun main() {
    Observable.zip(getObservable1(), getObservable2(), zipper())
        .subscribe(object : Observer<String> {
            override fun onComplete() {
                println("onComplete")
            }
            override fun onSubscribe(d: Disposable) {
                println("onSubscribe")
            }
            override fun onNext(t: String) {
                println("onNext $t")
            }
            override fun onError(e: Throwable) {
                println("onError $e")
            }
        })
}
fun getObservable1(): ObservableSource<String> {
    return Observable.create {
        it.onNext("A")
        it.onNext("B")
        it.onNext("C")
        Thread.sleep(1000)
    }
}
fun getObservable2(): ObservableSource<String> {
    return Observable.create {
        it.onNext("1")
        Thread.sleep(1000)
        it.onNext("2")
        Thread.sleep(1000)
        it.onNext("3")
    }
}
fun zipper(): BiFunction<String, String, String> {
    return BiFunction { s1, s2 -> s1 + s2 }
}
```

输出结果如下，onSubscribe 输出后过一秒输出 A1，再过一秒输出 A2，再过一秒输出 A3。

```
onSubscribe
onNext A1
onNext B2
onNext C3
```

那么为什么输出结果会是这个样子的呢？来看看zip的源码实现。

```java
public static <T1, T2, R> Observable<R> zip(
        ObservableSource<? extends T1> source1, ObservableSource<? extends T2> source2,
        BiFunction<? super T1, ? super T2, ? extends R> zipper) {
    return zipArray(Functions.toFunction(zipper), false, bufferSize(), source1, source2);
}
public static <T, R> Observable<R> zipArray(Function<? super Object[], ? extends R> zipper,
        boolean delayError, int bufferSize, ObservableSource<? extends T>... sources) {
    return RxJavaPlugins.onAssembly(new ObservableZip<T, R>(sources, null, zipper, bufferSize, delayError));
}
```

根据源码可以看出 `zip` 方法其实最终创建了一个 `ObservableZip` 实例，直接看其 `subscribeActual` 。

```java
// ObservableZip
public void subscribeActual(Observer<? super R> observer) {
    ObservableSource<? extends T>[] sources = this.sources;
    int count = sources.length;
    ZipCoordinator<T, R> zc = new ZipCoordinator<T, R>(observer, zipper, count, delayError);
    zc.subscribe(sources, bufferSize);
}
ZipCoordinator(Observer<? super R> actual,
        Function<? super Object[], ? extends R> zipper,
        int count, boolean delayError) {
    this.downstream = actual;
    this.zipper = zipper;
    this.observers = new ZipObserver[count];
    this.row = (T[])new Object[count];
    this.delayError = delayError;
}
public void subscribe(ObservableSource<? extends T>[] sources, int bufferSize) {
    ZipObserver<T, R>[] s = observers;
    int len = s.length;
    for (int i = 0; i < len; i++) {
        s[i] = new ZipObserver<T, R>(this, bufferSize);
    }
    this.lazySet(0);
    downstream.onSubscribe(this);
    for (int i = 0; i < len; i++) {
        if (cancelled) {
            return;
        }
        sources[i].subscribe(s[i]);
    }
}
```

可以看出 ZipCoordinator的 `subscribe` 内部创建了输入源大小的 ZipObserver 实例，然后调用每个输入源的 `subscribe` 方法，这样当输入源发送事件时就会调用 ZipObserver 的 `onNext` 方法。

```kotlin
// ZipObserver.java
public void onNext(T t) {
    queue.offer(t);
    parent.drain();
}
```

主要看看 ZipCoordinator 的 `drain` 方法。

```java
public void drain() {
    final ZipObserver<T, R>[] zs = observers;
    final Observer<? super R> a = downstream;
    final T[] os = row;
    final boolean delayError = this.delayError;
    for (;;) {
        for (;;) {
            int i = 0;
            int emptyCount = 0;
            for (ZipObserver<T, R> z : zs) {
                if (os[i] == null) {
                    boolean d = z.done;
                    T v = z.queue.poll();
                    boolean empty = v == null;
                    if (!empty) {
                        os[i] = v;
                    } else {
                        emptyCount++;
                    }
                } else {
                    // 主要是错误判断
                }
                i++;
            }
            if (emptyCount != 0) {
                break;
            }
            R v = zipper.apply(os.clone());
            a.onNext(v);
            Arrays.fill(os, null);
        }
    }
}
```

结合示例，首先 Obserable1 会发送一个 A 事件，将其放入到了一个队列中去，接着 `drain` 遍历所有的 ZipObserver，第一个 ZipObserver 可以从队列中事件将其赋值给 `os[0]` ，第二个取不到因此 `emptyCount++`，然后退出循环。接着 Observable1 又发送了一个 B 事件，再将其放入队列中，然后执行 `drain` ，这次因为 `os[0]` 已经不为 null 所以不会从队列中取，`os[1]` 还是 null，退出循环继续执行，接着 Observable1 再次发送一个 C 事件，这个跟 B 事件处理逻辑一样，再接着 Observable2 会发送一个 1 事件，将其放入队列，执行 `drain`将 `os[1]` 赋值成1，由于 `emptyCount` 等于0，因此会执行 `zipper.apply`，这个方法内部会回调传入的 BiFunction 的 `apply` 方法(示例中仅仅进行了字符串拼接)，获取到结果 A1，回调下游的 onNext 方法，然后将 row 这个数组置空，接着线程睡眠 1 秒，然后再次发送事件 2，将其放入队列中，执行 `drain`，方法内部遍历两个 ZipObserver 并且都能从队列中取到事件，所以 `emptyCount` 等于 0，接着就会执行 `apply` 然后获取到结果 B2，调用下游的 `onNext`，后面 Observable2 的 3 事件也跟 2 事件一样就不说了。

### 三、总结

通过分析 map、flatMap、zip 两个方法可以总结出以下几个的结论。

1. subscribeActual 方法总是会调用上游的 subscribe 方法。
2. onSubscribe 方法总是会调用下游的 onSubscribe 方法。
3. Observer 实例的 **onSubscribe** 会在事件发射前调用。
3. RxJava 提供的一些操作符其实会在内部创建自己的 Observable 和 Observer 实例，这样可以对下游 Observer 事件进行拦截。