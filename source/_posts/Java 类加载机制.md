---
title: Java 类加载机制
date: 2018-11-13 11:12:36
type: "JavaScript"
---



## 前言

类加载是 Java 程序运行的第一步，研究类的加载有助于了解 JVM 执行过程，同时对于 Android 热更新、插件化的理解也有很大帮助。<!-- more -->

## Java 类加载机制

Java 类加载机制主要分为 加载、验证、准备、解析、初始化、使用、卸载 七个步骤，如下图所示：

![类加载](类加载机制.png)

### 加载

* 查找字节流，据此生成一个 java.lang.Class 对象。注意：字节流不一定从 Class 文件中获取，可以从 jar 包中获取，也可以在运行时生成如动态代理。加载的信息会存储在 JVM 的方法区。

* 对于数组其没有对应的字节流，而是由 JVM 直接生成的，对于其它类 JVM 需要借助 ClassLoader 来查找字节流。

* 类加载器分为两种，启动类加载器和继承于 ClassLoader 的类加载器，前者是由 C++ 实现的，没有对应的 Java 对象。

* JVM 采用双亲委托模型进行类加载，当需要加载一个类时会首先检查该类是否已经自己被加载过，如果没被加载那么委托父类进行加载，一直到启动类加载器，如果启动类加载器检查该类还是没被自己加载过，那么在启动类加载器的指定目录下进行加载，还是找不到就再让子类加载器在其目录下加载，如果所有都加载不到则抛出 ClassNotFoundException。

* 双亲委托的好处是避免类的重复加载，当父 ClassLoader 已经加载了该类，子 ClassLoader 就没必要加载了，其次考虑安全因素，JDK 中的类不会被随意的替换(自定义 ClassLoader 可以替换)。

    下图为 Java 中的 ClassLoader 层次关系以及其负责加载的目录。

<img src="类加载器.png" style="zoom:50%">

* 有些情况需要破坏双亲委托，比如一个类使用 BootStrapClassLoader 加载，但是该类又需要加载不在该 ClassLoader 加载目录下的类，直接使用自定义 ClassLoader ，重写 loadClass 。

### 链接

* 验证，确保被加载类能够满足 Java 虚拟机的约束条件。

* 准备，为静态字段分配内存空间，以及初始化为 0 (对象为 null )  。
* 解析，将符号引用转换为直接引用。

### 初始化

* 执行静态代码块，初始化静态字段 (实际赋值)。
* 加载类并不一定会初始化，其会在以下几种情况下触发：
    1. 虚拟机启动时，初始化主类。
    2. 当使用 new 关键字，创建实例时。
    3. 当访问静态字段时。
    4. 当调用静态方法时。
    5. 子类初始化时。

## Android 类加载机制

在 Android 中所有的字节码都会被打入一个或者多个 dex 文件中，然后在安装时经过 DexOpt 进行处理，生成  odex 文件以加快运行速度 ，对于 Dalvik 虚拟机（JIT），每次运行时都会去加载 odex 文件转换为机器码进行执行。对于 Art 虚拟机（AOT），安装时会直接再将 odex 转换成机器码存储，运行时直接执行机器码。

基于上述原因，Android 中的 Davlik / ART 加载的是 dex 文件，主要有 BootClassLoader（Zygote 进程预加载类）、PathClassLoader（加载系统类和应用类）、DexClassLoader（加载 jar、apk、dex 文件），其中与 Java 中的 BootStrapClassLoader 不同， BootClassLoader 是在 Java 层实现的。注：PathClassLoader 与 DexClassLoader 的不同是前者在 Dalvik 虚拟机上不能加载未安装 apk 的 dex。

#### Android 热修复原理

QQ 空间的方案，就是将出错的类，单独打一个 dex 包，插入到 dexElements 数组的最前面，这样虚拟机就不会去加载那个错误的类了。但是有个问题，在将 dex 转换为 odex 的过程中如果某个调用的类都在同一个 dex 文件中，那么就会打上 ISPREVERIFIED 标志，然后写入 odex 文件，解决这个问题的方法是让应用的所有类都引用一个外 dex 的类，这个采用的字节码插入。缺点是不支持即时生效，必须通过重启 APP 。

Android 使用 DexClassLoader 加载 dex 文件，其源码如下：

```java
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), librarySearchPath, parent);
    }
}
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, optimizedDirectory);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }

    public void addDexPath(String dexPath) {
        pathList.addDexPath(dexPath, null /*optimizedDirectory*/);
    }
}
```

可以看到在 BaseDexClassLoader 中创建了 DexPathList 实例，然后 findClass 全部交给该实例去完成，看看 DexPathList.find 里面干了些什么。

```kotlin
public Class findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
}
```

迭代 dexElements，前一个 dexElement 找到目标类了那么就直接返回了，那么如果在应用启动时通过反射将修复后的 dexElement 通过反射插入到 dexElements 的首部，那么有错的类就不会被加载了。

#### Android 插件化原理

首先新建 DexClassLoader 对象，加载其它 APK 文件，获取到 dexElements 数组，然后获取到宿主 ClassLoader 内部的 dexElements 数组，将两者进行合并放入宿主 ClassLoader 的 dexElements 中。

Activity 加载需要先在 manifest 占几个坑，启动 Activity 会调用 Instrumentation.execStartActivity，因此可以对其 Hook ，将启动的 Activity 替换成占坑的 Activity，然后在调用 Instrumentation.newActivity 时创建原来想要启动的 Activity 实例即可。至于 Hook 方法就是通过新建一个 ActivityThread 类新建 currentActivityThread 方法骗过编译就行，实际运行还是会使用系统类的，或者直接反射从 ContextImpl 对象获取 mMainThread 对象。获取到 ActivityThread 实例以后替换 Instrumentation 对象。至于资源文件的加载，只需要反射修改调用 Resource.addAssetPath 即可。