---
title: Effective Kotlin 读后总结
date: 2020-01-16 12:34:53
type: "Android"
---



## 前言

以前阅读过 Effective Java 感觉很不错，最近看到国外出了本 Effective Kotlin 抱着好奇心买了一本电子书，本文主要记录下，阅读过程中值得注意的地方。参考网址 [Medium](https://blog.kotlin-academy.com/@marcinmoskala) 。<!--more-->

## Part 1: Good code

### 第一章：Safety

#### 1. 限制可变性

Kotlin 对安全所做的支持包括如下几个方面：

##### 仅读属性 val
 1. 虽然 val 默认不可变，其可以引用一个可变的对象。
 2. 当其拥有 getter 方法或者属性代理时，其就是可变的了(目的是让API变的灵活，后续可以更改)。
 3. 可以通过接口继承将父接口的 val 转变成 var，注意是接口继承，而不是实现。
 4. 不可变 val 编译器会支持智能强转，而可变 val 以及 var 则不支持。


```kotlin
val name: String? = "Hefuwei"
val fullName: String?
    get() = "Hefuwei"
    
fun main() {
    if (name != null) {
        println(name.length)
    }
    if (fullName != null) {
        // compile error: Smart cast to 'String' is impossible, because 'fullName' is a property that has open or custom getter
        println(fullName.length)
    }
}
```

注意：尽管 val 不是一定不可变的，当其拥有 getter 或者代理时就是可变的，通常能使用 val 尽量使用，不能用才考虑使用 var，因为默认 val 不存在线程同步问题。通过反编译成 Java 如果 val 没有 getter 方法那么就是 final 的，如果有那么就不是 final 的。

##### 将可变和只可读的容器分开

Kotlin 中对容器的层次关系进行了重新定义：

![image](容器.png)

这张图左边的蓝色的表示的是仅读的容器接口，右边的是可变的容器接口，每个可变的容器接口都继承了对应的不可变容器接口，以及继承上级可变容器接口。一共分为三层接口从上到下依次为 Iterable、Collection、List(Set) 等，仅读接口不提供任何修改容器内容的方法，比如 add、clear 等。

但是并不是 List 类型的对象一定是不可变的，比如 Interable.map 返回的是 List 类型对象，在 JVM 环境中实际对象是 ArrayList，这么做的原因是考虑到不同平台返回的实现类不同。但是即使我们知道返回的是 ArrayList，但是也不应该向下转型成 ArrayList，因为返回类型在其它平台或者新版本中可能会发生变化。既然方法返回了不可变类型就应该使用该不可变类型，盲目的向下转型可能导致错误。比如以下这个例子：

```kotlin
val list = listOf(1, 2)
fun main() {
    if (list is MutableList) {
        list.add(3)
    }
}
```

在 JVM 环境中 listOf 返回的是 Arrays.asList 的结果，Java 中的 List 接口被转换成了 Kotlin 中的 MutableList 接口，但是其不支持 add 等操作，因此抛出不支持异常，所以记住 listOf 返回的是 List 类型，**切记在 Kotlin 中不要将只读容器向下转型为可变容器**。

如果外界想要使用可变的 List，那么可以使用 toMutableList 将其转换为可变的 List。

```kotlin
val list = listOf(1, 2)
fun main() {
    val mutableList = list.toMutableList()
    mutableList.add(3)
}
```

##### data class 的 copy 方法

首先说说不可变对象，其优点有很多方面：

2. 容易共享给其它模块，因为它们之间不会有冲突。
3. 可以进行缓存，因为它们不会发生变化，如 Boolean.TRUE，Boolean.FALSE。
4. 用于可变对象中，不需要对其进行保护性拷贝。
5. 不可变的对象是构造其他对象的理想材料。
6. 可以加入到 Set 中或者作为 Map 中的 key (可变对象不要这么干)，因为如果可变对象加入到 Set 中去，可能内部属性发送变化，Map 中就无法找到该 key 对应的 value。

不可变对象最大的问题是有时候对象内部的数据需要被改变，解决方法是写一个方法产生一个修改数据后的对象。比如 Int 是不可变的，但是其提供了诸如 plus、minus 等方法用于产生一个新的 Int 对象。再比如 Iterable 是只读的，但是标准库中提供了 map、filter 等方法来生成一个新的对象。假设我们需要写一个不可变的 User 类，其支持更改姓名，那么可以这么写：

 ```kotlin
class User(val surname: String, val name: String) {
    fun withSurname(newSurname: String) = User(newSurname, name)
    ...
}
 ```

但是这种方法存在问题，一旦需要变化的属性很多，那么就要写很多这种方法及其不方便，这时候就可以使用 data class copy 方法通过可选参数支持所有参数更改。

```kotlin
data class User(val surname: String, val name: String)
fun main() {
    val me = User("何", "富威")
    println(me.copy(surname = "王"))
}
// outputs: User(surname=王, name=富威)
```

这是一种普遍的将数据类变成不可变的解决方案，确切的说这种方法的效率低于可变对象，但是它拥有上述一系列不可变对象的优点，应该首选这种方式。

##### 不同种类的可变点

假设需要一个可变的 List，如下两种方式都可以做到，那么应该选择哪种呢？

```kotlin
val list1: MutableList<Int> = mutableListOf(1, 2)
var list2: List<Int> = listOf(2, 1)
```

这两个属性都可以发生变化，使用不同的方式：

```kotlin
fun main() {
    list1.add(3)
    list2 = list2 + 3
}
```

注意：+ 号可用原因是 Collection 重载了操作符 plus，方法内部代码如下：

```kotlin
public operator fun <T> Collection<T>.plus(element: T): List<T> {
    val result = ArrayList<T>(size + 1)
    result.addAll(this)
    result.add(element)
    return result
}
```

并且上述两者还都可以使用 += 进行添加元素：

```kotlin
fun main() {
    list1 += 3 // 转变成 list1.plusAssign(3)
    list2 += 3 // 转变成 list2 = list2.plus(3)
}
```

上述两行代码都是对的，但是两者本质上是不同的，由于 list1 是 MutableList，其重载了操作符 plusAssign，内部就是简简单单往容器中加入一个元素。对于 list2 其是List，只重载了操作符 plus，然后将 plus 调用后的结果再赋值给了 list2。

第二种方式可以使用属性代理追踪属性的改变，便于调试，并且其还可以限制只能在类内部进行改变(通过私有 setter)：

```kotlin
var names by Delegates.observable(listOf<String>()) {
    _, old, new ->
    println("Names changed from $old to $new")
}

fun main() {
    names += "Hefuwei"
    names += "Wangchunlei"
}
/* outputs: 
Names changed from [] to [Hefuwei]
Names changed from [Hefuwei] to [Hefuwei, Wangchunlei] */
```

简而言之，使用可变集合是一个稍微快一点的选择，但是使用可变属性可以使我们对对象的更改方式有更多的控制。

注意：不要既是可变属性又是可变容器，这样会有两个可变点，可变点需要越少越好，同时其不再支持 += 语法，**不要写以下这种代码：**

```kotlin
var list3 = mutableListOf(1, 2)
fun main() {
    // error 编译器告知 +、+= 两个扩展方法都支持，没法区分。
    list3 += 3
}
```

##### 不要暴露可变点

设计类时如果需要将内部状态暴露给外部使用，要留个心眼，因为外界可能会误更改状态从而导致错误如：

```kotlin
class UserRepository {
    private val storedUsers: MutableMap<Int, String> = mutableMapOf()
    fun loadAll(): MutableMap<Int, String> = storedUsers
}
```

外界可以简简单单通过 loadAll 方法获取到私有状态，并改变。要解决这个问题，可以考虑使用**保护性拷贝**，对于一个**可变的 **data 类对象(不可变的 data 类对象直接返回没问题)，那么直接使用 copy 方法进行保护性拷贝，对于容器类，可以通过将返回类型转变成一个不可变类型解决。

```kotlin
class UserRepository {
    private val storedUsers: MutableMap<Int, String> = mutableMapOf()
    fun loadAll(): Map<Int, String> = storedUsers
}
```

##### 总结

主要有以下几点需要注意：

1. var、val 优先考虑 val。
2. 优先考虑不可变属性。
3. 优先考虑不可变对象。
4. 对于 data 类，如果要改变属性，考虑变成一个不可变的 data 类，然后使用 copy 方法改变属性。
5. 当持有状态时，优先考虑使用不可变容器，因为其可以通过属性代理追踪状态改变。
6. 尽量减少可变点。
7. 不要暴露可变的对象。

但是也不是任何情况下都是要优先考虑不可变，在某些性能优化时可能应该考虑使用可变，书后面应该有讲，现在就做到上面 7 点就行。

#### 2. 最小化变量的作用域

变量的作用域越小，那么程序就更加容易追踪和管理。作用域越大就会有更多导致其变化的地方。

1. 使用局部变量代替属性。
2. 尽可能的收紧变量的作用域，比如一个变量只在循环代码中使用，那么就应该定义在代码块中。

```kotlin
fun main() {
    val users = listOf<User>()
    var user: User
    // 第一种
    for (i in users.indices) {
        user = users[i]
        println("“User at $i is $user")
    }
    // 第二种
    for (i in users.indices) {
        val user = users[i]
        println("“User at $i is $user")
    }
    // 第三种
    for ((i, user) in users.withIndex()) {
        println("“User at $i is $user")
    }
}
```

上述三种方式应该优先采用第三种，其代码行数更少，并且将 user 的作用域限定在了循环体内。

**无论变量是只读的或者可读可写的，都要在其定义的时候对其进行初始化**，如果不这么做，就会强制开发者去寻找变量在哪里定义了。

```kotlin
// 禁止
val user: User
if (hasValue) {
    user = getValue()
} else {
    user = User()
}
// 推荐
val user: User = if (hasValue) {
    getValue()
} else {
    User()
}
```

如果需要设置两个变量，解构语法可以派上用场：

```kotlin
// 禁止
fun updateWeather(degrees: Int) {
    val description: String
    val color: Int
    if (degrees < 5) {
        description = "cold"
        color = Color.BLUE
    } else if (degrees < 23) {
        description = "mild"
        color = Color.YELLOW
    } else {
        description = "hot"
        color = Color.RED
    }
}
// 禁止
fun updateWeather(degrees: Int) {
    val description: String
    val color: Int
    when {
        degrees < 5 -> {
            description = "cold"
            color = Color.BLUE
        }
        degrees < 23 -> {
            description = "mild"
            color = Color.YELLOW
        }
        else -> {
            description = "hot"
            color = Color.RED
        }
    }
}
// 推荐
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 5 -> "cold" to Color.BLUE
        degrees < 23 -> "mild" to Color.YELLOW
        else -> "hot" to Color.RED
    }
}
```

后面还提到了一个小算法，**给定 2..100 的数字列表，如何获取到该列表中所有素数**？
这里有个思路，素数定义是只能被1或者本身整除的数字，那么也就是说只要能被 2..x-1 整除的数字就不是素数，首先判断数字列表是否为空，不为空就获取到第一个数字，将其加入到素数列表中去，接着将数字列表中所有可以被第一个数字整除的数字过滤掉，然后看看数字列表中是否还有数字，有的话再进行循环。**由于每次保留的数字列表都是不能被 2..x-1 整除的数字，所以数字列表的第一位一定是素数。** 下面是代码实现：

```kotlin
fun main() {
    var numbers = (2..100).toList()
    val primes = mutableListOf<Int>()
    while (numbers.isNotEmpty()) {
        val prime = numbers.first()
        primes.add(prime)
        numbers = numbers.filter { it % prime != 0 }
    }
    println(primes)
}
/* outputs:
[2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]
*/
```

如果想要创建一个无止境的素数序列，需要怎么做：

```kotlin
fun main() {
    val primes: Sequence<Int> = sequence {
        var numbers = generateSequence(2) { it + 1 }
        while (true) {
            val prime = numbers.first()
            yield(prime)
            numbers = numbers.filter { it % prime != 0 }
        }
    }
    println(primes.take(10).toList())
}
/* outputs:
[2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
*/
```

这段代码的基本意思可以搞明白，也就是当调用 toList 时，会执行 Lambda 表达式直到调用一个 yield 方法，然后 suspend，发现 1 个不够，那么 resume 继续执行表达式，直到产生 10 个素数。至于原理由于跟协程有关，暂时先不看。下面对其稍微做下改动：

```kotlin
fun main() {
    val primes: Sequence<Int> = sequence {
        var numbers = generateSequence(2) { it + 1 }
        var prime: Int
        while (true) {
            prime = numbers.first()
            yield(prime)
            numbers = numbers.filter { it % prime != 0 }
        }
    }
    println(primes.take(10).toList())
}
/* outputs:
[2, 3, 2, 3, 2, 3, 2, 3, 2, 3]
*/
```

只是把 prime 放到了循环体外面，结果就出错了，这是为什么呢？因为 filter 方法是一个中间操作，并不会立即执行，到真正执行的时候 prime 变量的值，早就不是原先那个值了，这种错误排查起来相对来说还是比较麻烦的，所以要最小化变量的作用域。

##### 总结

出于多种原因，必须要最小化变量的作用域，写代码时一定要注意 Lambda 表达式会捕获变量，如果表达式会被延时执行，一定要想想到真正执行的时候，变量的值是否还是所期待的值。

#### 3. 尽可能的消除平台类型

虽然 Kotlin 支持空安全，使得空指针发生的几率很小，或者完全不出现，但是 Java、C 不支持空安全，如果 Kotlin 调用一个返回 String 类型的 Java 方法，那么在 Kotlin 中对应的类型是什么呢？

1. 如果返回值被注解了 @Nullable 那么会被转换成 String?。
2. 如果返回值被注解了 @NotNull 那么被转换成 String。
3. 如果没注解那么转换成 String!。

为什么不将没注解的也当做 String? 进行处理呢，这样也更安全？原因是有些方法就是不可能返回 null 但是其没注解@NotNull，如果将这些返回的返回值当做 String? 处理那么很多地方就要使用 !! 进行强转，非常麻烦，比如 Java 方法返回 List\<User>，Kotlin 中不仅要把列表当做可空的，还要把列表中的元素当做可空的。**这里书上还提到了一个方法 filterNotNull，其会将容器中所有为空的元素去除返回一个新列表。**

平台类型就是在类型后面加个 !，表示该对象是从其它语言中获取的，拥有未知的可空性。注意：**在 Kotlin 的代码中不能声明一个变量的类型为平台类型。**

平台类型可以转化为非空类型或者是可空类型，亦或是不转化，这取决于开发者自身。

```kotlin
val userRepo = UserRepo()
val user1 = userRepo.user
val user2: User = userRepo.user
val user3: User? = userRepo.user
```

上面代码中 UserRepo 是一个 Java 类，user1 类型为 User!，user2 被转化为 User，user3 被转化为 User?。因为其也可以转化为非空类型，所以就不再需要很多没必要的强转，但是如果 API 没有被注解成 @NotNull 并且文档描述中也没说明返回值一定不为空，那么这个操作是很危险的，因为就算现在不可能为空，下一个版本也许就可能为空了，因此最好的方式还是把所有**可能被 Kotlin 调用的方法都加上注解**，这样才能有效避免错误，当 Kotlin 成为 Android 第一官方语言后，部分 Android API 也进行了相应的注解，但是还有好大一部分没加。

以下为所有 Kotlin 可识别的注解：

* JetBrains (@Nullable and @NotNull from org.jetbrains.annotations)
* Android (@Nullable and @NonNull from androidx.annotation as well as from com.android.annotations and from the support library android.support.annotations)
* JSR-305 (@Nullable, @CheckForNull and @Nonnull from javax.annotation)
* JavaX (@Nullable, @CheckForNull, @Nonnull from javax.annotation)
* FindBugs (@Nullable, @CheckForNull, @PossiblyNull and @NonNull from edu.umd.cs.findbugs.annotation
* ReactiveX (@Nullable and @NonNull from io.reactivex.annotations)
* Eclipse (@Nullable and @NonNull from org.eclipse.jdt.annotation)
* Lombok (@NonNull from lombok)