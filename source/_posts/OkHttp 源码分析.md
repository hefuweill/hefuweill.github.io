---
title: OkHttp 源码分析
date: 2019-05-08 13:23:10
type: "Android"
---



## 前言

半年前阅读了 Volley 源码，但是现在主流网络请求都是使用 OkHttp + Retrofit + RxJava ，因此打算好好研究下 OkHttp 的源码(基于 OkHttp3.14.1)，记录一下。注：[仓库地址](https://github.com/square/okhttp)<!--more-->

## 简单使用

这里只例举基本的同步方式的 Get 请求，详细的请看[官方文档](https://square.github.io/okhttp/)。

```kotlin
val client = OkHttpClient()
fun syncGet() {
    val request = Request.Builder().url(URL).build()
    val call = client.newCall(request)
    val response = call.execute()
    if (response.isSuccessful) {
        println("Response is successful")
    }
}
```

由上可知，发送一个基本的 get 请求需要如下几步：

1. 创建 OkHttpClient 实例。
2. 创建 Request 实例。
3. 创建 Call 实例。
4. 执行 Call.execute 方法。

下面按照这四步探索下源码实现。

## OkHttpClient 实例的创建

首先看看 OkHttpClient 的构造器。

```java
public OkHttpClient() {
    this(new Builder());
}

OkHttpClient(Builder builder) {
    // 主要是从 builder 中取出对应字段进行赋值，忽略
    ...
}
```

其拥有两个构造器不过我们只能直接调用无参的那个，另一个主要给 Builder 的 build 方法使用的(典型的建造者模式)，接着看看 Builder 的构造器。

```java
public Builder() {
    // 创建异步执行策略
    dispatcher = new Dispatcher();
    // 默认协议列表Http1.1、Http2.0
    protocols = DEFAULT_PROTOCOLS;
    // 连接规格，包括TLS(用于https)、CLEARTEXT(未加密用于http)
    connectionSpecs = DEFAULT_CONNECTION_SPECS;
    // 事件监听，默认没有
    eventListenerFactory = EventListener.factory(EventListener.NONE);
    // 代理选择器
    proxySelector = ProxySelector.getDefault();
    // 使用空对象设计模式
    if (proxySelector == null) {
        proxySelector = new NullProxySelector();
    }
    // 提供Cookie策略的持久性
    cookieJar = CookieJar.NO_COOKIES;
    // socket工厂
    socketFactory = SocketFactory.getDefault();
    // hostname验证器
    hostnameVerifier = OkHostnameVerifier.INSTANCE;
    // 证书标签
    certificatePinner = CertificatePinner.DEFAULT;
    // 代理认证
    proxyAuthenticator = Authenticator.NONE;
    // 认证
    authenticator = Authenticator.NONE;
    // 连接池
    connectionPool = new ConnectionPool();
    // dns
    dns = Dns.SYSTEM;
    // 跟随ssl重定向
    followSslRedirects = true;
    // 跟随重定向
    followRedirects = true;
    // 当连接失败时尝试
    retryOnConnectionFailure = true;
    callTimeout = 0;
    // 连接、读取、写超时10秒
    connectTimeout = 10_000;
    readTimeout = 10_000;
    writeTimeout = 10_000;
    pingInterval = 0;
}

Builder(OkHttpClient okHttpClient) {
    // 拷贝构造器
    ...
}
```

Builder 中的属性到用到时再好好的研究，接着来看看第二步 Request 实例的创建。

## Request 实例的创建

Request 实例通过 Builder 进行创建的，因此首先看看 Request.Builder 的构造器。

```java
public Builder() {
    this.method = "GET";
    this.headers = new Headers.Builder();
}
Builder(Request request) {
    this.url = request.url;
    this.method = request.method;
    this.body = request.body;
    this.tags = request.tags.isEmpty()
            ? Collections.emptyMap()
            : new LinkedHashMap<>(request.tags);
    this.headers = request.headers.newBuilder();
}
```

其提供了两个构造器，不过我们只能调用无参的那个，其内部设置了默认请求方法为 GET，并且创建了一个HeaderBuilder 实例用于统一管理请求头，第二个构造器用于 Request 实例中拷贝出对应参数赋值给当前 Builder实例，接着看看其 url 方法和 build 方法。

```java
public Builder url(String url) {
    if (url == null) throw new NullPointerException("url == null");
    // 默默的将web socket urls替换成http urls
    if (url.regionMatches(true, 0, "ws:", 0, 3)) {
        url = "http:" + url.substring(3);
    } else if (url.regionMatches(true, 0, "wss:", 0, 4)) {
        url = "https:" + url.substring(4);
    }
    return url(HttpUrl.get(url));
}
public Builder url(HttpUrl url) {
    if (url == null) throw new NullPointerException("url == null");
    this.url = url;
    return this;
}
public Request build() {
    if (url == null) throw new IllegalStateException("url == null");
    return new Request(this);
}
Request(Builder builder) {
    this.url = builder.url;
    this.method = builder.method;
    this.headers = builder.headers.build();
    this.body = builder.body;
    this.tags = Util.immutableMap(builder.tags);
}
```

其中 url 方法主要是将请求地址封装成 HttpUrl 实例并赋值给成员 url，build 方法创建了 Request 实例。至此第二步结束了接着看看第三步 Call 实例的创建。

## Call 实例的创建

通过调用 OkHttpClient 实例的 newCall() 创建 Call 实例。

```java
public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
}
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.transmitter = new Transmitter(client, call);
    return call;
}
public Transmitter(OkHttpClient client, Call call) {
    this.client = client;
    this.connectionPool = Internal.instance.realConnectionPool(client.connectionPool());
    this.call = call;
    this.eventListener = client.eventListenerFactory().create(call);
    this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
}
```

newCall 方法返回了一个 RealCall 实例，并且初始化了其 transmitter 属性，而在 Transmitter 构造器中又初始化了 connectionPool、eventListener 属性，执行 timeout 方法，设置整次调用超时时间（包括连接、读写）默认为 0 ，其中 Internal.instance 在 OkHttpClient 这个类一加载时就初始化了。

```java
Internal.instance = new Internal() {
    ...
    @Override
    public RealConnectionPool realConnectionPool(ConnectionPool connectionPool) {
        return connectionPool.delegate;
    }
    ...
}
```

Transmitter.connectionPool 就是 RealConnectionPool 实例，而 Transmitter.eventListener 默认为 EventListener.NONE 。

## Call.execute

我们知道 call 其实是一个 RealCall 实例，因此看看其 execute 方法。

```java
public Response execute() throws IOException {
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
    }
    transmitter.timeoutEnter();
    transmitter.callStart();
    try {
        client.dispatcher().executed(this);
        return getResponseWithInterceptorChain();
    } finally {
        client.dispatcher().finished(this);
    }
}
public final class Transmitter {

    public Transmitter(OkHttpClient client, Call call) {
        ...
        this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
    }
    private final AsyncTimeout timeout = new AsyncTimeout() {
        @Override 
        protected void timedOut() {
            cancel();
        }
    };
    public void timeoutEnter() {
        timeout.enter();
    }
    
    public void callStart() {
        this.callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()");
        eventListener.callStart(call);
    }
}
public final class Dispatcher {
	private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
    synchronized void executed(RealCall call) {
        runningSyncCalls.add(call);
    }
}
```

开始计时，当超时后会执行 AsyncTimeout.timeout 方法，不过默认超时时间为 0 ，所以不会超时，至于 ```callStart``` 方法回调 ```eventListener.callStart```  方法。然后执行 ```client.dispatcher().executed``` ，将 RealCall 实例放入 ```runningSyncCalls``` 这个双端队列中去，最后执行 ```getResponseWithInterceptorChain``` 。

```java
Response getResponseWithInterceptorChain() throws IOException {
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
        interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    // 责任链设计模式
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
        originalRequest, this, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());
    boolean calledNoMoreExchanges = false;
    try {
        Response response = chain.proceed(originalRequest);
        if (transmitter.isCanceled()) {
            closeQuietly(response);
            throw new IOException("Canceled");
        }
        return response;
    } catch (IOException e) {
        calledNoMoreExchanges = true;
        throw transmitter.noMoreExchanges(e);
    } finally {
        if (!calledNoMoreExchanges) {
            transmitter.noMoreExchanges(null);
        }
    }
}
```

该方法首先将用户设置的所有 Interceptor 与框架自己的几个 Interceptor 进行组合，注意用户可以设置 interceptors、networkInterceptors 前者在网络连接前执行，后者会在网络连接后执行，然后创建RealInterceptorChain 实例调用其 proceed方法获取到 Response，因此网络请求的主要逻辑就是在这个 proceed 方法中。

```java
public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange) throws IOException {
    // 默认index = 0，如果index超出了拦截器的总长就抛出错误
    if (index >= interceptors.size()) throw new AssertionError();
    calls++;
    // 这里刚才传入的exchange为null
    if (this.exchange != null && !this.exchange.connection().supportsUrl(request.url())) {
        throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
                + " must retain the same host and port");
    }
    // 同上
    if (this.exchange != null && calls > 1) {
        throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
                + " must call proceed() exactly once");
    }
    // 又创建了一个RealInterceptorChain实例，不过其index在原来基础上加了1
    RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,
            index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
    // 取出一个Interceptor实例调用其intercept方法
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    if (exchange != null && index + 1 < interceptors.size() && next.calls != 1) {
        throw new IllegalStateException("network interceptor " + interceptor
                + " must call proceed() exactly once");
    }
    // 如果Interceptor返回了null那么抛出NPE
    if (response == null) {
        throw new NullPointerException("interceptor " + interceptor + " returned null");
    }
    if (response.body() == null) {
        throw new IllegalStateException(
                "interceptor " + interceptor + " returned a response with no body");
    }
    return response;
}
```

该方法内部主要从 interceptors 中取出 index 位置上的一个 Interceptor 实例，然后创建一个 index=index+1 的 RealInterceptorChain 实例 next，最后调用 Interceptor 实例的 interceptor 方法将 next 传入，获取到 Response 返回，那么网络请求重点其实在 interceptor.intercept 方法内，而默认 index 等于0，我们的 OkHttpClient 自己又没有设置 Interceptor，于是会调用到 RetryAndFollowUpInterceptor (负责失败重试以及重定向)实例的 intercept 方法。

```java
public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Transmitter transmitter = realChain.transmitter();
    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
        transmitter.prepareToConnect(request);
        if (transmitter.isCanceled()) {
            throw new IOException("Canceled");
        }
        Response response;
        boolean success = false;
        try {
            response = realChain.proceed(request, transmitter, null);
            // 表示请求成功但是可能是一个重定向响应
            success = true;
        } catch (RouteException e) {
            // 尝试路由连接失败，请求还没发送，判断是否需要进行重试
            if (!recover(e.getLastConnectException(), transmitter, false, request)) {
                throw e.getFirstConnectException();
            }
            continue;
        } catch (IOException e) {
            // 试图与服务器通信失败，请求可能已经发送，判断是否需要进行重试
            boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
            if (!recover(e, transmitter, requestSendStarted, request)) throw e;
            continue;
        } finally {
            // 如果请求没有成功则释放资源
            if (!success) {
                transmitter.exchangeDoneDueToException();
            }
        }
        // 如果上个响应存在(表示上个响应是一个重定向响应，其不会拥有响应体)，构建一个新的Response实例
        // 将上个响应赋值给priorResponse属性
        if (priorResponse != null) {
            response = response.newBuilder()
                    .priorResponse(priorResponse.newBuilder()
                            .body(null)
                            .build())
                    .build();
        }
        // 暂时不理解这个exchange是干什么用的？？
        Exchange exchange = Internal.instance.exchange(response);
        Route route = exchange != null ? exchange.connection().route() : null;
        // 根据响应头判断是否是重定向，如果是就会新建一个Request实例返回
        Request followUp = followUpRequest(response, route);
        // 如果followUp为空也就是没有重定向那么直接返回响应
        if (followUp == null) {
            if (exchange != null && exchange.isDuplex()) {
                transmitter.timeoutEarlyExit();
            }
            return response;
        }
        RequestBody followUpBody = followUp.body();
        // 如果限制只发送一次，那也直接返回响应
        if (followUpBody != null && followUpBody.isOneShot()) {
            return response;
        }
        closeQuietly(response.body());
        if (transmitter.hasExchange()) {
            exchange.detachWithViolence();
        }
        // 重定向请求太多了，就抛出异常
        if (++followUpCount > MAX_FOLLOW_UPS) {
            throw new ProtocolException("Too many follow-up requests: " + followUpCount);
        }
        // 赋值新建的请求，保存上一个响应，最终的响应会包含所有前面重定向的响应
        request = followUp;
        priorResponse = response;
    }
}
public void prepareToConnect(Request request) {
    if (this.request != null) {
        if (sameConnection(this.request.url(), request.url()) && exchangeFinder.hasRouteToTry()) {
            return; // Already ready.
        }
        if (exchange != null) throw new IllegalStateException();

        if (exchangeFinder != null) {
            maybeReleaseConnection(null, true);
            exchangeFinder = null;
        }
    }
    this.request = request;
    this.exchangeFinder = new ExchangeFinder(this, connectionPool, createAddress(request.url()), call, eventListener);
}
```

首先会执行 ```transmitter.prepareToConnect``` 由于最初 ```request``` 为 null ，只是赋值了 ```request``` 和 ```exchangeFinder``` 字段，接着执行 ```realChain.proceed``` 调用下层 Interceptor 实例的 intercept 方法，当下层 Interceptor 抛出异常会判断是否有重试的必要。 当下层返回了一个 Response ，其会根据该 Response 判断是否为重定向响应，如果是就会新建一个 Request 实例，再次请求获取到新的 Response 实例后将原先的 Response赋值给其 priorResponse 属性，以此循环直到请求成功(不再重定向)、超出最大重定向数、抛出不可重试的异常。然后看看 BridgeInterceptor。

```java
public final class BridgeInterceptor implements Interceptor {
    private final CookieJar cookieJar;
    // 这里的CookieJar就是OkHttpClient的CookieJar，默认是一个空实现
    public BridgeInterceptor(CookieJar cookieJar) {
        this.cookieJar = cookieJar;
    }
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request userRequest = chain.request();
        Request.Builder requestBuilder = userRequest.newBuilder();
        RequestBody body = userRequest.body();
        // 如果有请求体，并且请求头如果没有Content-Type、Content-length、Host、Connection就加上
        if (body != null) {
            MediaType contentType = body.contentType();
            if (contentType != null) {
                requestBuilder.header("Content-Type", contentType.toString());
            }
            long contentLength = body.contentLength();
            if (contentLength != -1) {
                requestBuilder.header("Content-Length", Long.toString(contentLength));
                requestBuilder.removeHeader("Transfer-Encoding");
            } else {
                requestBuilder.header("Transfer-Encoding", "chunked");
                requestBuilder.removeHeader("Content-Length");
            }
        }
        if (userRequest.header("Host") == null) {
            requestBuilder.header("Host", hostHeader(userRequest.url(), false));
        }
        if (userRequest.header("Connection") == null) {
            requestBuilder.header("Connection", "Keep-Alive");
        }
        // 然后如果没有Accept-Encoding请求头并且不是请求部分资源，那么加上Accept-Encoding: gzip
        boolean transparentGzip = false;
        if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
            transparentGzip = true;
            requestBuilder.header("Accept-Encoding", "gzip");
        }
        // 从cookieJar中取出Cookie列表
        List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
        if (!cookies.isEmpty()) {
            requestBuilder.header("Cookie", cookieHeader(cookies));
        }
        // 没有UserAgrent就添加为okhttp/3.14.1
        if (userRequest.header("User-Agent") == null) {
            requestBuilder.header("User-Agent", Version.userAgent());
        }
        // 调用下一个Interceptor
        Response networkResponse = chain.proceed(requestBuilder.build());
        // 使用cookieJar保存cookie
        HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
        Response.Builder responseBuilder = networkResponse.newBuilder()
                .request(userRequest);
        if (transparentGzip
                && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
                && HttpHeaders.hasBody(networkResponse)) {
            // 设置对应的响应头
            GzipSource responseBody = new GzipSource(networkResponse.body().source());
            Headers strippedHeaders = networkResponse.headers().newBuilder()
                    .removeAll("Content-Encoding")
                    .removeAll("Content-Length")
                    .build();
            responseBuilder.headers(strippedHeaders);
            String contentType = networkResponse.header("Content-Type");
            // 自动进行解压，不过contentLength变成了-1
            responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
        }
        return responseBuilder.build();
    }
    private String cookieHeader(List<Cookie> cookies) {
        StringBuilder cookieHeader = new StringBuilder();
        for (int i = 0, size = cookies.size(); i < size; i++) {
            if (i > 0) {
                cookieHeader.append("; ");
            }
            Cookie cookie = cookies.get(i);
            cookieHeader.append(cookie.name()).append('=').append(cookie.value());
        }
        return cookieHeader.toString();
    }
}
```

BridgeInterceptor 的逻辑很清晰，其做为应用程序代码与网络代码中间的桥梁，首先根据给用户的 Request 实例创建一个添加了某些请求头的 Request 实例；其次调用了 chain.proceed 执行下一个拦截器；最后将下层返回的 Response 实例构建成用户需要的 Response 实例，此外上述代码还告诉我们如果想要持久化管理 cookie 可以实现 CookieJar 这个接口设置给 OkHttpClient ，接着继续看看下一层 CacheInterceptor。

```java
public final class CacheInterceptor implements Interceptor {

    // 这个cache就是OkHttpClient里面的internalCache
    final InternalCache cache;
    public CacheInterceptor(@Nullable InternalCache cache) {
        this.cache = cache;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        // 如果给OkHttpClient设置了InternalCache，那么从里面获取缓存的响应
        Response cacheCandidate = cache != null
                ? cache.get(chain.request())
                : null;
        long now = System.currentTimeMillis();
        // 这个里面主要是根据请求和缓存的响应判断缓存是否命中
        CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
        Request networkRequest = strategy.networkRequest;
        Response cacheResponse = strategy.cacheResponse;
        if (cache != null) {
            cache.trackResponse(strategy);
        }
        if (cacheCandidate != null && cacheResponse == null) {
            closeQuietly(cacheCandidate.body());
        }
        // 客户端设置了only-if-cached，表示只使用缓存而缓存又没有命中因此直接构建一个Response返回
        if (networkRequest == null && cacheResponse == null) {
            return new Response.Builder()
                    .request(chain.request())
                    .protocol(Protocol.HTTP_1_1)
                    .code(504)
                    .message("Unsatisfiable Request (only-if-cached)")
                    .body(Util.EMPTY_RESPONSE)
                    .sentRequestAtMillis(-1L)
                    .receivedResponseAtMillis(System.currentTimeMillis())
                    .build();
        }
        // 缓存命中，构造一个Response实例并将去掉了body的cacheResponse赋值给该实例的cacheResponse属性
        if (networkRequest == null) {
            return cacheResponse.newBuilder()
                    .cacheResponse(stripBody(cacheResponse))
                    .build();
        }
        Response networkResponse = null;
        try {
            networkResponse = chain.proceed(networkRequest);
        } finally {
            // 发生了异常需要将缓存响应体关闭
            if (networkResponse == null && cacheCandidate != null) {
                closeQuietly(cacheCandidate.body());
            }
        }
        // 如果有缓存响应并且响应码是304，就根据返回的响应和缓存的响应构造一个新的响应并且更新下缓存
        if (cacheResponse != null) {
            if (networkResponse.code() == HTTP_NOT_MODIFIED) {
                Response response = cacheResponse.newBuilder()
                        .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                        .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                        .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                        .cacheResponse(stripBody(cacheResponse))
                        .networkResponse(stripBody(networkResponse))
                        .build();
                networkResponse.body().close();
                cache.trackConditionalCacheHit();
                cache.update(cacheResponse, response);
                return response;
            } else {
                closeQuietly(cacheResponse.body());
            }
        }

        // 响应码不是304，则构造一个新的Response将cacheResponse、networkResponse分别去掉body赋值给
        // cacheResponse和networkResponse
        Response response = networkResponse.newBuilder()
                .cacheResponse(stripBody(cacheResponse))
                .networkResponse(stripBody(networkResponse))
                .build();
        if (cache != null) {
            if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
                // 将最终响应放到cache中
                CacheRequest cacheRequest = cache.put(response);
                return cacheWritingResponse(cacheRequest, response);
            }
            if (HttpMethod.invalidatesCache(networkRequest.method())) {
                try {
                    cache.remove(networkRequest);
                } catch (IOException ignored) {
                }
            }
        }
        return response;
    }
}
public final class CacheStrategy {
    public static class Factory {
        public Factory(long nowMillis, Request request, Response cacheResponse) {
            this.nowMillis = nowMillis;
            this.request = request;
            this.cacheResponse = cacheResponse;
            // 根据缓存响应头提取出一些缓存有关的信息
            if (cacheResponse != null) {
                this.sentRequestMillis = cacheResponse.sentRequestAtMillis();
                this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();
                Headers headers = cacheResponse.headers();
                for (int i = 0, size = headers.size(); i < size; i++) {
                    String fieldName = headers.name(i);
                    String value = headers.value(i);
                    if ("Date".equalsIgnoreCase(fieldName)) {
                        servedDate = HttpDate.parse(value);
                        servedDateString = value;
                    } else if ("Expires".equalsIgnoreCase(fieldName)) {
                        expires = HttpDate.parse(value);
                    } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {
                        lastModified = HttpDate.parse(value);
                        lastModifiedString = value;
                    } else if ("ETag".equalsIgnoreCase(fieldName)) {
                        etag = value;
                    } else if ("Age".equalsIgnoreCase(fieldName)) {
                        ageSeconds = HttpHeaders.parseSeconds(value, -1);
                    }
                }
            }
        }
    }
}
public CacheStrategy get() {
    CacheStrategy candidate = getCandidate();
    if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
        // 如果设置了onlyIfCached那么在CacheInterceptor的intercept方法内部会直接构造一个响应码为504的Response
        return new CacheStrategy(null, null);
    }
    return candidate;
}
private CacheStrategy getCandidate() {
    // 没有缓存响应就直接创建一个没响应的CacheStrategy实例
    if (cacheResponse == null) {
        return new CacheStrategy(request, null);
    }
    // 丢弃缓存响应，如果请求是https并且缺少必要的握手
    if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
    }
    // 如果不应该使用缓存也丢弃缓存响应
    if (!isCacheable(cacheResponse, request)) {
        return new CacheStrategy(request, null);
    }
    CacheControl requestCaching = request.cacheControl();
    // 如果Request包含noCache请求头，或者带上了 If-Modified-Since、If-None-Match两个请求头也丢弃响应
    // 应该带上这两个请求头表示客户端在询问服务端资源是否发生变化，没变化会返回304，因此不应该使用缓存
    if (requestCaching.noCache() || hasConditions(request)) {
        return new CacheStrategy(request, null);
    }
    CacheControl responseCaching = cacheResponse.cacheControl();
    long ageMillis = cacheResponseAge();
    long freshMillis = computeFreshnessLifetime();
    if (requestCaching.maxAgeSeconds() != -1) {
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
    }
    long minFreshMillis = 0;
    if (requestCaching.minFreshSeconds() != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
    }
    long maxStaleMillis = 0;
    if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
    }
    // 根据时间判断是否缓存命中，不知道具体是怎么判断的？
    if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        Response.Builder builder = cacheResponse.newBuilder();
        if (ageMillis + minFreshMillis >= freshMillis) {
            builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
            builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
        return new CacheStrategy(null, builder.build());
    }
    // 添加If-None-Match、If-Modified-Since请求头
    String conditionName;
    String conditionValue;
    if (etag != null) {
        conditionName = "If-None-Match";
        conditionValue = etag;
    } else if (lastModified != null) {
        conditionName = "If-Modified-Since";
        conditionValue = lastModifiedString;
    } else if (servedDate != null) {
        conditionName = "If-Modified-Since";
        conditionValue = servedDateString;
    } else {
        return new CacheStrategy(request, null);
    }
    Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
    Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);
    Request conditionalRequest = request.newBuilder()
            .headers(conditionalRequestHeaders.build())
            .build();
    return new CacheStrategy(conditionalRequest, cacheResponse);
}
```

CacheInterceptor 的主要功能就是从 InternalCache 中取出保存的响应，然后根据请求和缓存的响应判断缓存是否命中，命中就会直接构建一个新的响应返回，如果没命中(由于响应过期)，则会根据缓存响应的 ETag、LastModify 等响应头去构造当前的请求头，这样当服务器判断资源没变化时可以直接返回 304，框架也只需要更新下缓存的响应头就可以直接返回了。OkHttp 为我们提供了一个 Cache 类(内部使用 DiskLruCache 实现)如果我们需要能够缓存只需要进行如下设置.

```kotlin
val client = OkHttpClient.Builder().cache(Cache(cacheFile, 50 * 1000)).build()
```

接着看看下一个拦截器 ConnectInterceptor。

```java
public final class ConnectInterceptor implements Interceptor {
    public final OkHttpClient client;
    public ConnectInterceptor(OkHttpClient client) {
        this.client = client;
    }
    @Override
    public Response intercept(Chain chain) throws IOException {
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        Request request = realChain.request();
        Transmitter transmitter = realChain.transmitter();
        boolean doExtensiveHealthChecks = !request.method().equals("GET");
        Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);
        return realChain.proceed(request, transmitter, exchange);
    }
}
```

ConnectIntercept 代码很少，但是做的事情很多，其会先去 ConnectionPoll 中寻找是否有合适的RealConnection ，如果没有找到会去请求 dns 服务器获取目标 IP 再将目标 IP 封装成一个 Route 实例然后创建一个 RealConnection 接着创建 Socket 实例并发起连接如果是 Https 请求还会发起握手，校验证书，接着构建Http1ExchangeCodec 实例，最后再去构建 Exchange 实例，接着再看看 CallServerInterceptor。

```java
public final class CallServerInterceptor implements Interceptor {
    private final boolean forWebSocket;
    public CallServerInterceptor(boolean forWebSocket) {
        this.forWebSocket = forWebSocket;
    }
    @Override
    public Response intercept(Chain chain) throws IOException {
        RealInterceptorChain realChain = (RealInterceptorChain) chain;
        Exchange exchange = realChain.exchange();
        // 根据Request生成对应的字节数组并且写入到Buffer中
        Request request = realChain.request();
        long sentRequestMillis = System.currentTimeMillis();
        exchange.writeRequestHeaders(request);
        boolean responseHeadersStarted = false;
        Response.Builder responseBuilder = null;
        // 如果请求包含请求体，写入请求体
        if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
            if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
                exchange.flushRequest();
                responseHeadersStarted = true;
                exchange.responseHeadersStart();
                responseBuilder = exchange.readResponseHeaders(true);
            }
            if (responseBuilder == null) {
                if (request.body().isDuplex()) {
                    exchange.flushRequest();
                    BufferedSink bufferedRequestBody = Okio.buffer(
                            exchange.createRequestBody(request, true));
                    request.body().writeTo(bufferedRequestBody);
                } else {
                    BufferedSink bufferedRequestBody = Okio.buffer(
                            exchange.createRequestBody(request, false));
                    request.body().writeTo(bufferedRequestBody);
                    bufferedRequestBody.close();
                }
            } else {
                exchange.noRequestBody();
                if (!exchange.connection().isMultiplexed()) {
                    exchange.noNewExchangesOnConnection();
                }
            }
        } else {
            exchange.noRequestBody();
        }
        if (request.body() == null || !request.body().isDuplex()) {
            // 将Buffer中的数据写给服务端
            exchange.finishRequest();
        }
        if (!responseHeadersStarted) {
            exchange.responseHeadersStart();
        }
        if (responseBuilder == null) {
            // 获取响应头
            responseBuilder = exchange.readResponseHeaders(false);
        }
        Response response = responseBuilder
                .request(request)
                .handshake(exchange.connection().handshake())
                .sentRequestAtMillis(sentRequestMillis)
                .receivedResponseAtMillis(System.currentTimeMillis())
                .build();
        int code = response.code();
        if (code == 100) {
            response = exchange.readResponseHeaders(false)
                    .request(request)
                    .handshake(exchange.connection().handshake())
                    .sentRequestAtMillis(sentRequestMillis)
                    .receivedResponseAtMillis(System.currentTimeMillis())
                    .build();

            code = response.code();
        }
        exchange.responseHeadersEnd(response);
        if (forWebSocket && code == 101) {
            response = response.newBuilder()
                    .body(Util.EMPTY_RESPONSE)
                    .build();
        } else {
            response = response.newBuilder()
                    .body(exchange.openResponseBody(response))
                    .build();
        }
        if ("close".equalsIgnoreCase(response.request().header("Connection"))
                || "close".equalsIgnoreCase(response.header("Connection"))) {
            exchange.noNewExchangesOnConnection();
        }
        if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
            throw new ProtocolException(
                    "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
        }
        return response;
    }
}
```

CallServerInterceptor 真正的进行了网络请求，会根据 Request 实例构建出 Http 请求，获取到 Http 响应后再构建出 HttpResponse，网络请求成功后会接着执行前几个 Interceptor 的剩余代码，这里就不看了。直接回到RealCall.execute。

```java
@Override public Response execute() throws IOException {
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
    }
    transmitter.timeoutEnter();
    transmitter.callStart();
    try {
        client.dispatcher().executed(this);
        return getResponseWithInterceptorChain();
    } finally {
        client.dispatcher().finished(this);
    }
}
void finished(RealCall call) {
    finished(runningSyncCalls, call);
}
private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
        if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
        idleCallback = this.idleCallback;
    }
    boolean isRunning = promoteAndExecute();
    if (!isRunning && idleCallback != null) {
        idleCallback.run();
    }
}
```

可以看出当一次同步请求结束后，会将 RealCall 中队列中移除，然后启动正在等待的异步请求，如果没有异步请求会回调 IdleCallback 。接着看看异步请求过程。

## Call.enqueue

enqueue 方法用于执行异步请求。

```java
public void enqueue(Callback responseCallback) {
    synchronized (this) {
        if (executed) {
            throw new IllegalStateException("Already Executed");
        }
        executed = true;
    }
    transmitter.callStart();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

这里都和 execute 一样只是最后调用了 Dispatcher 的 enqueue 方法，不过传入的是 AsyncCall 实例。

```java
void enqueue(AsyncCall call) {
    synchronized (this) {
        // 将call加入到准备队列中去
        readyAsyncCalls.add(call);
        if (!call.get().forWebSocket) {
            // 刚刚创建的call不是使用webSocket所以进入这里
            AsyncCall existingCall = findExistingCallWithHost(call.host());
            // 目的只是为了统计每个Host有几个AsyncCall
            if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
        }
    }
    promoteAndExecute();
}
// 从正在执行或者等待执行的call队列中取出host属性为host的AsyncCall实例
private AsyncCall findExistingCallWithHost(String host) {
    for (AsyncCall existingCall : runningAsyncCalls) {
        if (existingCall.host().equals(host)) return existingCall;
    }
    for (AsyncCall existingCall : readyAsyncCalls) {
        if (existingCall.host().equals(host)) return existingCall;
    }
    return null;
}
private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));
    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
        for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
            AsyncCall asyncCall = i.next();
            // 如果已经达到最大请求数64就停止执行
            if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
            // 如果每个Host达到了最大请求数5个就跳过该call
            if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.
            i.remove();
            // 每个端口请求计数加1
            asyncCall.callsPerHost().incrementAndGet();
            executableCalls.add(asyncCall);
            runningAsyncCalls.add(asyncCall);
        }
        isRunning = runningCallsCount() > 0;
    }
    for (int i = 0, size = executableCalls.size(); i < size; i++) {
        AsyncCall asyncCall = executableCalls.get(i);
        // 创建一个线程池，然后执行AsyncCall的execute方法
        asyncCall.executeOn(executorService());
    }
    return isRunning;
}
final class AsyncCall extends NamedRunnable {
    ...
    void executeOn(ExecutorService executorService) {
        assert (!Thread.holdsLock(client.dispatcher()));
        boolean success = false;
        try {
            executorService.execute(this);
            success = true;
        } catch (RejectedExecutionException e) {
            ...
        } finally {
            if (!success) {
                client.dispatcher().finished(this); // This call is no longer running!
            }
        }
    }
    @Override
    protected void execute() {
        // 在线程池中执行
        boolean signalledCallback = false;
        transmitter.timeoutEnter();
        try {
            Response response = getResponseWithInterceptorChain();
            signalledCallback = true;
            responseCallback.onResponse(RealCall.this, response);
        } catch (IOException e) {
            if (signalledCallback) {
                Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
            } else {
                responseCallback.onFailure(RealCall.this, e);
            }
        } finally {
            // 端口请求计数减1
            client.dispatcher().finished(this);
        }
    }
}
```

如果调用了 enqueue 发送网络请求，那么最终会在线程池中执行 AsyncCall 的 execute 方法，其内部实现与同步执行基本类似，注意最后会在子线程中直接调用 onResponse ，因此我们不能在 onResponse 里面直接更新 UI 。我们可以写一个 WrapCall 将 Call 进行包装这样就能实现回调在主线程了，代码如下。

```kotlin
class WrapCall(private val call: Call) : Call by call {
    override fun enqueue(responseCallback: Callback) {
        call.enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {
                handler.post {
                    responseCallback.onFailure(call, e)
                }
            }
            override fun onResponse(call: Call, response: Response) {
                handler.post {
                    responseCallback.onResponse(call, response)
                }
            }
        })
    }
    // 保证 handler 只会创建一次
    companion object {
        private val handler = Handler(Looper.getMainLooper())
    }
}
// 外界使用，只要包装下
val call = WrapCall(client.newCall(request))
```

源码分析到这网络流程基本已经清晰，下面再来看看 OkHttp 的连接复用。

## ConnectionPoll

首先需要连接复用需要设置请求头 Connection: Keep-Alive ，这个已经在 BridgeInterceptor 里面设置了，当然如果响应头 Connection: false ，那么连接还是不能复用，连接池的具体实现是 RealConnectionPool ，每次在 ConnectInterceptor 的 intercepte 方法都会尝试着先从连接池中取出一个连接，取不到满足条件的才会新建一个连接。

```java
public final class RealConnectionPool {
    // 这个线程池是专门用来执行清理线程的
    private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
            Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
            new SynchronousQueue<>(), Util.threadFactory("OkHttp ConnectionPool", true));
    // 最大空闲连接数 默认5个
    private final int maxIdleConnections;
    // 最大保存存活的空闲连接时间 默认5分钟
    private final long keepAliveDurationNs;
    // 当向连接池中加入一个连接后会执行清理操作，内部会寻找空闲时间最长的连接，如果其空闲时间
    // 已经超过了最长时间就会将其关闭，不然就等待指定时间
    private final Runnable cleanupRunnable = () -> {
        while (true) {
            long waitNanos = cleanup(System.nanoTime());
            if (waitNanos == -1) return;
            if (waitNanos > 0) {
                long waitMillis = waitNanos / 1000000L;
                waitNanos -= (waitMillis * 1000000L);
                synchronized (RealConnectionPool.this) {
                    try {
                        RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
                    } catch (InterruptedException ignored) {
                    }
                }
            }
        }
    };
    // 双端队列保存当前OkHttpClient的所有连接
    private final Deque<RealConnection> connections = new ArrayDeque<>();
    // 记录了失败的路由
    final RouteDatabase routeDatabase = new RouteDatabase();
    // 清理线程是否正在执行
    boolean cleanupRunning;
    // 首先创建了一个最大空闲连接为5，最大保存存活时间5分钟的连接池
    public RealConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
        this.maxIdleConnections = maxIdleConnections;
        this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);
        if (keepAliveDuration <= 0) {
            throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
        }
    }
    // 获取空闲连接数量
    public synchronized int idleConnectionCount() {
        int total = 0;
        for (RealConnection connection : connections) {
            if (connection.transmitters.isEmpty()) total++;
        }
        return total;
    }
    // 总共的连接数
    public synchronized int connectionCount() {
        return connections.size();
    }
    // Transmitter调用该方法获取连接，内部判断如果连接池有连接满足条件就返回true，不然返回false
    boolean transmitterAcquirePooledConnection(Address address, Transmitter transmitter,
                                               @Nullable List<Route> routes, boolean requireMultiplexed) {
        assert (Thread.holdsLock(this));
        for (RealConnection connection : connections) {
            if (requireMultiplexed && !connection.isMultiplexed()) continue;
            if (!connection.isEligible(address, routes)) continue;
            transmitter.acquireConnectionNoEvents(connection);
            return true;
        }
        return false;
    }

    // 将一个新建的连接放入到连接池中
    void put(RealConnection connection) {
        assert (Thread.holdsLock(this));
        // 如果清理线程还没运行就开始运行
        if (!cleanupRunning) {
            cleanupRunning = true;
            executor.execute(cleanupRunnable);
        }
        // 再将连接放入双端队列中去
        connections.add(connection);
    }

    // 当一个连接从执行中变成了空闲时调用，该方法会唤醒清理线程
    boolean connectionBecameIdle(RealConnection connection) {
        assert (Thread.holdsLock(this));
        if (connection.noNewExchanges || maxIdleConnections == 0) {
            connections.remove(connection);
            return true;
        } else {
            notifyAll(); // Awake the cleanup thread: we may have exceeded the idle connection limit.
            return false;
        }
    }

    // 关闭所有的连接
    public void evictAll() {
        List<RealConnection> evictedConnections = new ArrayList<>();
        synchronized (this) {
            for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
                RealConnection connection = i.next();
                if (connection.transmitters.isEmpty()) {
                    connection.noNewExchanges = true;
                    evictedConnections.add(connection);
                    i.remove();
                }
            }
        }
        for (RealConnection connection : evictedConnections) {
            closeQuietly(connection.socket());
        }
    }
    // 如果可以清理的话就关闭Socket返回0，不然返回需要等待的时间
    long cleanup(long now) {
        int inUseConnectionCount = 0;
        int idleConnectionCount = 0;
        RealConnection longestIdleConnection = null;
        long longestIdleDurationNs = Long.MIN_VALUE;
        synchronized (this) {
            for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
                RealConnection connection = i.next();
                if (pruneAndGetAllocationCount(connection, now) > 0) {
                    inUseConnectionCount++;
                    continue;
                }
                idleConnectionCount++;
                long idleDurationNs = now - connection.idleAtNanos;
                if (idleDurationNs > longestIdleDurationNs) {
                    longestIdleDurationNs = idleDurationNs;
                    longestIdleConnection = connection;
                }
            }
            if (longestIdleDurationNs >= this.keepAliveDurationNs
                    || idleConnectionCount > this.maxIdleConnections) {
                connections.remove(longestIdleConnection);
            } else if (idleConnectionCount > 0) {
                return keepAliveDurationNs - longestIdleDurationNs;
            } else if (inUseConnectionCount > 0) {
                return keepAliveDurationNs;
            } else {
                cleanupRunning = false;
                return -1;
            }
        }
        closeQuietly(longestIdleConnection.socket());
        return 0;
    }
    private int pruneAndGetAllocationCount(RealConnection connection, long now) {
        // 对于Http1.1这个List最多也只有一个元素
        List<Reference<Transmitter>> references = connection.transmitters;
        for (int i = 0; i < references.size(); ) {
            Reference<Transmitter> reference = references.get(i);
            if (reference.get() != null) {
                i++;
                continue;
            }
            // 发现一个泄露的transmitter，这是一个应用程序的bug
            TransmitterReference transmitterRef = (TransmitterReference) reference;
            String message = "A connection to " + connection.route().address().url()
                    + " was leaked. Did you forget to close a response body?";
            Platform.get().logCloseableLeak(message, transmitterRef.callStackTrace);
            references.remove(i);
            connection.noNewExchanges = true; eviction.
            if (references.isEmpty()) {
                connection.idleAtNanos = now - keepAliveDurationNs;
                return 0;
            }
        }
        return references.size();
    }

    // 连接失败时调用将失败的Route放入到RouteDatabase中
    public void connectFailed(Route failedRoute, IOException failure) {
        // Tell the proxy selector when we fail to connect on a fresh connection.
        if (failedRoute.proxy().type() != Proxy.Type.DIRECT) {
            Address address = failedRoute.address();
            address.proxySelector().connectFailed(
                    address.url().uri(), failedRoute.proxy().address(), failure);
        }
        routeDatabase.failed(failedRoute);
    }
}
```

## 总结

不管是同步请求还是异步请求都是通过 Dispatcher 类进行分发，然后经过从上到下5个 Interceptor 才能发起请求，获取到响应后还会经过这5个拦截器然后才将结果返回到外界，典型的责任链模式与 Android 事件分发差不多 。因此 OkHttp 核心就是 Dispatcher、Intercept。

Dispatcher:

内部维护了三个双端队列（同步请求队列、异步请求队列、异步准备队列）、一个线程池（同 CacheThreadPoll ）。

1. 执行同步调用时加入到同步请求队列，请求完毕后移除，然后看看异步准备队列是否为空，不为空就请求。
2. 执行异步调用时判断是否达到最大请求数量，以及最大每个 Host 请求数量，如果都不满，那么加入异步请求队列，如果某个满了，那么加入异步准备队列，当执行完毕后异步准备队列是否为空，不为空就请求。

Interceptor:

1. RetryAndFollowUpInterceptor 用于错误重试，以及重定向。
2. BridgeInterceptor 用于添加请求头(User-Agent、Connection等等)，收到响应的时候可能会进行 GZip 解压。
3. CacheInterceptor 进行缓存管理，默认不带缓存，如果需要缓存可以给 OkHttpClient 设置 cache 属性，可以使用 OkHttp 内置的 Cache 类。
4. ConnectInterceptor 进行连接，首先从连接池中取出可以复用的连接，取不到就新建一个然后通过InetAddress 获取到域名对于的IP地址，然后创建 Socket 与服务端进行连接，连接成功后如果是Https请求还会进行握手验证证书操作。
5. CallServerInterceptor 用于真正的发起请求，从 Socket 获取的输出流写入请求数据，从输入流中读取到响应数据。