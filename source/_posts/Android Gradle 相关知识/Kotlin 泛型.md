---
title: Kotlin 泛型
date: 2018-12-02 13:08:23
type: "Kotlin"
---

## 前言

Kotlin 也拥有泛型的概念，和 Java 的有些相似，但是又不尽相同，本文主要记录 Java 和 kotlin 泛型的相同及差异点。<!-- more -->

## 什么是泛型？

泛型就是参数化类型，也就是说所操作的数据类型被指定为一个参数（type parameter）这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口、泛型方法。

## 为什么需要泛型？

在没有泛型之前，从集合中读取到的每一个对象都必须进行显示的强转，由于集合中可以存储所有对象，所有可能会出现强转错误，对于强转错误的情况，编译器不会给予提示，但是运行时会抛出异常。

泛型拥有以下优点：

1. 类型安全
2. 消除显示强转

举个例子：

```java
List<String> urls = new ArrayList<>();
```

其中 **List** 表示这是一个泛型类型为 **String** 的 List。urls 中只能添加 **String** 类型的对象(也可以添加子类，但是由于 String 类是 final 类，所以不存在子类)，对于其它类型编译器会直接报错，避免运行时出错，并且从 urls 里面取出对象时直接就是 String 类型不需要显示的强转。

## Java中的泛型

### 泛型类

泛型类型用于类的定义中，被称为泛型类。通过泛型可以完成对一组类的操作对外开放相同的接口。最典型的就是各种容器类，如：List、Set、Map。泛型类内部非静态成员可以直接使用泛型类型。

```java
// T 可以为任意标识符比如 K(key)、V(value)、E(element)
public class Generic<T>{ 
    private T key;
    public T getKey(){
        return key;
    }
    public void setKey(T key){
        this.key = key;
    }
}
public class Test {
    public static void main(String[] args) {
        Generic<String> generic = new Generic<>();
        generic.setKey("Hello, World!");
//        generic.setKey(1);  compile error 只能存String
        String key = generic.getKey();
    }
}
```

### 泛型接口

泛型接口与泛型类的定义及使用基本相同。泛型接口常被用在各种类的生产器中。

```java
public interface Generator<T> {
    public T next();
}
class RandomNumberGenerator implements Generator<Integer> {
    private Random random = new Random();
    @Override
    public Integer next() {
        return random.nextInt();
    }
}
```

### 泛型方法

非静态方法可以使用类声明的泛型类型。

```java
public <T> T genericMethod(Class<T> tClass){}
```

### 泛型的协变和逆变

先来看数组和泛型列表的区别。

```java
public class Test {
    public static void main(String[] args) {
        Fruit[] fruitArray = new Apple[10];
        // List<Fruit> fruitList = new ArrayList<Apple>(); compile error
        // fruitArray[0] = new Fruit(); throw exception
    }
}
class Fruit {}
class Apple extends Fruit {}
```

上例中可以看出数组是协变的，但是其不安全，**fruitArray** 里可以放入任何 **Fruit** 及其子类对象编译器不会报错，但是运行时可能会出错，而泛型是不变的，把 `ArrayList` 赋值给  `List` 会直接编译出错。

不过泛型借助于通配符也是可以实现协变和逆变。

#### 协变

先来看看协变。

```java
List<? extends Fruit> fruitList = new ArrayList<Apple>();
// fruitList.add(new Apple()); compile error
// fruitList.add(new Fruit()); compile error
// fruitList.add(new Object()); compile error
Fruit fruit = fruitList.get(0); // 编译通过
```

现在 fruitList 的类型是表示了该泛型上界是 Fruit，可以把任何泛型类型为 Fruit 及其子类的 `ArrayList` 赋值给它，在 fruitList 里面存的可能是 Fruit 或者其子类，由于编译器无法确定具体是哪个子类，因此拒绝往 fruitsList 里面放入任何对象。

相反由于知道了 fruitList 里面存的一定是 Fruit 或其子类所以调用 get 方法获取到的一定是 Fruit 实例。

```java
// List.java
boolean addAll(Collection<? extends E> var1); // 方法一
boolean addAll(Collection<E> var1); // 方法二
```

看看上述两个方法声明有什么不同点，为什么List要使用第一种方式，其有什么优势？假设 E 为 Number 类型。

- 优势一

由于方法一内部无法往 `val1` 中添加任何对象，一定程度杜绝了方法影响参数内容。

- 优势二

方法二只能接受参数 Collection\<Number>，而不能接受 Collection\<Integer> 等泛型类型是 Number 子类的 Collection。

##### 逆变

接着来看看逆变

```java
// List<? super Fruit> fruitList = new ArrayList<Apple>(); compile error
List<? super Fruit> fruitList1 = new ArrayList<Fruit>();
List<? super Fruit> fruitList2 = new ArrayList<Object>();
Object fruit = fruitList1.get(0);
```

现在 fruitList 的类型是 <? super Fruit>，表示了该泛型下界是 Fruit，可以把任何 ArrayList<Fruit及其父类> 赋值给它，在 fruitList 里面存的可能是 Fruit 或者其父类，所以可以把 Fruit 及其子类添加到 fruitList 中(多态)。

相反由于编译器无法知道 fruitList 确切存的是 Fruit 的哪个父类，因此 get 方法获取到的只能是 Object 对象。

```java
// Collections.java
public static <T> boolean addAll(Collection<? super T> c, T... elements) {} // 方法一
public static <T> boolean addAll(Collection<T> c, T... elements) {} // 方法二
```

看看上述两个方法声明有什么不同点，为什么 Collections 要使用第一种方式，其有什么优势？假设 E 为 Number类型。

优势为方法二只能传递 Collection\<Number>，而不能传递 Collection\<Object> 等泛型类型是 Number 父类的 Collection。

#### 协变与逆变总结

什么时候用 extends？，什么时候用 super？，\<\<Effecitive Java>> 第5章总结为 PECS（producer-extends, consumer-super）

了解了 java 中的泛型后现在回到正题上探究下 kotlin 的泛型。

## Kotlin中的泛型

### Array

上文已经讲到了 Java 的数组支持协变，然而 Kotlin 中的数组是不支持协变的，因为 Kotlin 中数组使用 Array 类，而该类泛型定义和集合类没什么区别：

```java
fun main() {
    val numbers: Array<Number> = Array<Int>(10) {} // compile error
}
```

### in 和 out 关键字

与 Java 中的泛型一样，Kotlin 中的泛型也是不可变的。

- Kotlin使用关键字 `out` 来支持协变，等价于Java中的上界通配符 `? extends`。
- Kotlin使用关键字 `in` 来支持逆变，等价于Java中的下界通配符 `? super`。

```kotlin
fun <T> addAll(var1: Collection<out T>)
boolean addAll(Collection<? extends E> var1); 

fun <T> addAll(var1: MutableCollection<in T>, vararg elements: T) {}
public static <T> boolean addAll(Collection<? super T> c, T... elements) {}
```

上述两组各自完全等价，唯一区别是 Kotlin 中 Collection 被设计成只读的，所以逆变需要使用 MutableCollection。

换了个写法，但作用是完全一样的。out 表示，我这个变量或者参数只用来输出，不用来输入，你只能读我不能写我；in 就反过来，表示它只用来输入，不用来输出，你只能写我不能读我。

```kotlin
class Producer<T> {
    fun produce(): T {...}
}
class Consumer<T> {
    fun consume(t: T) {}
}
fun main() {
//    val producer: Producer<Number> = Producer<Int>() compile error
    val producer: Producer<out Number> = Producer<Int>()
//    val consumer: Consumer<Number> = Consumer<Any>() compile error
    val consumer: Consumer<in Number> = Consumer<Any>()
}
```

此外 out 和 in 可以在泛型类声明类时使用，这点 Java 也是不支持的。

```java
public class JavaOut<? extends T> { } // compile error
public class JavaIn<? super T> { } // compile error
class KotlinIn<in T> {} // right
class KotlinOut<out T> {} // right
```

那么为什么 Kotlin 要新增这种语法呢？有什么优势？来看看下面的代码。

```kotlin
fun main() {
    val kotlinOut: KotlinOut<Number> = KotlinOut<Int>()
    val kotlinIn: KotlinIn<Number> = KotlinIn<Any>()
}
```

这段代码可以正确的编译通过，说明了如果在声明泛型类时使用了 out 那么该类就自带协变，使用了 in 那么该类就自带逆变。但是以下代码是错误的。

```kotlin
fun main() {
    // val kotlinOut: KotlinOut<in Number> = KotlinOut<Any>() compile error
    // val kotlinIn: KotlinIn<out Number> = KotlinIn<Int>() compile error
}
```

如果在声明泛型类时使用了 out 那么该类泛型就无法再被声明成 in，反之亦然。

### 通配符

Java使用 `?` 做为通配符，Kotlin中使用 `*` 做为通配符。

### where 关键字

Java 中声明类、接口、方法的时候，可以使用 extends 来设置边界，将泛型类型参数限制为某个类型的子集：

```java
class Java<T extends Number> { }
```

上述代码表示 T 的类型必须是 Number 或者其子类，注意：这里并没有用到?，要与前面说的分开。边界也可以设置多个。

```java
class Java<T extends OnClickListener & OnItemClickListener> { }

interface OnClickListener {}
interface OnItemClickListener {}
```

Kotlin中只是把设置单个边界从 `extends` 换成了 `:` ，设置多个边界使用 `where` 关键字。

```kotlin
class Kotlin<T> where T : OnClickListener, T : OnItemClickListener {}
```

### reified 关键字

先看看一个需求，比如我想从 Model 通过 Handler 发来的数据中取出 String 对象，但是由于发送过来的对象(Result#obj) 是一个 Objec t对象，所以就不得不进行强转，而强转有可能会失败，导致抛出异常，进而导致 App崩溃，因此就想写一个安全的强转方法，如下：

```kotlin
public static <T> T safelyCast(Object object) {
    if (object instanceof T) { // error
        return (T) object;
    } 
    return null;
}
```

但是由于 Java 中的泛型存在类型擦除的情况，任何在运行时需要知道泛型确切类型信息的操作都没法用了，因此 `object instanceof T` 不能通过编译。kotlin 中同样也不行：

```kotlin
fun <T> Any.safelyCast(): T? {
    if (this is T) { // error
        return this // 如果is T成立，kotlin会自动强转成T
    }
    return null
}
```

难道就没有办法可以做到吗？Java、Kotlin都是可以做到的，Java中可以通过传递一个Class对象进来如下所示：

```kotlin
public static <T> T safelyCast(Object object, Class<T> clazz) {
    if (clazz.isInstance(object)) {
        return (T) object;
    }
    return null;
}
```

在 Kotlin 中也可以这么解决，不过还有一种更优雅的实现，借助 reified 和 inline。

```kotlin
inline fun <reified T> Any.safelyCast(): T? {
    if (this is T) { // right
        return this
    }
    return null
}
```

下面是实际代码使用保证了绝无异常抛出：

```kotlin
override fun handleMessage(msg: Message): Boolean {
    msg.obj.safelyCast<Result>()?.obj?.salelyCast<String>()?.let {
        onGetTokenSuccess(it)
    }
    return true
}
private fun onGetTokenSuccess(token: String) { /* dosomething */ }
```

再举一个 `startActivity` 的例子

```kotlin
inline fun <reified T: Activity> Context.startActivity() {
    if (this is Activity) {
        startActivity(Intent(this, T::class.java))
    } else {
        startActivity(Intent(this, T::class.java).apply { 
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        })
    }
}
```

有了这个方法后启动Activity就可以简单的写为：

```kotin
fun startMainActivity(ctx: Context) {
    ctx.startActivity<MainActivity>()
}
```