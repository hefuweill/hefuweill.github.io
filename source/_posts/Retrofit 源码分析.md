---
title: Retrofit 源码分析
date: 2019-05-13 13:23:10
type: "Android"
---

## 前言

对于 Retrofit 来说网络请求本质上是 OkHttp 完成的，其仅负责网络请求接口的封装，上篇文章简单分析了 OkHttp 的源码，本篇文章来分析下 Retrofit 的源码实现，从其的简单使用方式出发。注：[仓库地址](https://github.com/square/retrofit)

<!--more-->

## 简单使用

首先需要定义一个网络请求的 API 接口，内部配合注解声明了请求方法、请求地址等信息，然后需要创建 Retrofit 实例，接着调用其 create 方法创建一个 API 接口实现类的一个实例，然后就可以通过调用该实例的指定方法拿到Call 实例，接下来就和 OkHttp 的使用方式没有什么不同了。

```java
interface DoubanAPI {
    @GET("v2/movie/in_theaters")
    fun inTheaters(@Query("start") start: Int) : Call<ResponseBody>
}

fun main() {
    val retrofit = Retrofit.Builder()
        .baseUrl("https://douban.uieee.com/")
        .validateEagerly(true)
        .build()
    val api = retrofit.create(DoubanAPI::class.java)
    val call = api.inTheaters(0)
    val response = call.execute()
    if (response.isSuccessful) {
        println("Request successful: ${response.body()!!.string()}")
    } else {
        println("Request failed: ${response.message()}")
    }
}
```

接下来按照 Retrofit 实例的创建、retrofit.create 方法的调用、api.inTheater 方法的调用、call.execute 方法的调用来分析 Retrofit 的源码。

## Retrofit 实例的创建

这里使用了建造者模式构建 Retrofit 实例因此首先看看 Builder 的构造方法。

```java
public static final class Builder {
    private final Platform platform;
    private @Nullable okhttp3.Call.Factory callFactory;
    private @Nullable HttpUrl baseUrl;
    private final List<Converter.Factory> converterFactories = new ArrayList<>();
    private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
    private @Nullable Executor callbackExecutor;
    private boolean validateEagerly;
    Builder(Platform platform) {
        this.platform = platform;
    }
    public Builder() {
        // 通过反射来判断是Android还是Java8
        this(Platform.get());
    }
}
```

接着又调用了 baseUrl 方法用来设置当前 Retrofit 实例的 baseUrl 。

```java
public Builder baseUrl(String baseUrl) {
    checkNotNull(baseUrl, "baseUrl == null");
    return baseUrl(HttpUrl.get(baseUrl));
}
public Builder baseUrl(HttpUrl baseUrl) {
    checkNotNull(baseUrl, "baseUrl == null");
    List<String> pathSegments = baseUrl.pathSegments();
    // 要求baseUrl必须要以/结尾
    if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
        throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
    }
    this.baseUrl = baseUrl;
    return this;
}
```

然后就直接执行 build 方法构造 Retrofit 实例。

```java
public Retrofit build() {
    // baseUrl是必须的，就算没用也得设置一个
    if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
    }
    okhttp3.Call.Factory callFactory = this.callFactory;
    // 如果没有设置callFactory就设置一个OkHttpClient实例赋值给callFactory
    if (callFactory == null) {
        callFactory = new OkHttpClient();
    }
    Executor callbackExecutor = this.callbackExecutor;
    // 如果没有设置callbackExecutor就设置，如果是Android会设置一个MainThreadExecutor，用于将回调切换到主线程执行
    if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
    }
    // 将外界添加的CallAdapter.Factory与系统自带的CallAdapter.Factory合并，比如RXjava提供的RxJava2CallAdapterFactory，注意用户的优先
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    // 在Android中如果API>=24会拥有CompletableFutureCallAdapterFactory、ExecutorCallAdapterFactory两个实例
    callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
    // 添加ConvertFactory，API24会默认拥有BuiltInConverters、OptionalConverterFactory
    List<Converter.Factory> converterFactories = new ArrayList<>(
            1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());
    converterFactories.add(new BuiltInConverters());
    converterFactories.addAll(this.converterFactories);
    converterFactories.addAll(platform.defaultConverterFactories());
    // 其中validateEagerly表示是否急切的验证方法的合法性
    return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
            unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
}
```

到此为止 Retrofit 实例已经创建完毕，接下来看看其 create 方法。

## Retrofit.create

create 方法内部做的事情主要有注解解析，动态代理，先来看看注解解析过程。

### 注解解析

```java
public <T> T create(final Class<T> service) {
    // 判断下传入的Class实例是否是接口，并且判断其是否继承了其它接口
    Utils.validateServiceInterface(service);
    // 如果该属性为true，在create方法执行的时候就会检查接口方法的合法性，不然要等到调用指定方法的时候才会检查
    if (validateEagerly) {  
        eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];
            @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
                throws Throwable {
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
                return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
          }
        });
}
private void eagerlyValidateMethods(Class<?> service) {
    Platform platform = Platform.get();
    for (Method method : service.getDeclaredMethods()) {
        // 如果不是默认方法就执行加载
        if (!platform.isDefaultMethod(method)) {
            loadServiceMethod(method);
        }
    }
}
ServiceMethod<?> loadServiceMethod(Method method) {
    // 从缓存中取，取到就直接返回(eagerlyValidateMethods是取不到的)
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;
    synchronized (serviceMethodCache) {
        // 再取一次，并且加了锁，同步问题，没加锁取不到可能是由于线程可见性导致的
        result = serviceMethodCache.get(method);
        if (result == null) {
            result = ServiceMethod.parseAnnotations(this, method);
            serviceMethodCache.put(method, result);
        }
    }
    return result;
}
```

接着调用了 parseAnnotations 用于解析接口中每一个方法上的注解。

```java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(method,
                "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) {
        throw methodError(method, "Service methods cannot return void.");
    }
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}
// RequestFactory.java
static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
}
// 读取了方法的注解，方法参数类型，方法参数注解
Builder(Retrofit retrofit, Method method) {
    this.retrofit = retrofit;
    this.method = method;
    this.methodAnnotations = method.getAnnotations();
    this.parameterTypes = method.getGenericParameterTypes();
    this.parameterAnnotationsArray = method.getParameterAnnotations();
}
RequestFactory build() {
    // 首先解析了每个方法上面的注解
    for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
    }
    // 必须要有请求方法不然抛出错误
    if (httpMethod == null) {
        throw methodError(method, "HTTP method annotation is required (e.g., @GET, @POST, etc.).");
    }
    // 如果这个请求不需要有请求体(根据请求方法判断)，但是设置了Multipart或者isFormEncoded就抛出错误
    if (!hasBody) {
        if (isMultipart) {
          throw methodError(method,
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
            throw methodError(method, "FormUrlEncoded can only be specified on HTTP methods with "
                + "request body (e.g., @POST).");
            }
        }
    }
    // 获取方法参数数量
    int parameterCount = parameterAnnotationsArray.length;
    parameterHandlers = new ParameterHandler<?>[parameterCount];
    for (int p = 0; p < parameterCount; p++) {
        parameterHandlers[p] = parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p]);
    }
    if (relativeUrl == null && !gotUrl) {
        throw methodError(method, "Missing either @%s URL or @Url parameter.", httpMethod);
    }
    if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError(method, "Non-body HTTP method cannot contain @Body.");
    }
    if (isFormEncoded && !gotField) {
        throw methodError(method, "Form-encoded method must contain at least one @Field.");
    }
    if (isMultipart && !gotPart) {
        throw methodError(method, "Multipart method must contain at least one @Part.");
    }
    return new RequestFactory(this);
}
```

build 方法内部又分别解析了每个方法上面的注解和每个方法参数上面的注解。

#### 解析方法上的注解

来看看 parseMethodAnnotation 这个方法内部干了些什么。

```java
private void parseMethodAnnotation(Annotation annotation) {
    if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
    } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
    } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
    } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
    } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
    } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
    } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
    } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
    } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
            throw methodError(method, "@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
    } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
            throw methodError(method, "Only one encoding annotation is allowed.");
        }
        isMultipart = true;
    } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
            throw methodError(method, "Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
    }
}
```

该方法解析了所有接口方法上面的注释，其中 Mutipart、FormUrlEncoded 这里处理比较简单就是两者不能共存，DELETE、GET、HEAD、PATCH、POST、PUT、OPTIONS 处理方式都一样都是直接调用了parseHttpMethodAndPath 。

```java
private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
    if (this.httpMethod != null) {
        throw methodError(method, "Only one HTTP method is allowed. Found: %s and %s.",
                this.httpMethod, httpMethod);
    }
    // 两成员赋值，后面会使用到
    this.httpMethod = httpMethod;
    this.hasBody = hasBody;
    // 如果没给注解设置值就直接返回
    if (value.isEmpty()) {
        return;
    }
    // 如果url中包含?号那么就相当于有请求参数
    int question = value.indexOf('?');
    if (question != -1 && question < value.length() - 1) {
        // 将查询参数字符串切割出来
        String queryParams = value.substring(question + 1);
        // 查询参数里面不允许使用{参数}的方式，取代方法是使用@Query注解
        Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
        if (queryParamMatcher.find()) {
            throw methodError(method, "URL query string \"%s\" must not have replace block. "
                    + "For dynamic query parameters use @Query.", queryParams);
        }
    }
    // 赋值相对路径
    this.relativeUrl = value;
    // 将url中的{参数}找出来
    this.relativeUrlParamNames = parsePathParameters(value);
}
static Set<String> parsePathParameters(String path) {
    Matcher m = PARAM_URL_REGEX.matcher(path);
    Set<String> patterns = new LinkedHashSet<>();
    while (m.find()) {
    	patterns.add(m.group(1));
    }
    return patterns;
}
```

然后看看 HTTP 注解的解析过程，与 GET 相比起不同点只在于 parseHttpMethodAndPath 的方法入参都是从注解实例中取出，我们可以这么定义 HTTP 注解。

```kotlin
interface DoubanAPI {
    @HTTP(method = "GET", path = "v2/movie/in_theaters")
    fun inTheaters() : Call<ResponseBody>
}
```

接着看看 Headers 注解的解析，首先判断了 Headers 注解的值是否设置，如果没设置就抛出错误，然后调用 parseHeaders 进行解析。

```java
private Headers parseHeaders(String[] headers) {
    Headers.Builder builder = new Headers.Builder();
    for (String header : headers) {
        int colon = header.indexOf(':');
        if (colon == -1 || colon == 0 || colon == header.length() - 1) {
            throw methodError(method,
              "@Headers value must be in the form \"Name: Value\". Found: \"%s\"", header);
        }
        String headerName = header.substring(0, colon);
        String headerValue = header.substring(colon + 1).trim();
        if ("Content-Type".equalsIgnoreCase(headerName)) {
            try {
                contentType = MediaType.get(headerValue);
            } catch (IllegalArgumentException e) {
                throw methodError(method, e, "Malformed content type: %s", headerValue);
            }
        } else {
            builder.add(headerName, headerValue);
        }
    }
    return builder.build();
}
```

#### 解析方法参数上的注解

RequestFactory.Builder 的 build 方法中在解析完方法上面注解后，又会调用到 parseParameter 解析方法每个参数上的注解，注意虽然每个参数可以有多个注解，但是每个参数只能拥有一个 Retrofit 注解。

```java
// 其中p指代该参数是方法的第几个参数，parameterTypes[p]表示当前参数的参数类型，
// parameterAnnotationsArray[p]表示当前参数的注解是个数组
parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p]);

private ParameterHandler<?> parseParameter(
        int p, Type parameterType, @Nullable Annotation[] annotations) {
    ParameterHandler<?> result = null;
    if (annotations != null) {
        for (Annotation annotation : annotations) {
            ParameterHandler<?> annotationAction =
                parseParameterAnnotation(p, parameterType, annotations, annotation);
            if (annotationAction == null) {
                continue;
            }
            if (result != null) {
                throw parameterError(method, p,
                  "Multiple Retrofit annotations found, only one allowed.");
            }
            result = annotationAction;
        }
    }
    // 方法的每个参数都必须有且仅有一个Refrofit注解
    if (result == null) {
        throw parameterError(method, p, "No Retrofit annotation found.");
    }
    return result;
}
```

接着又调用到了 parseParameterAnnotation 这个方法非常长有400行左右，就不展开了只是说说其大概做了些什么. 注意里面判断了一个不常用的注解 @QueryName 这个注解表示在 Url 后面拼加一个只有 name 没有 value 的字符串。

- @Url，则需要确保以下几点，最终会创建一个 ParameterHandler.RelativeUrl 实例返回。

1. 不能出现多个 @Url 注解。
2. @Url 不能和 @Path 共存。
3. @Url 不能声明在 @Query 后。
4. @Url 不能声明在 @QueryName 后。
5. @Url 不能声明在 @QueryMap 后。
6. @Url 使用了就不能再在请求方法后面加上相对地址。
7. 检测参数类型必须是 HttpUrl、String、URI、Uri 中的一种不然抛出错误。

- @Path，则需要确保以下几点，最终会创建一个 ParameterHandler.Path 实例返回。

1. @Path 不能声明在 @Query 后面。
2. @Path 不能声明在 @QueryName 后面。
3. @Path 不能声明在 @QueryMap 后面。
4. @Path 不能和 @Url共存。
5. @Path 使用了就必须要在请求方法后面加上相对地址。
6. 刚才解析方法上 Url 的时候会把所有{参数}解析出来放到 relativeUrlParamNames 这个 Set 中去，如果这个集合中不存在 Path.value 就会报错。

* @Query，则会根据参数类型最终返回 ParameterHandler.Query<>(name, converter, encoded).iterable()、 ParameterHandler.Query<>(name, converter, encoded).array() 、ParameterHandler.Query<>(name, converter, encoded) 三者之一，就这告诉我们 @Query 修饰的参数类型可以是一个容器或者数组，如下所示。

    ```kotlin
    interface DoubanAPI {
        @GET("v2/movie/in_theaters")
        fun inTheaters(@Query("keyword") List<String> keywords) : Call<ResponseBody>
    }
    val list = ArrayList<String>()
    list.add("keyword1")
    list.add("keyword2")
    val api = retrofit.create(DoubanAPI::class.java)
    val call = api.inTheaters(list)
    // 最终的url为https://api.douban.com/v2/movie/in_theaters?keyword=keyword1&keyword=keyword2
    ```

* @ QueryName，则会根据参数类型最终返回 ParameterHandler.QueryName<>(converter, encoded).iterable()、ParameterHandler.QueryName<>(converter, encoded).array()、 ParameterHandler.QueryName<>(converter, encoded)三者之一，跟 @Query 基本一样。

* @QueryMap，首先会检查下参数类型是否是 Map，然后还会检查下 Map 的 key 是否是 String，最后会封装成 ParameterHandler.QueryMap 返回。

* @Header，最终也会根据参数类型转换为 ParameterHandler.Header<>(name, converter).iterable()、 ParameterHandler.Header<>(name, converter).array()、ParameterHandler.Header<>(name, converter) 之一，这里也能看出 @Header 注解支持容器(内部类型要是 String )或者数组。
* @HeaderMap，解析步骤和 @QueryMap 一样，最后返回 ParameterHandler.HeaderMap 。

* @Field 首先会检查是否已经设置了 FormUrlEncoded，没设置就报错也会根据参数类型最后返回  ParameterHandler.Field<>(name, converter, encoded).iterable()、ParameterHandler.Field<>(name, converter, encoded).array()、ParameterHandler.Field 三者之一。
* @FieldMap 与 @QueryMap 判断一致最后会返回一个 ParameterHandler.FieldMap 实例。

- @Part 首先检查是否设置了 Multipart 注解，没设置就报错，然后判断注解的 value 属性是否为空，接着根据类型返回 ParameterHandler.RawPart.INSTANCE.iterable()、 ParameterHandler.RawPart.INSTANCE.array()、ParameterHandler.RawPart.INSTANCE 之一，如果 value 属性不为空会先添加几个 Header，然后根据类型返回 ParameterHandler.Part<>(headers, converter).iterable()、ParameterHandler.Part<>(headers, converter).array()、ParameterHandler.Part<>(headers, converter)。
- @PartMap 与 @QueryMap判断基本一致，最终返回 ParameterHandler.PartMap。
- @Body 如果设置了@FormUrlEncoded 或者 @Multipart就报错，否则返回一个 ParameterHandler.Body 实例。

至此方法上面和请求参数上面的注解已经都解析完毕了。

### HttpServiceMethod 实例的生成

接着调用 HttpServiceMethod.parseAnnotations 获取 ServiceMethod 实例将其放入到 serviceMethodCache 中。

```kotlin
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
            Retrofit retrofit, Method method, RequestFactory requestFactory) {
    CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method);
    Type responseType = callAdapter.responseType();
    if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError(method, "'"
                + Utils.getRawType(responseType).getName()
                + "' is not a valid response body type. Did you mean ResponseBody?");
    }
    if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
        throw methodError(method, "HEAD method must use Void as response type.");
    }
    Converter<ResponseBody, ResponseT> responseConverter =
            createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    return new HttpServiceMethod<>(requestFactory, callFactory, callAdapter, responseConverter);
}
```

#### CallAdapter 实例的获取

然后调用到了 createCallAdapter 用于获取能处理该返回类型的 CallAdapter 。

```java
private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
      Retrofit retrofit, Method method) {
    Type returnType = method.getGenericReturnType();
    Annotation[] annotations = method.getAnnotations();
    try {
        return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
    } catch (RuntimeException e) {
        throw methodError(method, e, "Unable to create call adapter for %s", returnType);
    }
}
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
}
public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
  Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");
    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
        CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
        if (adapter != null) {
            return adapter;
        }
    }
    StringBuilder builder = new StringBuilder("Could not locate call adapter for ")
        .append(returnType)
        .append(".\n");
    if (skipPast != null) {
        builder.append("  Skipped:");
        for (int i = 0; i < start; i++) {
            builder.append("\n   * ").append(callAdapterFactories.get(i).getClass().getName());
        }
        builder.append('\n');
    }
    builder.append("  Tried:");
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
        builder.append("\n   * ").append(callAdapterFactories.get(i).getClass().getName());
    }
    throw new IllegalArgumentException(builder.toString());
}
```

可以看出 nextCallAdapter 会在 Retrofit 的 callAdapterFactories 里面遍历查找能处理该返回类型的CallAdapter，而 callAdapterFactories 在文章一开始已经说过了API24及以上默认包含 `CompletableFutureCallAdapterFactory`、`DefaultCallAdapterFactory ` 两个CallAdapterFactory，也就是说默认方法返回值只能是 **CompletableFuture** 或者 **retrofit2.Call** ，其余都将报错。

#### Converter 实例的获取

HttpServiceMethod.parseAnnotations 后面接着又会调用 createResponseConverter 获取 Convert 实例。

```java
private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
      Retrofit retrofit, Method method, Type responseType) {
    Annotation[] annotations = method.getAnnotations();
    try {
        return retrofit.responseBodyConverter(responseType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(method, e, "Unable to create converter for %s", responseType);
    }
}
public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);
}
public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
            @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    checkNotNull(type, "type == null");
    checkNotNull(annotations, "annotations == null");
    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
        Converter<ResponseBody, ?> converter =
                converterFactories.get(i).responseBodyConverter(type, annotations, this);
        if (converter != null) {
            return (Converter<ResponseBody, T>) converter;
        }
    }
    StringBuilder builder = new StringBuilder("Could not locate ResponseBody converter for ")
            .append(type)
            .append(".\n");
    if (skipPast != null) {
        builder.append("  Skipped:");
        for (int i = 0; i < start; i++) {
            builder.append("\n   * ").append(converterFactories.get(i).getClass().getName());
        }
        builder.append('\n');
    }
    builder.append("  Tried:");
    for (int i = start, count = converterFactories.size(); i < count; i++) {
        builder.append("\n   * ").append(converterFactories.get(i).getClass().getName());
    }
    throw new IllegalArgumentException(builder.toString());
}
```

跟获取 CallAdapter 流程基本一样，其会在 converterFactories 里面遍历寻找是否有 Convert 可以将ResponseBody 转化为 type 类型，API>=24会默认拥有 BuiltInConverters、OptionalConverterFactory 两个ConvertFactory，首先看看 BuiltInConverters 。

##### BuiltInConverters

代码如下：

```java
// BuiltInConverters.java
final class BuiltInConverters extends Converter.Factory {
    private boolean checkForKotlinUnit = true;
    @Override
    public @Nullable
    Converter<ResponseBody, ?> responseBodyConverter(
            Type type, Annotation[] annotations, Retrofit retrofit) {
        if (type == ResponseBody.class) {
            return Utils.isAnnotationPresent(annotations, Streaming.class)
                    ? StreamingResponseBodyConverter.INSTANCE
                    : BufferingResponseBodyConverter.INSTANCE;
        }
        if (type == Void.class) {
            return VoidResponseBodyConverter.INSTANCE;
        }
        if (checkForKotlinUnit) {
            try {
                if (type == Unit.class) {
                    return UnitResponseBodyConverter.INSTANCE;
                }
            } catch (NoClassDefFoundError ignored) { 
                checkForKotlinUnit = false;
            }
        }
        return null;
    }
}
static final class StreamingResponseBodyConverter
        implements Converter<ResponseBody, ResponseBody> {
    static final StreamingResponseBodyConverter INSTANCE = new StreamingResponseBodyConverter();
    @Override
    public ResponseBody convert(ResponseBody value) {
        return value;
    }
}
static final class BufferingResponseBodyConverter
            implements Converter<ResponseBody, ResponseBody> {
    static final BufferingResponseBodyConverter INSTANCE = new BufferingResponseBodyConverter();
    @Override
    public ResponseBody convert(ResponseBody value) throws IOException {
        try {
            // 全部读入内存
            return Utils.buffer(value);
        } finally {
            value.close();
        }
    }
}
static ResponseBody buffer(final ResponseBody body) throws IOException {
    Buffer buffer = new Buffer();
    // 将body的内容全部读入到buffer中去
    body.source().readAll(buffer);
    return ResponseBody.create(body.contentType(), body.contentLength(), buffer);
}
static final class VoidResponseBodyConverter implements Converter<ResponseBody, Void> {
        static final VoidResponseBodyConverter INSTANCE = new VoidResponseBodyConverter();
    @Override
    public Void convert(ResponseBody value) {
        value.close();
        return null;
    }
}
```

- 如果目标转化类型为 ResponseBody 并且方法上有 @Stream 注解，那么会原封不动的返回。
- 如果目标转化类型为 ResponseBody 并且方法上没有 @Stream 注解，那么会将 ResponseBody 整个读入内存(注意当文件过大可能会 OOM)。
- 如果目标转化类型为 Void，那么将 ResponseBody 直接关闭，外界拿到的响应体就是 null。
- 如果目标转化类型为 Unit，那么跟 Void 一样。

##### OptionalConverterFactory

代码如下：

```java
final class OptionalConverterFactory extends Converter.Factory {
    static final Converter.Factory INSTANCE = new OptionalConverterFactory();
    @Override
    public @Nullable
    Converter<ResponseBody, ?> responseBodyConverter(
            Type type, Annotation[] annotations, Retrofit retrofit) {
        if (getRawType(type) != Optional.class) {
            return null;
        }
        Type innerType = getParameterUpperBound(0, (ParameterizedType) type);
        Converter<ResponseBody, Object> delegate =
                retrofit.responseBodyConverter(innerType, annotations);
        return new OptionalConverter<>(delegate);
    }
}
```

很明显只能转化为 Optional，并且如果是 Optional 还会再检查其持有类型，至此通过 requestFactory (解析注解获取)、CallAdapter (根据方法返回值寻找)、Convert (将 ResponseBody 转化为指定类型寻找)、Call.Factory(默认设置是新建的 OkHttpClient 实例，可以传入自己的 Call.Factory )构建出 HttpServiceMethod 实例，然后将其放入 `serviceMethodCache `中，到此 `eagerlyValidateMethods` 分析完毕，接着回到 retrofit.create 方法。

```java
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
        eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[]{service},
            new InvocationHandler() {
                private final Platform platform = Platform.get();
                private final Object[] emptyArgs = new Object[0];
                @Override
                public Object invoke(Object proxy, Method method, @Nullable Object[] args)
                        throws Throwable {
                    if (method.getDeclaringClass() == Object.class) {
                        return method.invoke(this, args);
                    }
                    if (platform.isDefaultMethod(method)) {
                        return platform.invokeDefaultMethod(method, service, proxy, args);
                    }
                    return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
                }
            });
}
```

这里就是一个动态代理，当调用 create 方法返回实例的任何方法都会直接回调 invoke 方法。

## DoubanAPI.inTheater

上述动态代理内部判断如果调用的方法来自 Object 那么直接调用，如果调用的是默认方法在 Android 中会直接抛出错误，此外就去获取 ServiceMethod 实例调用其 invoke 方法，由于我们刚才已经把所有ServiceMethod 都放入到了 serviceMethodCache 中因此直接取出执行就行了。接着看 HttpServiceMethod.invoke 。

```java
@Override final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
}
```

如果按照本文刚开始的例子那么 callAdapter 由 DefaultCallAdapterFactory.get 获取。

```java
public @Nullable CallAdapter<?, ?> get(
      Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
        return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
        @Override public Type responseType() {
            return responseType;
        }
        @Override public Call<Object> adapt(Call<Object> call) {
            return call;
        }
    };
}
```

相对于直接返回给外界了一个 OkHttpCall 实例，继续看看该类的 execute 方法。

```java
public Response<T> execute() throws IOException {
        okhttp3.Call call;
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already executed.");
        executed = true;
        if (creationFailure != null) {
            if (creationFailure instanceof IOException) {
                throw (IOException) creationFailure;
            } else if (creationFailure instanceof RuntimeException) {
                throw (RuntimeException) creationFailure;
            } else {
                throw (Error) creationFailure;
            }
        }
        call = rawCall;
        if (call == null) {
            try {
                call = rawCall = createRawCall();
            } catch (IOException | RuntimeException | Error e) {
                throwIfFatal(e); //  Do not assign a fatal error to creationFailure.
                creationFailure = e;
                throw e;
            }
        }
    }
    if (canceled) {
        call.cancel();
    }
    return parseResponse(call.execute());
}
```

这个 OkHttpCall 与 RealCall 一致都只能被执行一次，先不去管响应，接着调用 createRawCall 创建 RealCall 实例。

```java
private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
        throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
}
```

这里的 callFactoty 一般也就是一个 OkHttpClient 实例，来看看 requestFactory.create 是如何创建一个 Request 实例。

```java
okhttp3.Request create(Object[] args) throws IOException {
    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;
    int argumentCount = args.length;
    if (argumentCount != handlers.length) {
        throw new IllegalArgumentException("Argument count (" + argumentCount
                + ") doesn't match expected count (" + handlers.length + ")");
    }
    // 将通过注解解析到的内容加入到RequestBuilder中
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl,
            headers, contentType, hasBody, isFormEncoded, isMultipart);
    List<Object> argumentList = new ArrayList<>(argumentCount);
    // 调用每个ParameterHandler实例的apply方法
    for (int p = 0; p < argumentCount; p++) {
        argumentList.add(args[p]);
        handlers[p].apply(requestBuilder, args[p]);
    }
    return requestBuilder.get()
            .tag(Invocation.class, new Invocation(method, argumentList))
            .build();
}
```

inTheaters 方法有一个参数，根据上述源码我们知道其会创建一个 ParameterHandler.Query 实例，看看其apply 方法。

```java
// 这里的valueConverter其实是遍历了converterFactories然后调用stringConverter，如果我们没设置那么默认
// valueConverter就是BuiltInConverters里面的ToStringConverter实例
static final class Query<T> extends ParameterHandler<T> {
    private final String name;
    private final Converter<T, String> valueConverter;
    private final boolean encoded;
    Query(String name, Converter<T, String> valueConverter, boolean encoded) {
        this.name = checkNotNull(name, "name == null");
        this.valueConverter = valueConverter;
        this.encoded = encoded;
    }
    @Override
    void apply(RequestBuilder builder, @Nullable T value) throws IOException {
        if (value == null) return; // Skip null values.
        String queryValue = valueConverter.convert(value);
        if (queryValue == null) return; // Skip converted but null values
        builder.addQueryParam(name, queryValue, encoded);
    }
}
static final class ToStringConverter implements Converter<Object, String> {
    static final ToStringConverter INSTANCE = new ToStringConverter();
    @Override
    public String convert(Object value) {
        return value.toString();
    }
}
```

很明显 ParameterHandler.Query 只是添加了一个请求参数，接着调用 requestBuilder.get 去真正的创建  okhttp3.Request.Builder 实例。

```java
Request.Builder get() {
    HttpUrl url;
    HttpUrl.Builder urlBuilder = this.urlBuilder;
    if (urlBuilder != null) {
        url = urlBuilder.build();
    } else {
        // 组合Url
        url = baseUrl.resolve(relativeUrl);
        if (url == null) {
            throw new IllegalArgumentException(
                    "Malformed URL. Base: " + baseUrl + ", Relative: " + relativeUrl);
        }
    }
    RequestBody body = this.body;
    if (body == null) {
        if (formBuilder != null) {
            body = formBuilder.build();
        } else if (multipartBuilder != null) {
            body = multipartBuilder.build();
        } else if (hasBody) {
            body = RequestBody.create(null, new byte[0]);
        }
    }
    MediaType contentType = this.contentType;
    if (contentType != null) {
        if (body != null) {
            body = new ContentTypeOverridingRequestBody(body, contentType);
        } else {
            requestBuilder.addHeader("Content-Type", contentType.toString());
        }
    }
    return requestBuilder
            .url(url)
            .method(method, body);
}
```

至此创建 Request 实例成功，接着调用了 call.execute 方法执行网络请求，接下来看看其是如何解析响应的。

```java
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
    // 首先移除了响应体，以便于长时间保存Response
    rawResponse = rawResponse.newBuilder()
            .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
            .build();
    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
        try {
            // 将响应体全部读入内存，封装成一个错误的响应
            ResponseBody bufferedBody = Utils.buffer(rawBody);
            return Response.error(bufferedBody, rawResponse);
        } finally {
            rawBody.close();
        }
    }
    if (code == 204 || code == 205) {
        rawBody.close();
        // 204、205无响应体封装成一个成功的响应
        return Response.success(null, rawResponse);
    }
    ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
    try {
        T body = responseConverter.convert(catchingBody);
        return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
        catchingBody.throwIfCaught();
        throw e;
    }
}
```

最终调用 responseConverter 的 convert 方法将 ResponseBody 实例转化为指定的类型，由于本例中 inTheaters 方法返回值为 Call\<ResponseBody>，因此这里的 T 就是 ResponseBody 类型，而 responseConverter 就是 BufferingResponseBodyConverter 实例。

```java
static final class BufferingResponseBodyConverter
            implements Converter<ResponseBody, ResponseBody> {
    static final BufferingResponseBodyConverter INSTANCE = new BufferingResponseBodyConverter();
    @Override
    public ResponseBody convert(ResponseBody value) throws IOException {
        try {
            return Utils.buffer(value);
        } finally {
            value.close();
        }
    }
}
```

内部直接将所有 ResponseBody 全部读入内存，然后新建一个 Response 实例返回，parseResponse 最后再调用 Response.success 构造一个成功的响应返回给外界，至此 Retrofit 的基本工作流程已经梳理完毕。下面来看看CallAdapter 和 Convert 的作用。

## CallAdapter、Converter

首先看看 CallAdapter，下面是它的接口定义。

```java
public interface CallAdapter<R, T> {

    // 返回一个代理了call的实例，不需要代理就原样返回好了
    T adapt(Call<R> call);
    
    // 返回T所对应的类型，{@code Call<Repo>} is {@code Repo}
    Type responseType();
    
    abstract class Factory {
        // 返回一个能处理接口返回类型的CallAdapter，如果不能处理该类型就返回null
        public abstract @Nullable
        CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
                              Retrofit retrofit);
                              
        // 提取指定index位置泛型参数类型的上界
        // index 1 of {@code Map<String, ? extends Runnable>} returns {@code Runnable}.
        protected static Type getParameterUpperBound(int index, ParameterizedType type) {
            return Utils.getParameterUpperBound(index, type);
        }
        // 获取原始的类型 {@code List<? extends Runnable>} returns {@code List.class}.
        protected static Class<?> getRawType(Type type) {
            return Utils.getRawType(type);
        }
    }
}
```

在创建 HttpServiceMethod 的时候会遍历所有的 CallAdapter.Factory 并调用其 get 方法，一旦某个 CallAdapter.Factory 的 get 方法不返回 null 就停止遍历，如果所有的都返回 null，那么会报错。如果成功返回CallAdapter 实例 callAdapter，那么接着调用 callAdapter 的 responseType 方法获取到泛型类型，接着根据这个泛型类型去 ConvertFactory 列表中寻找可以将 ResponseBody 转化为该泛型类型的 Converter。最后在内部创建 OkHttpCall 后会调用 adapt 方法。总结下就是在 get 方法中判断 returnType 是否能处理，如果能处理的话， adapt 方法必须要将 call 转化为 returnType 类型的实例，接着看看 Converter，下面是它的定义。

```java
public interface Converter<F, T> {
    @Nullable
    T convert(F value) throws IOException;
    abstract class Factory {
        // 返回一个能将ResponseBody转化为type类型的Converter，如果不能转换那么返回null
        public @Nullable
        Converter<ResponseBody, ?> responseBodyConverter(Type type,
            Annotation[] annotations, Retrofit retrofit) {
            return null;
        }
        // 返回一个能将type类型转换为RequestBody类型的Converter，如果不能转换那么返回null
        // 只在拥有@Body或者@Part或者@PartMap时才会调用
        public @Nullable
        Converter<?, RequestBody> requestBodyConverter(Type type,
            Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
            return null;
        }
        // 返回一个能将制定类型转换为字符串的Converter，只在拥有以下注解时才会调用
        // @Field、@FieldMap、@Header、@HeaderMap、@Path、@Query、@QueryMap
        Converter<?, String> stringConverter(Type type, Annotation[] annotations,
                                             Retrofit retrofit) {
            return null;
        }
        // 这两个方法与CallAdapterFactory一致
        protected static Type getParameterUpperBound(int index, ParameterizedType type) {
            return Utils.getParameterUpperBound(index, type);
        }
        protected static Class<?> getRawType(Type type) {
            return Utils.getRawType(type);
        }
    }
}
```

Converter.Factory 提供了 responseBodyConverter、requestBodyConverter、stringConverter 三个重要方法，分别用于将 ResponseBody 转换为指定类型、将指定类型转换为 RequestBody、将指定类型转换为 String。三个方法都返回值都是一个 Converter 实例其只有一个方法 convert 方法内部做转换操作，下面为了加深理解来看看 GsonConverterFactory 的源码。

```java
public final class GsonConverterFactory extends Converter.Factory {
    public static GsonConverterFactory create() {
        return create(new Gson());
    }
    public static GsonConverterFactory create(Gson gson) {
        if (gson == null) throw new NullPointerException("gson == null");
        return new GsonConverterFactory(gson);
    }
    private final Gson gson;
    private GsonConverterFactory(Gson gson) {
        this.gson = gson;
    }
    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
                                                            Retrofit retrofit) {
        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
        return new GsonResponseBodyConverter<>(gson, adapter);
    }
    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type,
                                                          Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
        return new GsonRequestBodyConverter<>(gson, adapter);
    }
}

final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
    private final Gson gson;
    private final TypeAdapter<T> adapter;

    GsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter) {
        this.gson = gson;
        this.adapter = adapter;
    }
    @Override
    public T convert(ResponseBody value) throws IOException {
        // 获取输入流转换成JsonReader实例，然后调用read转换为指定类型的实例
        JsonReader jsonReader = gson.newJsonReader(value.charStream());
        try {
            T result = adapter.read(jsonReader);
            if (jsonReader.peek() != JsonToken.END_DOCUMENT) {
                throw new JsonIOException("JSON document was not fully consumed.");
            }
            return result;
        } finally {
            value.close();
        }
    }
}
final class GsonRequestBodyConverter<T> implements Converter<T, RequestBody> {
    private static final MediaType MEDIA_TYPE = MediaType.parse("application/json; charset=UTF-8");
    private static final Charset UTF_8 = Charset.forName("UTF-8");

    private final Gson gson;
    private final TypeAdapter<T> adapter;

    GsonRequestBodyConverter(Gson gson, TypeAdapter<T> adapter) {
        this.gson = gson;
        this.adapter = adapter;
    }

    @Override
    public RequestBody convert(T value) throws IOException {
        // 获取输出流构建成JsonWrite实例，然后调用write方法写入buffer中
        Buffer buffer = new Buffer();
        Writer writer = new OutputStreamWriter(buffer.outputStream(), UTF_8);
        JsonWriter jsonWriter = gson.newJsonWriter(writer);
        adapter.write(jsonWriter, value);
        jsonWriter.close();
        return RequestBody.create(MEDIA_TYPE, buffer.readByteString());
    }
}
```

## 总结

Retrofit 的工作流程主要包括解析注解、动态代理、Request 的生成、Response 的解析四个大步骤，内部还是使用了 OkHttp 进行网络请求，其中前两步顺序可能会相反这取决于 validateEagerly 是否为 true，Request 的生成和 Response 的解析依赖于 CallAdapterFactory 和 ConverterFactory。