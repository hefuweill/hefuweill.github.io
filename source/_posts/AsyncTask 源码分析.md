---
title: AsyncTask 源码分析
date: 2018-11-11 12:22:10
type: "Android"
---

## 前言

AsyncTask 能够很容易的实现在子线程执行耗时操作，然后在主线程中更新进度，任务完成后能在主线程中收到结果，其提供了以下几个主要方法，先从一个例子开始。<!--more-->

* onPreExecute 在子线程执行任务前被调用，主线程调用。

* doInBackground 在线程池中执行，子线程调用。

* onPostExecute 在子线程执行完毕后调用，如果Task被取消了将不会被调用。

* publishProgress 用于在 doInBackground 中调用触发 onProgressUpdate，一般子线程调用。

* onProgressUpdate 由 publishProgress 触发。

## 例子

假设应用程序需要进行更新，下载新版本APK，在下载过程中需要实时更新进度，这时就可以使用 AsyncTask，代码如下，注意:需要在清单文件中加上网络和写文件权限。

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        DownloadTask(pb).execute("http://down.360safe.com/360mse/360mse_nh00002.apk")
    }
}
class DownloadTask(pb: ProgressBar): AsyncTask<String, Int, Unit>() {
    private val mPb = WeakReference(pb)
    private val mTag = "download"
    override fun doInBackground(vararg params: String?) {
        val url = URL(params[0])
        val path = "${Environment.getExternalStorageDirectory()}/360safe.apk"
        val conn = url.openConnection() as HttpURLConnection
        if (conn.responseCode.toString().startsWith("20")) {
            var inputStream: InputStream? = null
            var outputStream: OutputStream? = null
            try {
                inputStream = conn.inputStream
                outputStream = BufferedOutputStream(FileOutputStream(File(path)))
                val totalSize = conn.contentLength
                val size = 1024
                val buffer = ByteArray(size)
                var downloadedSize = 0
                var length = inputStream.read(buffer)
                while (length != -1) {
                    downloadedSize += size
                    outputStream.write(buffer, 0, length)
                    onProgressUpdate(totalSize, downloadedSize)
                    length = inputStream.read(buffer)
                }
            } catch (e: Exception) {
                e.printStackTrace()
            } finally {
                inputStream?.close()
                outputStream?.close()
            }
        }
    }
    override fun onProgressUpdate(vararg values: Int?) {
        mPb.get()?.max = values[0]!!
        mPb.get()?.progress = values[1]!!
    }
    override fun onPostExecute(result: Unit?) {
        Log.d(mTag, "下载完成")
    }
}
```

这样就能在子线程下载 APK，然后实时在主线程更新进度，下面看看 AsyncTask 的源码。

## 源码

首先来看看 AsyncTask 的构造方法。

```java
public AsyncTask() {
    this((Looper) null);
}
@hide
public AsyncTask(@Nullable Handler handler) {
    this(handler != null ? handler.getLooper() : null);
}
@hide
public AsyncTask(@Nullable Looper callbackLooper) {
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
                postResult(result);
            }
            return result;
        }
    };
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```

可以看出虽然有三个构造器但是我们只能使用无参构造方法，其它两个构造方法都是隐藏的，其实参数无论是 Handler 也好 Looper 也罢，最终只是想要一个 Looper 做为内部 Handler 的入参，这里由于 `callbackLooper` 为 null，所以会调用 `getMainHandler` 并将结果赋值给 `mHandler` ，然后新建了 `mWorker`和 `mFuture`，先来看看 `getMainLooper` 。

```java
private static Handler getMainHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler(Looper.getMainLooper());
        }
        return sHandler;
    }
}
```

继续看看 InternalHandler。

```java
private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```

这里可以看出当进度改变或者任务结束的时候都会发送消息过来在这里回调，然后看看 AsyncTask 的 `execute` 方法。

```java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }
    mStatus = Status.RUNNING;
    onPreExecute();
    mWorker.mParams = params;
    exec.execute(mFuture);
    return this;
}
```

主要做了以下几件事情：

1. 判断了当前的状态如果正在运行或者已经运行结束了就直接抛出异常。
2. 将当前状态改为运行中，调用 `onPreExecute` 该方法是一个空方法交由子类实现可以在执行任务之前通过该方法做一些操作。
3. 将参数赋值给 `mWorker.mParams` 。
4. 将 `mFuture` 丢给 SerialExecutor 进行执行。
5. 返回当前 AsyncTask 实例 。

接下来看看 `SerialExecutor.execute` 做了什么。

```java
private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;
    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }
    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

首先往 `mTasks` 里面添加了一个 Runnable 实例，然后判断当前是否有任务在执行，如果没有就调用 `scheduleNext` 执行任务，该方法会从队列中取出第一个 Runnable 实例丢给 `THREAD_POOL_EXECUTOR` 进行执行，而当一个 Runnable 执行完毕后又会调用 `scheduleNext` 执行下一个 Runnable，所以说同一时间只会有一个Runnable 执行，下面来看看 `THREAD_POOL_EXECUTOR`。

```java
public static final Executor THREAD_POOL_EXECUTOR;
static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;
private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);
    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};
private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);
```

可以看到 `THREAD_POOL_EXECUTOR ` 是一个核心线程为2-4，最大线程数为处理器数*2+1，线程超时时间30秒，允许核心线程超时的线程池，那么有个疑问刚才不是说是串行执行 Runnable 的吗？既然是一个个执行的搞个SingleThreadExecutor 不就行了？答案是如果我们调用 `execute` 方法确实是串行执行的，但是我们也可以直接调用 `executeOnExecutor` 方法将 `THREAD_POOL_EXECUTOR` 当做入参这样我们的 Task 就可以并行执行的了，下面代码调用到了 `mFuture.run`，先来看看 FutureTask 的构造方法。

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;
}
```

这里的 callable 其实就是在 AsyncTask 的构造方法中创建的 `mWorker`，这里将其赋值给了 `callable` 变量，然后将状态变更为 NEW，然后来看看其 `run` 方法。

```java
public void run() {
    if (state != NEW ||
        !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

首先判断 state 如果不是 New ，或者 runner 不是 null 那么就直接退出。接着调用 `mWorker.call` ，如果执行成功就调用 `set` 设置结果，执行失败就调用 `setException` 设置异常，先来看看 `mWorker.call` 。

```java
public Result call() throws Exception {
    mTaskInvoked.set(true);
    Result result = null;
    try {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        result = doInBackground(mParams);
        Binder.flushPendingCommands();
    } catch (Throwable tr) {
        mCancelled.set(true);
        throw tr;
    } finally {
        postResult(result);
    }
    return result;
}
```

该方法先将 `mTaskInvoked` 设置为 true，表明该 Task 已经得到执行，然后设置优先级为后台线程，再然后调用 `doInBackground` 获取到执行结果，如果执行失败会将 `mCancelled` 设置为 true，然后再抛出异常，最后调用 `postResult`。

```java
private Result postResult(Result result) {
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

该方法只是发了一个消息，根据前面讲到的 InternalHandler 的处理逻辑会调用 `finish` 。

```java
 private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

这里判断如果任务已经取消了就调用 `onCancelled` ，否则就调用 `onPostExecute`，最后将状态置为 FINISHED，再回到 FutureTask 的 `run` 方法，这样正常情况已经处理完毕了 ```onPreExecute、doInBackground、onPostExecute ```。不过还有 ```call```  抛出异常情况没处理，来看看 `setException`。

```java
protected void setException(Throwable t) {
    if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
        outcome = t;
        U.putOrderedInt(this, STATE, EXCEPTIONAL); // final state
        finishCompletion();
    }
}
```

判断状态是否是 NEW 是的话就把当前的状态改成 COMPLETING 然后调用 `finishCompletion`。

```java
private void finishCompletion() {
    ...
    done();
    callable = null;
}
```

 `done` 方法其实一个空实现，`mFuture` 实现了它，来看看。

```java
mFuture = new FutureTask<Result>(mWorker) {
    @Override
    protected void done() {
        try {
            postResultIfNotInvoked(get());
        } catch (InterruptedException e) {
            android.util.Log.w(LOG_TAG, e);
        } catch (ExecutionException e) {
            throw new RuntimeException("An error occurred while executing doInBackground()",
                    e.getCause());
        } catch (CancellationException e) {
            postResultIfNotInvoked(null);
        }
    }
};
private void postResultIfNotInvoked(Result result) {
    final boolean wasTaskInvoked = mTaskInvoked.get();
    if (!wasTaskInvoked) {
        postResult(result);
    }
}
```

内部首先调用 FutureTask.get ，该方法会抛出异常（由于刚才 call 抛出了异常），因此会往外再抛出异常。到此为止异常流程也已经走完了，但是还有一个方法在这个流程中没讲到那就是 `publishProgress`。

```java
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
public void handleMessage(Message msg) {
    AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
    switch (msg.what) {
        case MESSAGE_POST_RESULT:
            result.mTask.finish(result.mData[0]);
            break;
        case MESSAGE_POST_PROGRESS:
            result.mTask.onProgressUpdate(result.mData);
            break;
    }
}
```
根据 InternalHandler 可以看出其实是回调了 `onPregressUpdate`，至此全部流程都走完了，来总结一下。

## 总结

* AsyncTask 内部切换线程的原理也是线程池加 Handler。
* 默认情况下 AsyncTask 是串行执行的，也就是说假设你同时开启三个 AsyncTask，它们会一个个执行，而不会同时执行要想做到并行执行需要手动调用 `executeOnExecutor` 。

