---
title: Java 基础
date: 2017-11-15 13:12:36
type: "JavaScript"
---



## 前言

本文主要记录下 Java 的一些基础知识以及易错点，主要注意包装类型的缓存、字符串池、泛型。<!-- more -->

## 基本数据类型

1. byte/8
2. char/16
3. short/16
4. int/32
5. float/32
6. long/64
7. double/64
8. boolean/~

boolean 类型只有两个值：true、false 可以使用 1bit 来存储，编译器一般将其转换为 int，使用 1 表示 true，0 表示 false。

## 包装类型

每个基本类型都有对应的包装类型，基本类型与其包装类型之间的赋值使用自动装箱与拆箱完成。

```java
Integer integer = 5;  // 自动装箱，调用了 Integer.valueOf(5)
int i = integer;      // 自动拆箱，调用了 integer.intValue()
```

Integer 默认情况下会缓存 value 为 -128 ~ 127 的 Integer 对象，当使用 valueOf(i) 的时候如果 i 在该区间则每次调用都会读取缓存，否则每次都新建对象。

```java
Integer integer1 = Integer.valueof(127);
Integer integer2 = Integer.valueOf(127);
System.out.println(integer1 == integer2); // true
Integer integer3 = Integer.valueof(128);
Integer integer3 = Integer.valueOf(128);
System.out.println(integer3 == integer4); // false
```

valueOf 相关源码如下：

```java
static final int low = -128;
static final int high;
static final Integer cache[];

static {
    high = 127;
    cache = new Integer[(high - low) + 1];
    int j = low;
    for(int k = 0; k < cache.length; k++)
        cache[k] = new Integer(j++);
}

public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

其它包装类型对应的缓存为：

1. Boolean：true、false
2. Byte：-128 ~ 127
3. Short：-128 ~ 127
4. Char：0 ~ 128

## 字符串

string 类被声明成 final 类，因此其不可被继承（Integer 等包装类型也不能被继承）。

在 Java 8 及以下，String 内部使用 char 数组存储数据。

```java
private final char value[];
```

在 Java 9 及以上，String 类实现改用 byte 数组存储字符串，同时使用 coder 来标识使用了哪种编码。

```java
private final byte[] value;
private final byte coder;
```

value 数组是 final 的并且 String 内部也没有改变它的方法，因此可以保证 String 不可变。

不可变的好处包括：

1. 线程安全。
2. 可以缓存 hash 值，避免每次计算。
3. 安全，作为参数传递，不用怕改变。
4. 便于 String Pool 缓存字符串。

### String、StringBuilder、StringBuffer 之间的比较

String 不可变，StringBuilder 和 StringBuffer 可变。

String 、StringBuffer 线程安全，StringBuilder 线程不安全。

### String Pool 字符串池

常量池其实还是保存着字符串的引用，真正的对象还是在堆中保存的，class 文件中记载的内容包括 魔数、次主版本号、常量池元素数量、常量池元素，在 class 文件被加载时，JVM 寻找到 class 文件中记载的常量池元素，然后一个个检查运行时常量池中是否包含该元素，如果不包含那么在堆中创建一个字符串对象然后把引用保存到常量池中的常量表中。

在 Java 中创建字符串对象一共有两种方式：

1. 采用字面值的方式赋值。
2. 采用 new 关键字

```java
String str1 = "hfw"; // 伪代码 = @A0000001
String str2 = "hfw"; // 伪代码 = @A0000001
System.out.println(str1 == str2); // true
```

在这段代码运行前，首先会加载该类，这时候 JVM 就会读取字节码中记录的常量池元素 "hfw" ，遍历常量池中引用的所有字符串对象，发现全部都不与之相同，于是就在堆中新建了一个 "hfw" 对象，将其引用赋值给常量池的常量表中，然后将字节码中所有使用该字面量的地方都会被替换成该引用。由于 str1 、str2 都等于常量池中记载的 "hfw" 对象引用，因此 str1 == str2 为 true。

```java
String str3 = new String("hfw"); // 伪代码 = new String(@A0000001)
String str4 = new String("hfw"); // 伪代码 = new String(@A0000001)
System.out.println(str3 == str4); // false
```

使用 new 创建字符串对象时，在执行到方法第一行时 JVM 会在堆中再创建一个 "hfw" 对象，然后将该引用返回，**而不是返回常量池中的引用**。执行到方法第二行时 JVM 又会在堆中创建一个，再返回新创建的对象引用因此 str3 == str4 为 false 。

当调用 intern 方法内部会遍历常量池保存的字符串引用，通过调用 equals 判断是否有与当前对象相等的引用，如果有那么返回常量池中保存的对象引用，如果没有将该引用保存到常量池，然后返回该引用。

```java
String str5 = new StringBuilder().append("he").append("fu").toString();
String str6 = new StringBuilder().append("he").toString();
String str7 = new String("fu");
System.out.println(str5.intern() == str5); // 1 true
System.out.println(str6.intern() == str6); // 2 false
System.out.println(str7.intern() == str7); // 3 false
```

首先加载该类时会在堆中新建 "he"、"fu" 两个字符串对象，将其引用保存到常量池中，str5 执行后在堆中创建了一个新的字符串对象 "hefu" ，此时常量池中不含有该对象引用，当执行代码 1 时，发现常量池中没有与 "hefu" 相等对象引用，于是就是该对象引用保存到常量池中，然后返回该对象引用，因此代码 1 返回 true 。当执行代码 2 时，发现常量池已经有与 ''he " 相等的对象引用，因此直接返回在加载时创建 "he" 的引用而不是新在堆中创建的引用因此代码 2 返回 false，代码 3 同理。

最后需要注意一点，编译时会将纯字符串拼接后的字符串当做常量池元素。

```java
String s1 = "he" + "fu"; // 1
String s2 = new StringBuilder().append("he").append("fu").toString(); // 2
System.out.println(s2.intern() == s2); // false
```

代码 1 由于是全字面量拼接，因此编译后将 "hefu" 放入了字节码常量池元素中去了，于是代码 2 执行时发现常量池已经有与 "hefu" 相等的对象引用，因此 s2.intern == s2 为 false 。

### 三、运算

byte、short、char、int、float、double 之间的相互赋值。

switch 支持 byte(Byte)、short(Short)、char(Character)、int(Integer)、String、Enum，不支持 long。

## 关键字

final 

1. 数据，表示常量首次赋值后不能再修改。

2. 方法，表示方法不能被子类重写。
3. 类，表示类不能被继承。

static

1. 静态变量与方法，都与实例无关，属于类，可以直接通过类名调用。
2. 静态代码块，在类初始化时会运行一次。
3. 静态内部类，不含有外部类实例，其实就跟一个普通类一致。
4. 静态导包，可以导入静态方法，使用时就不需要类名了。
5. 静态初始化顺序，按照声明顺序进行执行。

## Object 通用方法

Object 类含有如下方法：

```java
public boolean equals(Object obj)
public native int hashCode()
protected native Object clone() throws CloneNotSupportedException
public String toString()
public final native Class<?> getClass()
protected void finalize() throws Throwable {}
public final native void notify()
public final native void notifyAll()
public final native void wait(long timeout) throws InterruptedException
public final void wait(long timeout, int nanos) throws InterruptedException
public final void wait() throws InterruptedException
```

1. equals 用于判断两个对象是否相等，并非一定要引用相同。
2. hashCode 在将一个自定义类对象放入散列表中时，注意重写该方法，否则即使两个对象 equals 返回 true，但是由于 hashCode 不同，因此还会被认为是两个不同对象。
3. clone 只有实现了 cloneable 接口才能调用该方法，不然会抛出异常，最好不使用该方法，改为使用拷贝构造器或者拷贝工厂方法。
4. toString 将该对象转换为一个字符串，默认是对象名@内存地址。
5. getClass 获取当前类对应的 Class 对象。
6. finalize 在对象被 GC 回收前可能会被调用，不靠谱别使用。
7. notify 唤醒一个处于 wait 状态（锁是当前对象）的线程。
8. notifyAll 缓存所有处于 wait 状态（锁是当前对象）的线程。
9. wait 使当前线程处于 wait 状态，直到被唤醒，只能在同步代码块中使用。

## 访问权限

public > protected > default > private 

注意：类不要有公有字段，应该改用 getter、setter 替代，这样就可以保留后续修改的能力，不过也有例外，如果是包级私有的或者私有的嵌套类，那么可以使用公有字段。

## 接口

Java 8 添加了默认方法，通过给接口添加默认接口，不需要修改实现类。此外接口中的字段都是 public static final 的，方法都是 public 的。

## 反射

反射可以做到一些正常情况下无法办到的事情，但是反射存在性能损耗。

## 异常

异常分为受检异常和非受检异常。

受检异常：

1. IOException
2. FileNotFoundException
3. SQLException

非受检异常：

1. NullPointException
2. ClassCastException
3. ArrayIndexsOutOfBoundsException
4. ArithmeticException

### 十、泛型

Java 的泛型是什么？使用泛型的好处是什么？

以容器为例，假设没有泛型，那么容器中可以存储任意的对象，每次取出对象的时候都需要强转，并且容易抛出 ClassCastException，因为编译器不会帮你检测该容器内是否放入了其它错误的对象。

Java 的泛型是如何工作的？什么是类型擦除？

编译器在编译时擦除了所有的类型相关的信息，所以在运行时不存在任何类型相关的信息，比如代码中的 List\<String> 在运行时就变成了一个 List 来表示，这么做的目的是为了兼容 Java 5 前的代码。

什么是泛型中的限定通配符和非限定通配符？

限定通配符有两种，<? extends T> 以及 <? super T>，前者指定了上界，后者指定了下界。

非限定通配符就是 <?>。

List<? extends T> 和 List<? super T> 之间有什么区别？

前者可以接收任何泛型是 T 及其子类的 List ，后者可以接收任何泛型时 T 及其父类的 List，当从内部取值时前者可以转化为 T 类型，后者只能转化为 Object 类型，当往里面添加数据时，前者只能添加 null ，后者所有 T 及其子类都可以添加。

### 十一、注解

Java 中提供了 4 个源注解，用于修饰其它的注解，分别是：

1. @Documented 注解是否包含在 JavaDoc 中。
2. @Retention 什么时候使用该注解。
3. @Target 注解用于什么地方。
4. @Inherited 是否允许子类继承该注解。

@Retension 取值为：

1. RetensionPolicy.SOURCE：在编译阶段丢弃，比如 @Override、@SuppressWarnings。
2. RetensionPolicy.CLASS：在类加载时丢弃，处理字节码文件有用。
3. RetensionPolicy.RUNTIME：始终不会丢弃，运行时可以使用反射获取。一般自定义时使用这种。

@Target 取值为：

1. ElementType.CONSTRUCTOR: 用于描述构造器。
2. ElementType.FIELD: 成员变量（包括enum实例）。
3. ElementType.LOCAL_VARIABLE: 用于描述局部变量。
4. ElementType.METHOD: 用于描述方法。
5. ElementType.PACKAGE: 用于描述包。
6. ElementType.PARAMETER: 用于描述参数。
7. ElementType.TYPE: 用于描述类、接口、枚举。