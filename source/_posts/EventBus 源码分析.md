---
title: EventBus 源码分析
date: 2018-12-15 9:12:46
type: "Android"
---

## 前言

EventBus 使用发布/订阅者模式，能够很好的进行模块之间的通信、解耦。以前一直停留在会用的层次，为了探究底层实现阅读了其源码实现，这也是继 Volley 以外我第二个阅读的框架源码。<!--more-->

## 概述

EventBus 仓库地址为：https://github.com/greenrobot/EventBus 。官方给出以下一张图来帮助理解。

![官方说明](官方说明.png)

Publisher 使用 post 方法发送 Event ，Subscribe 在 onEvent 方法中接收 Event。据官方介绍 EventBus 有如下优点：

* 简化组件间的交流。
    * 分离事件发送至和接收者。
    * 更好的处理 Activities、Fragments、Background threads。
    * 避免复杂且易出错的依赖关系和生命周期问题。
* 使你的代码更简单。
* 速度快。
* 库较小。
* 使用该库的 APP 多。
* 拥有一些高级功能比如指定分发线程，设置订阅者优先级等。

## 基本用法

这里以 EventBusSecondActivity 通知 EventBusFirstActivity 为例，代码如下：

```java
data class MessageEvent() // 1
class EventBusFirstActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_eventbus_first)
    }
    
    @Subscribe(threadMode = ThreadMode.MAIN, sticky = false)
    fun onReceiveEvent(msg: MessageEvent) { // 2
        Log.d("Receive message!")
    }

    override fun onStart() { // 3
        super.onStart()
        EventBus.getDefault().register(this)
    }
    
    override fun onStop() {
        super.onStop()
        EventBus.getDefault().unregister(this)
    }

    fun jump(view: View) {
        startActivity(Intent(this, EventBusSecondActivity::class.java))
    }
}
class EventBusSecondActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_eventbus_second)
    }

    fun send(view: View) {
        EventBus.getDefault().post(MessageEvent())
    }
}
```

1. 定义事件类： MessageEvent ，如果需要额外信息可以添加字段。
2. 准备订阅者：声明以及注解订阅方法 onReceiveEvent ，可选 threadMode、sticky 、priority 。
3. 注册订阅者：调用  ```EventBus.getDefault().register(this)``` 。
4. 发送事件：调用  ```EventBus.getDefault().post(MessageEvent())``` 。

经过以上四步，EventBusFirstActivity.onReceiveEvent 就会被执行。下面来分析下源码实现，首先第一步没什么好说的，从 Subscribe 注解开始说起。

## Subscribe 注解

代码如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    ThreadMode threadMode() default ThreadMode.POSTING;
    boolean sticky() default false;
    int priority() default 0;
}
```

一共有以下三个属性：

*  threadMode 表示注解的方法需要在哪个线程执行。可选值为（**当前线程为执行 post 方法的线程**）：
    1. POSTING，直接在当前线程调用。
    2. MAIN，如果当前线程是主线程那么在当前直接调用，如果不是那么切换线程到主线程调用，因此调用线程一定是主线程。
    3. MAIN_ORDERD，对比 MAIN ，其不判断当前是哪个线程，全部切换到主线程调用，因此调用线程一定是主线程。
    4. BACKGROUND，如果当前线程是主线程那么切换到子线程中调用，并且多个事件是串行处理的，如果不是那么直接在当前线程调用，因此调用线程一定是子线程。
    5. ASYNC，对比 BACKGROUND ，其不判断当前是哪个线程，全部切换到子线程中调用，并且多个事件是并行处理的，因此调用线程一定是子线程。
* sticky 表示是否接收在注册前已经发送的粘性事件。
* priority 表示订阅方法调用的优先级，只对于相同的 threadMode 生效。

## EventBus.register

代码如下：

```java
public EventBus() {
    this(DEFAULT_BUILDER);
}
public static EventBus getDefault() { // 1
    if (defaultInstance == null) {
        synchronized (EventBus.class) {
            if (defaultInstance == null) {
                defaultInstance = new EventBus();
            }
        }
    }
    return defaultInstance;
}
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass); // 2
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod); // 3
        }
    }
}
```

1. 双重校验锁单例模式，但是 EventBus 的构造器不是私有的，并且还可以通过 EventBusBuilder.build 创建实例，这里先只考虑使用 EventBus.getDefault 的情况。
2. 调用 SubscriberMethodFinder.findSubscriberMethods 查询类内所有注解了 @Subscribe 的方法。
3. 对每个查询到的方法执行 EventBus.subscribe 方法。

### SubscriberMethodFinder.findSubscriberMethods

代码如下：

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass); // 1
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    if (ignoreGeneratedIndex) { // 2
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    if (subscriberMethods.isEmpty()) { // 3
        throw new EventBusException("Subscriber " + subscriberClass
                                    + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        METHOD_CACHE.put(subscriberClass, subscriberMethods); // 4
        return subscriberMethods;
    }
}
```

1. 如果 ```METHOD_CACHE```（ConcurrentHashMap） 里面已经存在了，那么使用缓存。
2. 默认 ```ignoreGeneratedIndex``` （是否忽略 APT 生成的 Index ）为 false，因此会走到  ```findUsingInfo(subscriberClass)``` 。
3. 如果在类中没找到注解了 @Subscribe 的方法那么抛出异常。
4. 将找到的方法放入 ```METHOD_CACHE``` 进行缓存。

#### SubscriberMethodFinder.findUsingInfo

代码如下：

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState(); // 1
    findState.initForSubscriber(subscriberClass); // 2
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState); // 3
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState); // 4
        }
        findState.moveToSuperclass(); // 5
    }
    return getMethodsAndRelease(findState); // 6
}
```

1. 从缓存中取出或者创建一个实例，注：FindState 只要为了寻找出订阅者中哪些方法算合法订阅。
2. 将 ```findState``` 中的 ```subscribeClass```、```clazz``` 字段都置为 ```subscribeClass``` ，为寻找方法做准备。
3. 根据 ```findState``` 找出订阅信息，如果不使用插件，这里一开始会返回 null。
4. 调用 ```findUsingReflectionInSingleClass``` 找出 ```findState.clazz``` 类中所有符合的方法。
5. 调用 ```findState.moveToSuperclass``` 将 ```findState.class``` 变为其父类。
6. 调用 ```getMethodsAndRelease``` 

##### SubscriberMethodFinder.prepareFindState

代码如下：

```java
private FindState prepareFindState() {
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {
                FIND_STATE_POOL[i] = null;
                return state;
            }
        }
    }
    return new FindState();
}
```

如果 ```FIND_STATE_POOL``` 有 FindState 实例，那么取出一个，否则新建一个实例。

##### SubscriberMethodFinder.initForSubscriber

代码如下：

```java
void initForSubscriber(Class<?> subscriberClass) {
    this.subscriberClass = clazz = subscriberClass;
    skipSuperClasses = false;
    subscriberInfo = null;
}
```

将 ```subscribeClass```、```clazz``` 字段都置为 ```subscribeClass``` ，为寻找方法做准备，clazz 会不断改变，从当前类到其父类。

##### SubscriberMethodFinder.findUsingReflectionInSingleClass

代码如下：

```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        methods = findState.clazz.getDeclaredMethods(); // 1
    } catch (Throwable th) {
        try {
            methods = findState.clazz.getMethods();
        } catch (LinkageError error) {
            ...
            throw new EventBusException(msg, error);
        }
        findState.skipSuperClasses = true;
    }
    for (Method method : methods) {
        int modifiers = method.getModifiers();
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) { // 2
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) { // 2
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class); // 2
                if (subscribeAnnotation != null) {
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) { // 3
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                                                     subscribeAnnotation.priority(), subscribeAnnotation.sticky())); // 4
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) { // 5
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                                            "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) { // 7
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

1. ```getDeclaredMethod```  速度比 ```getMethod``` 快，前者获取的是当前类的所有方法，后者获取的是当前类及其父类的所有公有方法，如果使用了 ```getMethod```  那么就跳过从父类中寻找方法。
2. ```method``` 必须是满足，可见性为 publish、非静态、非抽象、参数一个、 @Subscribe 注解，否则跳过。 
3. 执行 **```findState.checkAdd```** 存储订阅方法，这个方法很难理解但是很重要，决定了该方法要不要。
4. 将符合条件的方法封装成 SubscriberMethod 实例，加入到 FindState 的 ```subscriberMethods``` 中。
5. 各种开启严格模式，抛出异常的情况。

###### FindState.checkAdd

代码如下：

```java
boolean checkAdd(Method method, Class<?> eventType) {
    Object existing = anyMethodByEventType.put(eventType, method);
    if (existing == null) {
        return true;
    } else {
        if (existing instanceof Method) {
            if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                throw new IllegalStateException();
            }
            anyMethodByEventType.put(eventType, this);
        }
        return checkAddWithMethodSignature(method, eventType);
    }
}
private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
    methodKeyBuilder.setLength(0);
    methodKeyBuilder.append(method.getName());
    methodKeyBuilder.append('>').append(eventType.getName());
    String methodKey = methodKeyBuilder.toString();
    Class<?> methodClass = method.getDeclaringClass();
    Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
    if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) { // 1
        return true;
    } else {
        subscriberClassByMethodKey.put(methodKey, methodClassOld);
        return false;
    }
}
```

由于方法相对比较难以理解，这里以类 A 存在两个接收 Message 类实例的方法 a1、a2 为例。

1. a1 方法进入，Message 事件还没存在于 ```anyMethodByEventType``` ，那么 ```existing``` 为 null ，方法直接返回 true。
2. a2 方法进入，Message 事件已经存在于 ```anyMethodByEventType```，那么 ```existing``` 不为空，执行 ```checkAddWithMethodSignature``` 将 a1 方法传入，a1 的 methodKey 不存在于 ```subscriberClassByMethodKey``` ，那么 ```methodClassOld``` 为空，直接返回 true ，然后直接把当前 **FindState** 实例放入 ```anyMethodByEventType``` ，原先的 a1 被替换成 FindState 实例，接着执行第二次 ```checkAddWithMethodSignature``` 将 a2 方法传入，a2 的 methodKey 不存在于 ```subscriberClassByMethodKey``` ，那么 ```methodClassOld``` 为空，直接返回 true ， 来看看现在内存情况，```anyMethodByEventType``` 存放 Message - FindState 直接的映射，```subscriberClassByMethodKey``` 存放 a1 - A、b1 - B 。
3. 经过上述两步，应该基本明白了，如果订阅类一个事件就对应一个方法，那么 ```anyMethodByEventType``` 就已经够用了，内部存放 Message - Method 映射，如果一个事件对应多个方法，那么 ```anyMethodByEventType``` 不够用，内部存放 Message - FindState 映射。
4. 再说说注释一处，这里其实考虑的也挺多，假设 A 类有方法 a（接收 Message1）、b（接收 Messages2）、a（接收 Message 2），第二个 a 在代码一处判断为 false || true ，会将同名不同参方法添加，再假设 SuperA 类有方法 a ，A 重写方法 a，这时候代码一处就变成了 false || false ，不将已被子类重写的父类方法缓存。

##### FindState.moveToSuperClass

代码如下：

```java
void moveToSuperclass() {
    if (skipSuperClasses) {
        clazz = null;
    } else {
        clazz = clazz.getSuperclass();
        String clazzName = clazz.getName();
        if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") ||
            clazzName.startsWith("android.") || clazzName.startsWith("androidx.")) {
            clazz = null;
        }
    }
}
```

如果不跳过查询父类，将 FindState 的 ```clazz``` 字段变成其父类的 Class 实例，如果类全名遇到 java. 、javax. 、android. 、androidx. 开头那么跳过。

##### SubscriberMethodFinder.getMethodsAndRelease

```java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    findState.recycle();
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (FIND_STATE_POOL[i] == null) {
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    return subscriberMethods;
}
```

将 FindState 对象重置并回收。思考：**是否有必要这么设计 FindState ？** 我认为如果就便于理解层面来看，可以使用 Map<Class, Set\<String>> 替代 FindState 内部的两个 Map，首先消息类型 Class 如果不一致，那么出现重名方法的可能性就只有重写父类方法了，可以简简单单通过 contains 进行判断，忽略掉重写的父类方法。

回到 EventBus.register ，看看 ```subscribe``` 方法。

### EventBus.subscribe

代码如下：

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod); // 1
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>(); // 2
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) { // 3
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                                        + eventType);
        }
    }
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) { // 4
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents); // 5
    }
    subscribedEvents.add(eventType);

    if (subscriberMethod.sticky) { // 6
        if (eventInheritance) {
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

1. 将订阅者对象与找到的方法封装成 Subscription 对象。
2. 维护数据结构 Map<Class, CopyOnWriteArrayList\<Subscription>> key 为事件的 Class 对象，value 为接收该事件的所有订阅。
3. Subscription 类重写了 equals 方法，如果订阅者对象一样，并且内部的订阅方法也一样那么抛出异常。
4. 按照优先级顺序插入到 CopyOnWriteArrayList 中。
5. 维护数据结构 Map<Object, List<Class<?>>> key 为订阅者对象，value 为该类所有事件的 Class 列表。
6. 处理粘性事件，稍后再解释。

注册流程已经完成，下面看下发送流程。

## EventBus.post

代码如下：

```java
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get(); // 1
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState); // 2
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

1. 获取当前线程的 PostThreadState 实例。
2. 执行 ```postSingleEvent``` 方法

### EventBus.postSingleEvent

代码如下：

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass); // 1
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz); // 2
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass); // 2
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
            eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

1. 默认情况下 ```eventInheritance``` 为 true ，该属性表示是否发送父事件，比如发送 Message 事件，那么同时也会发送 Parcelable 事件，因为 Message 实现了 Parcelable 接口。
2. 执行 ``` postSingleEventForEventType``` 进行事件的发送。

#### EventBus.postSingleEventForEventType

代码如下：

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        subscriptions = subscriptionsByEventType.get(eventClass); // 1
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted;
            try {
                postToSubscription(subscription, event, postingState.isMainThread); // 2
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

1. 获取原先注册时收集为接收该事件的 subscriptions 。
2. 执行 ```postToSubscription``` 进行事件的发送。

##### EventBus.postToSubscription

代码如下：

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

1. POSTING，直接在当前线程使用反射调用。
2. MAIN，如果当前是主线程，那么直接调用，否则执行 ```mainThreadPoster.enqueue``` 。
3. MAIN_ORDERED，由于默认情况 ```mainThreadPoster``` 不为空，于是执行 ```mainThreadPoster.enqueue``` 。
4. BACKGROUND，如果当前是主线程，那么执行 ```backgroundPoster.enqueue```，否则直接反射调用。
5. ASYNC，执行 ```asyncPoster.enqueue``` 。

###### HandlerPoster.enqueue

代码如下：

```java
public void enqueue(Subscription subscription, Object event) {
    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event); // 1
    synchronized (this) {
        queue.enqueue(pendingPost); // 1
        if (!handlerActive) {
            handlerActive = true;
            if (!sendMessage(obtainMessage())) { // 2
                throw new EventBusException("Could not send handler message");
            }
        }
    }
}
public void handleMessage(Message msg) {
    boolean rescheduled = false;
    try {
        long started = SystemClock.uptimeMillis();
        while (true) {
            PendingPost pendingPost = queue.poll(); // 3
            if (pendingPost == null) {
                synchronized (this) {
                    pendingPost = queue.poll(); // 3
                    if (pendingPost == null) {
                        handlerActive = false;
                        return;
                    }
                }
            }
            eventBus.invokeSubscriber(pendingPost); // 3
            long timeInMethod = SystemClock.uptimeMillis() - started;
            if (timeInMethod >= maxMillisInsideHandleMessage) { // 4
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
                rescheduled = true;
                return;
            }
        }
    } finally {
        handlerActive = rescheduled;
    }
}
void invokeSubscriber(PendingPost pendingPost) {
    Object event = pendingPost.event;
    Subscription subscription = pendingPost.subscription;
    PendingPost.releasePendingPost(pendingPost);
    if (subscription.active) {
        invokeSubscriber(subscription, event);
    }
}
```

1. 从 PendingPost 池中取出或者创建一个 PendingPost 实例，并将其放入队列中（单链表实现）。
2. 如果 ```handleMessage``` 的 when 循环没有执行，那么发送消息使其执行。
3. 从队列中取出消息，如果取不到则使用同步再取一次，可能情况是其它线程添加进去了当前线程不可见，使用同步强制去主存读一次，取到后回收 PendingPost 实例，然后反射执行。
4. 如果事件总执行时间超过 ```maxMillisInsideHandleMessage``` 那么重新发送事件，意图是防止卡顿，给 UI 刷新执行时间，不然一直卡在这循环了。

###### BackgroundPoster.enqueue

代码如下：

```java
public void enqueue(Subscription subscription, Object event) {
    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event); // 1
    synchronized (this) {
        queue.enqueue(pendingPost); // 1
        if (!executorRunning) { // 2
            executorRunning = true;
            eventBus.getExecutorService().execute(this);
        }
    }
}
@Override
public void run() {
    try {
        try {
            while (true) {
                PendingPost pendingPost = queue.poll(1000); // 3
                if (pendingPost == null) {
                    synchronized (this) {
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            executorRunning = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost); // 3
            }
        } catch (InterruptedException e) {
            eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
        }
    } finally {
        executorRunning = false;
    }
}
```

1. 从 PendingPost 池中取出或者创建一个 PendingPost 实例，并将其放入队列中（单链表实现）。
2. 如果当前 run 方法还没被执行，那么将其放入线程池中执行，默认线程池为 ```Executors.newCachedThreadPool``` 由于无线程上界几乎可以立即执行。
3. 从队列中取出消息，如果1秒内取不到则使用同步再取一次，可能情况是其它线程添加进去了当前线程不可见，使用同步强制去主存读一次，取到后回收 PendingPost 实例，然后反射执行，**事件是顺序执行的**，必须等到上个事件处理完毕后，下一个才能被处理。

###### AsyncPoster.enqueue

代码如下：

```java
public void enqueue(Subscription subscription, Object event) {
    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
    queue.enqueue(pendingPost);
    eventBus.getExecutorService().execute(this);
}

@Override
public void run() {
    PendingPost pendingPost = queue.poll();
    if(pendingPost == null) {
        throw new IllegalStateException("No pending post available");
    }
    eventBus.invokeSubscriber(pendingPost);
}
```

获取 PendingPost 对象，将其放入队列，然后将自身放入线程池执行 run 方法，在内部再取出 PendingPost 对象，反射执行方法调用，与 BackgroundPoster 不同的是，**事件是并发执行的**，只要有事件就立马执行，下一个不需要等上一个事件处理完毕。

至此，注册订阅以及发送事件流程已经完成了，下面来看看取消注册。

## EventBus.unregister

代码如下：

```java
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

首先从 ```typesBySubscriber``` 中取出订阅者类拥有的所有订阅类型，然后再根据订阅类型找到对应的 ```subscriptions``` ，并且遍历如果发现其订阅者是 ```subscriber``` 将其 ```activie``` 置为 false（防止订阅方法再被执行） 然后将其移除。就完成了注册和解注册，接着看看粘性事件。

## EventBus.postSticky

代码如下：

```java
public void postSticky(Object event) {
    synchronized (stickyEvents) {
        stickyEvents.put(event.getClass(), event);
    }
    post(event);
}
```

相对于非粘性事件，只是多了往 ```stickyEvents``` 里面添加事件类以及事件对象，想想也是，粘性事件相比非粘性只是在注册时可能会立即执行，那么回到 EventBus.subscribe 。

### EventBus.subscribe

代码如下：

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
	...
    if (subscriberMethod.sticky) { // 1
        if (eventInheritance) { // 2
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

1. 首先如果注解属性 sticky 为 true 才会进入。
2. 默认 ```eventInheritance``` 为 true 。
3. 从 ```stickyEvents``` 中取出已缓存的粘性事件对象，如果订阅的事件是粘性事件或者是其父类，那么执行 ```checkPostStickyEventToSubscription ``` 。
4. 如果  ```eventInheritance``` 为 false ，那么从 ```stickyEvents``` 中取出已缓存的粘性事件对象，如果订阅的事件是粘性事件，那么执行 ```checkPostStickyEventToSubscription ``` 。

#### EventBus.checkPostStickyEventToSubscription

代码如下：

```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
    if (stickyEvent != null) {
        postToSubscription(newSubscription, stickyEvent, isMainThread());
    }
}
```

又回到了 ```postToSubscription``` 根据 ThreadMode 在对应线程反射执行方法。

## 总结

EventBus 的核心是以下三个 Map：

1. Map<Object, List\<Class> 用于解除注册。
2. Map<Class, CopyOnWriteArrayList\<Subscription>> 用于分发事件。
3. Map<Class, Object> 用于粘性事件。

基于反射的 EventBus 会对性能有一点影响，后续看一看其提供的 APT 插件。