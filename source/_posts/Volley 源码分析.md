---
title: Volley 源码分析
date: 2018-12-13 19:46:11
type: "Android"
---

## 前言

学习了 Android 这么久，一直没完整看过一个框架的源码，打算先看看以前用过的 Volley 源码，虽然大体上被 OkHttp 替代了，不过它也有优点，比如其非常适合进行数据量不大，但通信频繁的网络操作、占用空间比较小等。 <!-- more -->

## 基本用法

发起一个简单的网络请求代码如下：

```kotlin
private fun getPhoneInfo() {
    val requestQueue = Volley.newRequestQueue(applicationContext)
    val stringRequest = StringRequest(url, { message: String ->
        println(message)
    }, { error: VolleyError ->
        println(error.message)
    }).apply {
        tag = this@MainActivity
    }
    requestQueue.add(stringRequest)
}
```

本文的流程图如下：

![流程图](流程图.png)

由上述代码可知，使用 Volley 的主要步骤包括创建 RequestQueue、创建 Request、将 Request 加入到RequestQueue 三步。

## RequestQueue 的创建

常规做法是调用 Volley.newRequestQueue 这个 API 创建 RequestQueue 实例，代码如下：

```java
public static RequestQueue newRequestQueue(Context context) {
    return newRequestQueue(context, (BaseHttpStack) null);
}
public static RequestQueue newRequestQueue(Context context, BaseHttpStack stack) {
    BasicNetwork network;
    if (stack == null) {
        if (Build.VERSION.SDK_INT >= 9) {
            network = new BasicNetwork(new HurlStack());
        } else {
            // 小于 2.3 使用 HttpClient 忽略。
        }
    } else {
        network = new BasicNetwork(stack); // 1
    }
    return newRequestQueue(context, network);
}
private static RequestQueue newRequestQueue(Context context, Network network) {
    File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);
    RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network); // 2
    queue.start();
    return queue;
}
public RequestQueue(
            Cache cache, Network network, int threadPoolSize, ResponseDelivery delivery) {
    mCache = cache;
    mNetwork = network;
    mDispatchers = new NetworkDispatcher[threadPoolSize];
    mDelivery = delivery;
}
public void start() {
    stop();
    mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery); // 3
    mCacheDispatcher.start();
    for (int i = 0; i < mDispatchers.length; i++) { // 4
        NetworkDispatcher networkDispatcher =
            new NetworkDispatcher(mNetworkQueue, mNetwork, mCache, mDelivery);
        mDispatchers[i] = networkDispatcher;
        networkDispatcher.start();
    }
}
```

1. 创建了一个 BaseNetwork 实例，传入参数 null 。
2. 创建了一个 RequestQueue 实例，传入参数 DiskBasedCache、BaseNetwork 实例。
3. 创建了一个 CacheDispatcher（ 派生自 Thread ） 实例，传入参数两个 PriorityBlockingQueue（无界，基于二叉小顶堆，插入不会阻塞，取出可能阻塞），并启动。
4. 创建了四个 NetworkDispatcher（ 派生自 Thread ） 实例，将其保存到 mDispatcher 中，并分别启动。

### CacheDispatcher.run

代码如下：

```java
public CacheDispatcher(
            BlockingQueue<Request<?>> cacheQueue,
            BlockingQueue<Request<?>> networkQueue,
            Cache cache,
            ResponseDelivery delivery) { // 1
    mCacheQueue = cacheQueue;
    mNetworkQueue = networkQueue;
    mCache = cache;
    mDelivery = delivery;
    mWaitingRequestManager = new WaitingRequestManager(this);
}
public void run() { // 2
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    mCache.initialize();
    while (true) {
        try {
            processRequest();
        } catch (InterruptedException e) {
            if (mQuit) {
                Thread.currentThread().interrupt();
                return;
            }
        }
    }
}
```

1. ```cache``` 对应 DiskBasedCache 实例，```delivery``` 对应 ExecutorDelivery 实例。
2. 执行 ```cache.initialize```、 ```processRequest``` 。

为了简便起见，暂时不考虑缓存，先看 processRequest 。

#### DiskBasedCache.initialize

代码如下：

```java
public synchronized void initialize() {
    if (!mRootDirectory.exists()) { // 1
        if (!mRootDirectory.mkdirs()) {
            VolleyLog.e("Unable to create cache dir %s", mRootDirectory.getAbsolutePath());
        }
        return;
    }
    File[] files = mRootDirectory.listFiles();
    if (files == null) { // 2
        return;
    }
    for (File file : files) {
        try {
            long entrySize = file.length();
            CountingInputStream cis =
                new CountingInputStream(
                new BufferedInputStream(createInputStream(file)), entrySize); // 3
            try {
                CacheHeader entry = CacheHeader.readHeader(cis); // 4
                entry.size = entrySize;
                putEntry(entry.key, entry);
            } finally {
                cis.close();
            }
        } catch (IOException e) {
            file.delete();
        }
    }
}
static CacheHeader readHeader(CountingInputStream is) throws IOException {
    int magic = readInt(is); // 5
    if (magic != CACHE_MAGIC) {
        throw new IOException();
    }
    String key = readString(is);
    String etag = readString(is);
    long serverDate = readLong(is);
    long lastModified = readLong(is);
    long ttl = readLong(is);
    long softTtl = readLong(is);
    List<Header> allResponseHeaders = readHeaderList(is);
    return new CacheHeader(
        key, etag, serverDate, lastModified, ttl, softTtl, allResponseHeaders);
}
```

1. 如果目录 cachedir / volley 不存在，那么就创建该目录。
2. 如果目录下没有文件，那么直接返回。
3. CountingInputStream 只是多了剩余字节统计。
4. 读取缓存的文件信息，将读取到的缓存头信息保存到 mEntries 中。
5. 读取方式跟 Class 文件基本一致，首先是魔数，如果是字符串前面 4 个字节做为后续字符串的长度。

接下来回到 processRequest 。

#### CacheDispatcher.processRequest

代码如下：

```java
private void processRequest() throws InterruptedException {
    final Request<?> request = mCacheQueue.take();
    processRequest(request);
}
```

由于 mCacheQueue 为空，因此 CacheDispatcher 在  ```mCacheQueue.take()```  阻塞。回到 RequestQueue.start ，看看 NetworkDispatcher.run 。

### NetworkDispatcher.run

代码如下：

```java
public void run() {
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    while (true) {
        try {
            processRequest();
        } catch (InterruptedException e) {
            if (mQuit) {
                Thread.currentThread().interrupt();
                return;
            }
        }
    }
}
private void processRequest() throws InterruptedException {
    Request<?> request = mQueue.take();
    processRequest(request);
}
```

根据上述代码，4 个 NetworkDispatcher 线程都会在  ```mQueue.take()```  阻塞，接着进行第二步， Request 实例的创建。

## Request 的创建

Volley 提供了 StringRequest、JsonRequest、ImageRequest ，分别用于将 Response 转换为 String、Object、Bitmap，一般不需要自定义 Request ，看看 StringRequest 构造器，代码如下：

```java
public class StringRequest extends Request<String> {

    private Listener<String> mListener;

    public StringRequest(
            int method,
            String url,
            Listener<String> listener,
            @Nullable ErrorListener errorListener) {
        super(method, url, errorListener);
        mListener = listener;
    }

    public StringRequest(
            String url, Listener<String> listener, @Nullable ErrorListener errorListener) {
        this(Method.GET, url, listener, errorListener);
    }
    ...
}
public abstract class Request<T> implements Comparable<Request<T>> {
    public Request(int method, String url, @Nullable Response.ErrorListener listener) {
        mMethod = method;
        mUrl = url;
        mErrorListener = listener;
        setRetryPolicy(new DefaultRetryPolicy());
        mDefaultTrafficStatsTag = findDefaultTrafficStatsTag(url);
    }
}
```

主要进行了 method、url、listener、errorListener 的缓存，以及 RetryPolity、DefaultTrafficStatsTag 的设置，注意， **Request 类构造器不接收 Listener 实例**，这是因为 NetworkResponse 是子类自身负责解析的，各个子类解析后的结果都不同，因此 Listener 需要子类自己进行处理。接下来看看将 Request 加入到 RequestQueue 。

## RequestQueue 添加 Request

这一步是使用 Volley 三步的的最后一步，因此网络请求肯定也是从这里开始的，RequestQueue.add 代码如下：

```java
public <T> Request<T> add(Request<T> request) {
    request.setRequestQueue(this);
    synchronized (mCurrentRequests) {
        mCurrentRequests.add(request);
    }
    request.setSequence(getSequenceNumber());
    request.addMarker("add-to-queue");
    if (!request.shouldCache()) {
        mNetworkQueue.add(request);
        return request;
    }
    mCacheQueue.add(request);
    return request;
}
```

默认情况下 Request 是需要进行缓存的，不过为了简便起见，先考虑不需要缓存的情况，于是 request 会被放入 mNetworkQueue 中，上文创建 RequestQueue 中已经说到 4 个 NetworkDispatcher 线程全部由于 ```mNetworkQueue.take```  而阻塞，因此 4 个 NetworkDispatcher 线程中的某一个会脱离阻塞。

```java
private void processRequest() throws InterruptedException {
    Request<?> request = mQueue.take();
    processRequest(request);
}
void processRequest(Request<?> request) {
    long startTimeMs = SystemClock.elapsedRealtime();
    try {
        if (request.isCanceled()) { // 1
            request.finish("network-discard-cancelled");
            request.notifyListenerResponseNotUsable();
            return;
        }
        addTrafficStatsTag(request);
        NetworkResponse networkResponse = mNetwork.performRequest(request); // 2
        if (networkResponse.notModified && request.hasHadResponseDelivered()) {
            request.finish("not-modified");
            request.notifyListenerResponseNotUsable();
            return;
        }
        Response<?> response = request.parseNetworkResponse(networkResponse); // 3
        request.addMarker("network-parse-complete");
        if (request.shouldCache() && response.cacheEntry != null) {
            mCache.put(request.getCacheKey(), response.cacheEntry); // 4
        }
        request.markDelivered();
        mDelivery.postResponse(request, response);
        request.notifyListenerResponseReceived(response);
    } catch (VolleyError volleyError) {
        volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
        parseAndDeliverNetworkError(request, volleyError);
        request.notifyListenerResponseNotUsable();
    } catch (Exception e) {
        VolleyError volleyError = new VolleyError(e);
        volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
        mDelivery.postError(request, volleyError);
        request.notifyListenerResponseNotUsable();
    }
}
```

1. 如果在发起请求前 Request 已经被取消了，那么就不发起请求。
2. 执行  ```network.performRequest()``` ，这里的 network 为 BasicNetWork 实例。
3. 执行  ```request.parseNetworkResponse()``` ，这里的 request 为 StringRequest 实例。
4. 如果需要进行缓存，并且缓存内容不为空，那么放入 ```mCache``` 中，对应 DiskBasedCache 实例，暂时不考虑。
5. 执行  ```mDelivery.postResponse() ```  ，这里的 mDelivery 对应 ExecutorDelivery 实例。

### BasicNetWork.performRequest

代码如下：

```java
public BasicNetwork(HttpStack httpStack) {
    this(httpStack, new ByteArrayPool(DEFAULT_POOL_SIZE));
}
public BasicNetwork(HttpStack httpStack, ByteArrayPool pool) {
    mHttpStack = httpStack;
    mBaseHttpStack = new AdaptedHttpStack(httpStack);
    mPool = pool;
}
public BasicNetwork(BaseHttpStack httpStack, ByteArrayPool pool) {
    mBaseHttpStack = httpStack;
    mHttpStack = httpStack;
    mPool = pool;
}
public NetworkResponse performRequest(Request<?> request) throws VolleyError {
    long requestStart = SystemClock.elapsedRealtime();
    while (true) {
        HttpResponse httpResponse = null;
        byte[] responseContents = null;
        List<Header> responseHeaders = Collections.emptyList();
        try {
            Map<String, String> additionalRequestHeaders =
                getCacheHeaders(request.getCacheEntry());
            httpResponse = mBaseHttpStack.executeRequest(request, additionalRequestHeaders); // 1
            int statusCode = httpResponse.getStatusCode();
            responseHeaders = httpResponse.getHeaders();
            if (statusCode == HttpURLConnection.HTTP_NOT_MODIFIED) { // 2
                Entry entry = request.getCacheEntry();
                if (entry == null) {
                    return new NetworkResponse(
                        HttpURLConnection.HTTP_NOT_MODIFIED,
                        /* data= */ null,
                        /* notModified= */ true,
                        SystemClock.elapsedRealtime() - requestStart,
                        responseHeaders);
                }
                List<Header> combinedHeaders = combineHeaders(responseHeaders, entry);
                return new NetworkResponse(
                    HttpURLConnection.HTTP_NOT_MODIFIED,
                    entry.data,
                    /* notModified= */ true,
                    SystemClock.elapsedRealtime() - requestStart,
                    combinedHeaders);
            }
            InputStream inputStream = httpResponse.getContent();
            if (inputStream != null) { // 3
                responseContents =
                    inputStreamToBytes(inputStream, httpResponse.getContentLength());
            } else {
                responseContents = new byte[0];
            }
            long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
            if (statusCode < 200 || statusCode > 299) {
                throw new IOException();
            }
            return new NetworkResponse(
                statusCode,
                responseContents,
                /* notModified= */ false,
                SystemClock.elapsedRealtime() - requestStart,
                responseHeaders);
        } catch (SocketTimeoutException e) {
            attemptRetryOnException("socket", request, new TimeoutError()); // 4
        } catch (MalformedURLException e) {
            throw new RuntimeException("Bad URL " + request.getUrl(), e);
        } catch (IOException e) {
            int statusCode;
            if (httpResponse != null) {
                statusCode = httpResponse.getStatusCode();
            } else {
                throw new NoConnectionError(e);
            }
            NetworkResponse networkResponse;
            if (responseContents != null) {
                networkResponse =
                    new NetworkResponse(
                    statusCode,
                    responseContents,
                    /* notModified= */ false,
                    SystemClock.elapsedRealtime() - requestStart,
                    responseHeaders);
                if (statusCode == HttpURLConnection.HTTP_UNAUTHORIZED
                    || statusCode == HttpURLConnection.HTTP_FORBIDDEN) {
                    attemptRetryOnException(
                        "auth", request, new AuthFailureError(networkResponse));
                } else if (statusCode >= 400 && statusCode <= 499) {
                    throw new ClientError(networkResponse);
                } else if (statusCode >= 500 && statusCode <= 599) {
                    if (request.shouldRetryServerErrors()) {
                        attemptRetryOnException(
                            "server", request, new ServerError(networkResponse));
                    } else {
                        throw new ServerError(networkResponse);
                    }
                } else {
                    throw new ServerError(networkResponse);
                }
            } else {
                attemptRetryOnException("network", request, new NetworkError()); // 5
            }
        }
    }
}
```

1. ```mBaseHttpStack```  为 HurlStack 实例，内部会真正去请求网络。
2. 如果响应码为 304 ，直接使用缓存数据，对响应头做相应的修改，组装成 NetworkResponse 实例返回。
3. 如果有响应体，那么读取字节数组，保存到变量  ```responseContents```  ，内部使用了 ByteArrayPool 进行字节数组的缓存，**注意其会将响应流的所有内容都读入内存，如果数据很多那么可能导致 OOM** 。
4. 如果网络请求超时，根据 RetryPolicy 判断是否需要重试，如果需要重试则 while 循环再执行一遍，如果不需要重试再向上抛 SocketTimeoutException 。
5. 如果网络请求出现异常，根据 RetryPolicy 判断是否需要重试，如果需要重试则 while 循环再执行一遍，如果不需要重试再向上抛 NetworkError 。

#### HurlStack.executeRequest

```java
public HttpResponse executeRequest(Request<?> request, Map<String, String> additionalHeaders)
            throws IOException, AuthFailureError {
    String url = request.getUrl();
    HashMap<String, String> map = new HashMap<>();
    map.putAll(additionalHeaders);
    map.putAll(request.getHeaders()); // 1
    if (mUrlRewriter != null) { // 2
        String rewritten = mUrlRewriter.rewriteUrl(url);
        if (rewritten == null) {
            throw new IOException("URL blocked by rewriter: " + url);
        }
        url = rewritten;
    }
    URL parsedUrl = new URL(url);
    HttpURLConnection connection = openConnection(parsedUrl, request); // 3
    boolean keepConnectionOpen = false;
    try {
        for (String headerName : map.keySet()) {
            connection.setRequestProperty(headerName, map.get(headerName)); // 4
        }
        setConnectionParametersForRequest(connection, request); // 5
        int responseCode = connection.getResponseCode();
        if (responseCode == -1) {
            throw new IOException("Could not retrieve response code from HttpUrlConnection.");
        }
        if (!hasResponseBody(request.getMethod(), responseCode)) { // 6
            return new HttpResponse(responseCode, convertHeaders(connection.getHeaderFields()));
        }
        keepConnectionOpen = true;
        return new HttpResponse(
            responseCode,
            convertHeaders(connection.getHeaderFields()),
            connection.getContentLength(),
            new UrlConnectionInputStream(connection));
    } finally {
        if (!keepConnectionOpen) {
            connection.disconnect(); // 7
        }
    }
}
private HttpURLConnection openConnection(URL url, Request<?> request) throws IOException {
    HttpURLConnection connection = createConnection(url);
    int timeoutMs = request.getTimeoutMs();
    connection.setConnectTimeout(timeoutMs);
    connection.setReadTimeout(timeoutMs);
    connection.setUseCaches(false);
    connection.setDoInput(true);
    if ("https".equals(url.getProtocol()) && mSslSocketFactory != null) {
        ((HttpsURLConnection) connection).setSSLSocketFactory(mSslSocketFactory);
    }
    return connection;
}
protected HttpURLConnection createConnection(URL url) throws IOException {
    HttpURLConnection connection = (HttpURLConnection) url.openConnection();
    connection.setInstanceFollowRedirects(HttpURLConnection.getFollowRedirects());
    return connection;
}
static void setConnectionParametersForRequest(
            HttpURLConnection connection, Request<?> request) throws IOException, AuthFailureError {
    switch (request.getMethod()) {
        case Method.DEPRECATED_GET_OR_POST:
            byte[] postBody = request.getPostBody();
            if (postBody != null) {
                connection.setRequestMethod("POST");
                addBody(connection, request, postBody);
            }
            break;
        case Method.GET:
            connection.setRequestMethod("GET");
            break;
        case Method.DELETE:
            connection.setRequestMethod("DELETE");
            break;
        case Method.POST:
            connection.setRequestMethod("POST");
            addBodyIfExists(connection, request);
            break;
        case Method.PUT:
            connection.setRequestMethod("PUT");
            addBodyIfExists(connection, request);
            break;
        case Method.HEAD:
            connection.setRequestMethod("HEAD");
            break;
        case Method.OPTIONS:
            connection.setRequestMethod("OPTIONS");
            break;
        case Method.TRACE:
            connection.setRequestMethod("TRACE");
            break;
        case Method.PATCH:
            connection.setRequestMethod("PATCH");
            addBodyIfExists(connection, request);
            break;
        default:
            throw new IllegalStateException("Unknown method type.");
    }
}
private static void addBodyIfExists(HttpURLConnection connection, Request<?> request)
            throws IOException, AuthFailureError {
    byte[] body = request.getBody();
    if (body != null) {
        addBody(connection, request, body);
    }
}
private static void addBody(HttpURLConnection connection, Request<?> request, byte[] body)
            throws IOException {
 
    connection.setDoOutput(true);
    if (!connection.getRequestProperties().containsKey(HttpHeaderParser.HEADER_CONTENT_TYPE)) {
        connection.setRequestProperty(
            HttpHeaderParser.HEADER_CONTENT_TYPE, request.getBodyContentType());
    }
    DataOutputStream out = new DataOutputStream(connection.getOutputStream());
    out.write(body);
    out.close();
}
```

1. 将合并后的请求头放入 ```map``` 中，当然由于暂时考虑不需要缓存因此 ```additionalHeaders``` 必定为 null 。
2. 提供 UrlRewriter 的接口，在发起请求前，修改 Url ，可以用于批量添加请求参数等。
3. 为 HttpUrlConnection 设置是否重定向、超时时间，如果是 Https 请求还会设置 SSLSocketFactory。 
4. 为 HttpUrlConnection 设置合并后的请求头。
5. 为 HttpUrlConnection 设置请求方法，如果是 POST 请求还会添加 Content-Type 请求头以及请求体。
6. 判断 Response 是否有响应体，100 <= code < 200 ，code = 204，code = 304、HEAD 请求方式等就没响应头。
7. 如果没有响应体那么就直接断开连接，有响应体由于还要读取，先不断开。

### StringRequest.parseNetworkResponse

代码如下：

```java
protected Response<String> parseNetworkResponse(NetworkResponse response) {
    String parsed;
    try {
        parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers)); // 1
    } catch (UnsupportedEncodingException e) {
        parsed = new String(response.data);
    }
    return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response)); // 2
}
```

1. 将响应体使用对应的 Charset 转换为字符串。
2.  执行  ```httpHeaderParser.parseCacheHeaders```  解析响应头信息并返回 Response 实例，注意这里必须要解析响应头信息，不然该请求不会进行缓存。

#### HttpHeaderParser.parseCacheHeaders

代码如下：

```java
 public static Cache.Entry parseCacheHeaders(NetworkResponse response) {
     long now = System.currentTimeMillis();
     Map<String, String> headers = response.headers;
     long serverDate = 0;
     long lastModified = 0;
     long serverExpires = 0;
     long softExpire = 0;
     long finalExpire = 0;
     long maxAge = 0;
     long staleWhileRevalidate = 0;
     boolean hasCacheControl = false;
     boolean mustRevalidate = false;
     String serverEtag = null;
     String headerValue;
     headerValue = headers.get("Date");
     if (headerValue != null) {
         serverDate = parseDateAsEpoch(headerValue); // 1
     }
     headerValue = headers.get("Cache-Control");
     if (headerValue != null) {
         hasCacheControl = true;
         String[] tokens = headerValue.split(",", 0);
         for (int i = 0; i < tokens.length; i++) {
             String token = tokens[i].trim();
             if (token.equals("no-cache") || token.equals("no-store")) { // 2
                 return null;
             } else if (token.startsWith("max-age=")) {
                 try {
                     maxAge = Long.parseLong(token.substring(8)); // 3
                 } catch (Exception e) {
                 }
             } else if (token.startsWith("stale-while-revalidate=")) { // 4
                 try {
                     staleWhileRevalidate = Long.parseLong(token.substring(23));
                 } catch (Exception e) {
                 }
             } else if (token.equals("must-revalidate") || token.equals("proxy-revalidate")) { // 5
                 mustRevalidate = true;
             }
         }
     }

     headerValue = headers.get("Expires");
     if (headerValue != null) {
         serverExpires = parseDateAsEpoch(headerValue); // 6
     }
     headerValue = headers.get("Last-Modified");
     if (headerValue != null) {
         lastModified = parseDateAsEpoch(headerValue);
     }
     serverEtag = headers.get("ETag");
     if (hasCacheControl) { // 7
         softExpire = now + maxAge * 1000;
         finalExpire = mustRevalidate ? softExpire : softExpire + staleWhileRevalidate * 1000;
     } else if (serverDate > 0 && serverExpires >= serverDate) {
         softExpire = now + (serverExpires - serverDate);
         finalExpire = softExpire;
     }

     Cache.Entry entry = new Cache.Entry();
     entry.data = response.data;
     entry.etag = serverEtag;
     entry.softTtl = softExpire;
     entry.ttl = finalExpire;
     entry.serverDate = serverDate;
     entry.lastModified = lastModified;
     entry.responseHeaders = headers;
     entry.allResponseHeaders = response.allHeaders;
     return entry;
 }
```

1. 将服务器返回的处理请求的时间转化为 long 并保存到 ```serverDate``` 变量。
2. 缓存控制响应头要求禁止缓存，那么直接返回 null 。
3. 解析缓存控制响应头 max-age 信息并保存到 ```maxAge``` 变量。
4. 解析缓存控制响应头 stale-while-revalidate 信息并保存到  ```staleWhileRevalidate ``` 变量，该字段意思是在 staleWhileRevalidate 时间内可以直接把缓存内容返回。
5. 解析缓存控制响应头 must-revalidate 信息并保存到  ```mustRevalidate``` 变量，该字段意思是本地副本过期前，可以使用本地副本，本地副本一旦过期，必须向服务器进行有效性校验。
6. 解析响应头 Expires 信息并保存到  ```serverExpires``` 变量 。
7. 计算缓存有效时间，以缓存控制响应头优先。

### ExecutorDelivery.postResponse

代码如下：

```java
public ExecutorDelivery(final Handler handler) { // 1
    mResponsePoster = new Executor() {
        @Override
        public void execute(Runnable command) {
            handler.post(command);
        }
    };
}
public void postResponse(Request<?> request, Response<?> response) {
    postResponse(request, response, null);
}
@Override
public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
    mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable)); // 2
}
```

1. ```handler```  中的 Looper 是主线程的 Looper。
2. 切换到主线程执行 ResponseDeliveryRunnable.run 。

#### ResponseDeliveryRunnable.run

代码如下：

```java
public void run() {
    if (mRequest.isCanceled()) { // 1
        mRequest.finish("canceled-at-delivery");
        return;
    }
    if (mResponse.isSuccess()) { // 2
        mRequest.deliverResponse(mResponse.result);
    } else {
        mRequest.deliverError(mResponse.error);
    }
    if (mResponse.intermediate) {
        mRequest.addMarker("intermediate-response");
    } else {
        mRequest.finish("done");
    }
    if (mRunnable != null) { 
        mRunnable.run(); // 3
    }
}
```

1. 如果请求已经被取消了，那么停止分发响应，注意如果子线程在判断后再执行取消，那么还是会被分发，因此尽量在主线程取消，或者在回调中判断没取消再进行相应的操作。
2. 如果响应成功，那么调用 ```mRequest.deliverResponse``` ，否则调用 ```mRequest.deliverError``` 。
3. ``` mRunnable ``` 为空，因此不会执行。

##### StringRequest.deliverResponse

代码如下：

```java
protected void deliverResponse(String response) {
    Response.Listener<String> listener;
    synchronized (mLock) {
        listener = mListener;
    }
    if (listener != null) {
        listener.onResponse(response);
    }
}
```

执行  ```listener.onResponse``` 这里锁是一定要的，volatile 并不适用（可能导致 NPE ），同时只同步了一行语句，提升了效率。

##### StringRequest.deliverError

代码如下：

```java
public void deliverError(VolleyError error) {
    Response.ErrorListener listener;
    synchronized (mLock) {
        listener = mErrorListener;
    }
    if (listener != null) {
        listener.onErrorResponse(error);
    }
}
```

执行  ```listener.onErrorResponse``` 同样的这里锁是一定要的。

到此为止，看完了 Volley 的网络请求逻辑，接着看看 Volley 的缓存处理机制。

## Cache 

假设 Request 需要进行缓存，回到 NetworkDispatcher.processRequest 。

### NetworkDispatcher.processRequest 

代码如下：

```java
void processRequest(Request<?> request) {
    ...
    if (request.shouldCache() && response.cacheEntry != null) {
        mCache.put(request.getCacheKey(), response.cacheEntry);
    }
    ...
}
```

这里的 mCache 对应 DiskBasedCache 实例。

#### DiskBasedCache.put

```java
public synchronized void put(String key, Entry entry) {
    pruneIfNeeded(entry.data.length); // 1
    File file = getFileForKey(key);
    try {
        BufferedOutputStream fos = new BufferedOutputStream(createOutputStream(file));
        CacheHeader e = new CacheHeader(key, entry);
        boolean success = e.writeHeader(fos); // 2
        if (!success) {
            fos.close();
            throw new IOException();
        }
        fos.write(entry.data); // 3
        fos.close();
        putEntry(key, e);
        return;
    } catch (IOException e) {}
    boolean deleted = file.delete();
    if (!deleted) {
        VolleyLog.d("Could not clean up file %s", file.getAbsolutePath());
    }
}
```

1. 由于 DiskBasedCache 内部是基于 LRU 缓存的，默认最大缓存大小为 5MB ，因此如果判断添加本次响应空间不够，就会删除最近最少使用的缓存，直到加上本次缓存，空间占用小于 90% 。
2. 将响应头写入缓存文件中。
3. 将响应体写入缓存文件中。

至此，缓存文件添加完成，接着来看看缓存文件的读取，在 RequestQueue 的创建部分中说到 CacheDispatcher 会先执行 DiskBasedCache.initialize 。

### CacheDispatcher.initalize

代码如下：

```java
public synchronized void initialize() {
    if (!mRootDirectory.exists()) { // 1
        if (!mRootDirectory.mkdirs()) {
            VolleyLog.e("Unable to create cache dir %s", mRootDirectory.getAbsolutePath());
        }
        return;
    }
    File[] files = mRootDirectory.listFiles();
    if (files == null) { // 2
        return;
    }
    for (File file : files) {
        try {
            long entrySize = file.length();
            CountingInputStream cis =
                new CountingInputStream(
                new BufferedInputStream(createInputStream(file)), entrySize); // 3
            try {
                CacheHeader entry = CacheHeader.readHeader(cis); // 4
                entry.size = entrySize;
                putEntry(entry.key, entry);
            } finally {
                cis.close();
            }
        } catch (IOException e) {
            file.delete();
        }
    }
}
static CacheHeader readHeader(CountingInputStream is) throws IOException {
    int magic = readInt(is); // 5
    if (magic != CACHE_MAGIC) {
        throw new IOException();
    }
    String key = readString(is);
    String etag = readString(is);
    long serverDate = readLong(is);
    long lastModified = readLong(is);
    long ttl = readLong(is);
    long softTtl = readLong(is);
    List<Header> allResponseHeaders = readHeaderList(is);
    return new CacheHeader(
        key, etag, serverDate, lastModified, ttl, softTtl, allResponseHeaders);
}
```

1. 如果缓存目录不存在，就创建该目录。
2. 如果目录下没有文件，那么直接返回。
3. CountingInputStream 只是多了剩余字节统计。
4. 读取缓存的文件信息，将读取到的缓存头信息保存到内存中。
5. 读取方式跟 Class 文件基本一致，首先是魔数，如果是字符串前面 4 个字节做为后续字符串的长度。

至此，缓存文件的写入、读取都已经完成，接着来看看缓存文件的使用，回到 RequestQueue 添加 Request 。

### RequestQueue.add

代码如下：

```java
public <T> Request<T> add(Request<T> request) {
    ...
    if (!request.shouldCache()) {
        mNetworkQueue.add(request);
        return request;
    }
    mCacheQueue.add(request);
    return request;
}
```

上文已经说明了不需要缓存的情况，现在考虑需要缓存，```mCacheQueue.add``` 会被执行，于是 CacheDispatcher 线程就会脱离阻塞，执行其 ```processRequest``` 。

#### CacheDispatcher.processRequest

代码如下：

```java
void processRequest(final Request<?> request) throws InterruptedException {
    if (request.isCanceled()) { // 1
        request.finish("cache-discard-canceled");
        return;
    }
    Cache.Entry entry = mCache.get(request.getCacheKey()); // 2
    if (entry == null) {
        request.addMarker("cache-miss");
        if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) { // 3
            mNetworkQueue.put(request);
        }
        return;
    }
    if (entry.isExpired()) {
        request.setCacheEntry(entry);
        if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) { // 4
            mNetworkQueue.put(request);
        }
        return;
    }
    Response<?> response = request.parseNetworkResponse( // 5
        new NetworkResponse(entry.data, entry.responseHeaders));
    if (!entry.refreshNeeded()) { // 6
        mDelivery.postResponse(request, response);
    } else {
        request.setCacheEntry(entry);
        response.intermediate = true;
        if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) { // 7
            mDelivery.postResponse(
                request,
                response,
                new Runnable() {
                    @Override
                    public void run() {
                        try {
                            mNetworkQueue.put(request);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                    }
                });
        } else {
            mDelivery.postResponse(request, response);
        }
    }
}
```

1. 如果请求已经被取消了，那么直接结束本次请求。
2. 调用 ```mCache.get``` 从缓存中读取 Entry 。
3. 如果没有缓存，并且没有相同的请求正在等待响应，那么将其放入网络阻塞队列，进行网络请求。
4. 如果缓存过期，并且没有相同的请求正在等待响应，那么将其放入网络阻塞队列，进行网络请求。
5. 缓存命中，将其解析成 Response 实例。
6. 如果缓存不需要刷新，那么直接分发缓存的响应。
7. 如果缓存需要刷新，并且没有相同的请求正在等待响应，那么先分发一次缓存的响应，同时再放入网络阻塞队列进行请求，如果有相同请求正在等待响应，那么直接分发缓存的响应。

##### DiskBasedCache.get

代码如下：

```java
public synchronized Entry get(String key) {
    CacheHeader entry = mEntries.get(key);
    if (entry == null) {
        return null;
    }
    File file = getFileForKey(key);
    try {
        CountingInputStream cis =
            new CountingInputStream(
            new BufferedInputStream(createInputStream(file)), file.length());
        try {
            CacheHeader entryOnDisk = CacheHeader.readHeader(cis);
            if (!TextUtils.equals(key, entryOnDisk.key)) {
                removeEntry(key);
                return null;
            }
            byte[] data = streamToBytes(cis, cis.bytesRemaining());
            return entry.toCacheEntry(data);
        } finally {
            cis.close();
        }
    } catch (IOException e) {
        remove(key);
        return null;
    }
}
```

由于原先只是把响应头给缓存到了内存中，因此需要从文件中读取响应体。

还有最后一个问题，如果有缓存，无论是否过期，都会给 Request 设置 CacheEntry ，那么这个有什么用？这需要再回到 BasicNetWork.performRequest 。

### BasicNetWork.performRequest

代码如下：

```java
public NetworkResponse performRequest(Request<?> request) throws VolleyError {
	...
    Map<String, String> additionalRequestHeaders =
        getCacheHeaders(request.getCacheEntry());
    httpResponse = mBaseHttpStack.executeRequest(request, additionalRequestHeaders); // 1
    ...
}
```

如果有缓存，获取需要带上的请求头。

#### BasicNetwork.getCacheHeaders

代码如下：

```java
private Map<String, String> getCacheHeaders(Cache.Entry entry) {
    if (entry == null) {
        return Collections.emptyMap();
    }
    Map<String, String> headers = new HashMap<>();
    if (entry.etag != null) { // 1
        headers.put("If-None-Match", entry.etag);
    }
    if (entry.lastModified > 0) {
        headers.put( // 2
            "If-Modified-Since", HttpHeaderParser.formatEpochAsRfc1123(entry.lastModified));
    }
    return headers;
}
```

1. 如果缓存中有 etag ，那么带上 If-None-Match 请求头，这样如果服务端判断 etag 没有发生变化就会返回 304，优化性能。
2. 如果缓存中有 lastModified ，那么带上 If-Modified-Since 请求头，这样如果服务端判断资源的最后修改时间与之一致，那么就返回 304 ， 优化性能。

缓存相关也基本阅读完成，来看下最后一个 RetryPolicy 。

## RetryPolicy

上文已经说到了当网络请求超时，或者遇见异常，都会使用 RetryPolicy 来判断是否需要进行重试操作。

代码如下：

```java
public interface RetryPolicy {
    int getCurrentTimeout(); // 1
    int getCurrentRetryCount(); // 2
    void retry(VolleyError error) throws VolleyError; // 3
}
```

1. 每次请求时都会调用该方法，获取 HttpURLConnection 的连接以及读取超时时间。
2. 该方法只用于响应较慢超过 3 秒打印 log 时，可以不需要实。
3. 如果想要进行重试，那么该方法可以什么都不做，如果不想要进行重试，抛出异常即可。

## 总结

Volley 源码基本上算是阅读完毕了，总结下其执行流程。

1. 创建 RequestQueue ，开启一个 CacheDispatcher 线程（首先读取文件缓存），以及四个 NetworkDispatcher 线程，分别由于 NetworkPriorityBlockingQueue、CachePriorityBlockingQueue 为空而阻塞。
2. 创建 Request 对象，并将其添加到 RequestQueue 中，判断是否需要缓存，如果需要那么将其添加到 CachePriorityBlockingQueue，如果不需要那么将其添加到 NetworkPriorityBlockingQueue 中。
3. 如需要缓存，CacheDispatcher 脱离阻塞，读取缓存文件，判断缓存文件是否有效，如有效那么就直接回调请求成功，如果无效或者无缓存，那么将其加入到 NetworkPriorityBlockingQueue 中。
4. 如不需要缓存，缓存无效或无缓存，NetworkDispatcher 脱离阻塞，进行网络请求，缓存如果有 etag、lastmodify 会额外添加两个请求头，获取到响应后，如果请求超时或者出现异常情况，根据重试策略判断是否需要进行重试，如果请求成功，交给 Request 解析 Response ，完成后如果需要进行缓存，则将解析后 Response 进行缓存，接着再使用 Handler 切换线程到主线程回调 onResponse 。

由于 Volley 内部使用了 ByteArrayPool 避免每次都去新建字节数组对象，所以适用于处理高频率的请求，同时由于其将所有响应通过字节数组输出流读入了内存所以其不适合进行大文件的下载，否则容易造成OOM。

