---
title: Thinking in Java 总结
date: 2018-02-21 14:22:34
type: "Java"
---

## 前言

老早以前就听说过这本书，一直没看，主要是以前认为这本书讲的比较深，事实上也确实比较深，最近花了近一个月阅读了一遍，理解到的不多，权当记录了。<!--more-->

## 关键点

* 当调用一个对象中的方法时，比如调用 Dog 类实例 dog.eat()，编译器会把 dog 引用当成 eat() 的第一个参数传入即 eat(this, otherParams)。

* 在构造器中可以使用 this 关键字去显式的调用构造器，并且只能位于构造器的第一行，其它地方都不能调用构造器。

* Java 为类做的准备工作包括以下三步：

    1. 加载，ClassLoader 在 ClassPath 下寻找到类的字节码文件后，根据字节码创建一个 Class对象。
    2. 链接， 验证类中的字节码，为静态域分配存储空间，并设置默认值，将类的二进制数据中的符号引用替换为直接引用。符号引用就是字节码中保存的 类、字段、方法、字符串引用，只是一些描述信息，因为编译时不可能知道实际的内存地址。直接引用指向对象、方法区，就在这时候会根据字节码的常量池创建 String 对象，并将引用保存到 StringPool 中。
    3. 初始化，如果该类有父类加载父类，然后初始化父类的静态域，然后回来初始化子类静态域。

    注意：如果只是获取一个编译器常量 static final 那么加载都不会触发，如果获取 .class 不会执行初始化步骤。

* 创建对象执行 1-5，调用静态字段、方法执行 1-2，注意图中的加载表示加载、链接两步。

    1. 当 Java 运行时需要某个类时(调用该类的静态域或者构造器（其实也属于静态方法），除去编译器常量，会触发改类的加载、链接、初始化，如果其有父类则在子类初始化开始后，触发父类的加载 、 链接 、初始化，以此类推。
    2. 从基类到子类初始化静态字段和静态代码块按照书写顺序。
    3. 在堆中分配一块空间存放对象，并把该空间都置于零，基本数据类型为0，引用为null。
    4. 调用构造方法第一行直到 Object。
    5. 基类的非静态字段进行初始化，接着进行子类的非静态字段进行初始化。

    ![代码执行流程](代码执行流程.png)

* 静态字段属于类，当类初始化就会初始化，非静态字段属于对象，当对象不存在时是不会进行初始化的。
* 垃圾回收机制，主要有以下几种：
    1. 引用计数法 每个对象都有一个引用计数器，当有引用连接到该对象引用计数器++，当引用离开作用域或者置为 null 引用计数器–，当垃圾回收器发现某个对象的引用计数器为 0 时就释放它，缺点：当对象相互引用的时候引用计数器不为 0，但是两个对象都不再需要，这时候就没法释放了。
    2. 自适应的、分代的、停止-复制、标记-清扫式垃圾回收器。**停止-复制**，先暂停程序从GCRoot (静态区，堆栈引用) 出发找到所有活的对象，并且把这些对于一个紧挨一个放到新的一块内存中，然后更新堆栈中和静态区的引用然后删除原来占据的堆内存，缺点：需要第二块内存，并且当可回收的对象过少的时候效率不高 优点：整理了堆中的对象可以有更多空间来存储。**标记-清扫**，也是先暂停程序也从 GCRoot 出发找对象每找到一个活着的对象就把它标记，等所有的引用都遍历结束后，删除没有被标记的对象。分代的，每个内存块都有一个代数，大型对象不会被复制，内含小型对象的块则会被复制和整理。自适应的，虚拟机会跟踪标记-清扫的效果，要是堆空间中出现很多小碎片就会切换到停止-复制。
* 可以作为 GCRoot 的包括：
    1. Java 虚拟机栈中引用的对象。
    2. 方法区类静态属性引用的对象。
    3. 方法区常量引用的对象。
    4. Native 栈中引用的对象。

* 数组初始化有以下两种，其中第一种只能在定义时初始化，数组分为基本数据类型数组和引用数组，数组本身也是一个对象，可以通过调用 Arrays.toString(arr) 来打印数组。

    ```java
    int[] iArr = {1,2,3,4,5}; //1
    int[] bArr = new int[10]; //2
    Obj[] arrs = new Obj[10]; //创建了一个引用数组，并没有初始化里面的引用，因此都是null
    ```

* 可变参数，使用方式如下，必须位于方法的最后一个参数，就相当于一个数组。

    ```java
    static void print(Object... objects) { //可以传入0-n个Object对象或者是一个引用数组(如果传了基本数据类型数组，那么相当于只是传了一个Object对象)
        System.out.println(Arrays.toString(objects));
    }
    ```

* 枚举使用 enum 关键字，其也是一个类，只是产生了某些编译器行为，编译器会创建toString()、ordinal() 方法，并且会给枚举类创建一个 values() 静态方法获取所有的实例，并且枚举里面声明的枚举对象都是静态的，代码如下：

    ```java
    enum PagerMoney {
        ONE_DOLLAR, FIVE_DOLLARS, TEN_DOLLARS, FIFTY_DOLLARS, ONE_HUNDRED_DOLLARS
    }
    for (PagerMoney money: PagerMoney.values()) { //返回一个PagerMoney数组，包含所有Dog类的实例
        System.out.println(money + " " + money.ordinal()); //该方法获取实例创建的顺序从0开始
    }
    ```

* 静态导入，当静态导入一个类后调用该类中的静态方法就可以不写类名了。

    ```java
    import static com.hfw.utils.CommonUtils.*; //静态导入就不用写类名了
    
    public class TestUtils {
        public static void main(String[] args) {
            print("Hello,World");
    	}
    }
    ```

* 包名必须与文件目录一一对应，因为 java 解释器在加载类的时候会将包名的 . 替换成 / ，生成一个目录，然后在所有的 CLASSPATH 下面的这个目录（由于 CLASSPATH 中含有 . 因此也会查找当前位置）查找.class文件，所以如果包名与文件目录不对应那么 java 解释器将找不到文件。

* 访问控制，在创建类的时候最好把字段和方法按照 public、protected、default、private 的顺序书写，外部类只有 public 和 default 两种访问控制，内部类具有四种（如果设置为 private 则只有同属于一个最外部才能访问，其默认构造方法也是 private，如果设置为 protected，则其默认构造方法也是 protected）也就是**默认构造器的访问控制符与类的访问控制符一致**。

    1. public 全局都可以访问被 public 修饰的类或者字段或者方法。
    2. protected 同包、同类、子类可以访问该方法。
    3. default，同包、同类中能访问。
    4. private 只能在同属于一个最外部类访问。

* 当创建一个子类对象时，其实会先创建一个父类对象，相当于一个子类对象中包含一个父类对象（解释了为什么子类对象不能调用父类对象的 private 字段和方法）并且在子类构造器中会首先调用父类构造器，**父类构造器如果有参数则必须在子类构造器中明确使用 super 调用父类构造器**（**在父类构造器没走完之前是不会初始化子类的非静态的字段的，当父类构造器执行完后才初始化子类非静态字段，然后继续执行子类的构造器）**，测试代码如下：

    ```java
    package com.hfw.section7.practice;
    public class PracticeFive {
        public static void main(String[] args) {
            new CC();
        }
    }
    class AA {
        AA() {
            System.out.println("AA");
        }
    }
    class BB {
        BB() {
            System.out.println("BB");
        }
    }
    class CC extends AA {
        BB bb = new BB();
        CC() {
    	    System.out.println("CC");
        }
    }
    //outputs: 
    AA
    BB
    CC
    ```

* 代码复用有以下几种方式：

    1. 组合， 一个类中包含另一个类的引用，需要时从该类获取引用调用其方法。
    2. 继承， 调用时直接调用其方法。
    3. 代理 ，结合前两者，其拥有另一个类引用，并且创建对应的方法如 f()，然后在该方法中调用另一个类对象的 f()，**idea可以自动生成代理方法**。

* 关键字 final 的用法：

    1. 字段，表示其是一个常量，**被final修饰的字段只能在定义时或者构造器中初始化**，而一旦初始化就不能更改了。

    2. 方法参数，在方法内部无法更改参数，常用与方法内部的匿名内部类。

    3. 方法，不可以被重写，基类的 private 方法隐含 final，因为 private 方法不能被重写，子类就算写了一个与父类声明一样的方法也不是重写此外 static 方法也不能被重写。

    4. 类 不可以被继承。

* 方法绑定 java 中除了 static、final (private也属于final) 方法外，其他所有方法都是动态绑定（也就是在运行时绑定），多态的实现就是依赖动态绑定，在运行时确定调用的方法。

* 多态 子类可以向上转型成父类(将子类对象赋值给父类)，并且在运行时程序能够正确调用方法，书写程序时最好只与基类进行通信而不依赖于某一个具体的实现，这样程序就是可扩展的。多态是一项将改变的事物与未变的事物分离开来的重要技术，当基类中需要增加一些其他方法时完全不影响原来的代码。注意点：1. 子类是无法重写父类的私有方法的，因此上转型到父类调用该方法会调用父类的方法(**在子类中不要创建与父类的私有方法一样的方法**) 2. 如果父类拥有一个字段A，子类也拥有字段 A，那么将子类上转型到父类获取 A 得到的是父类的 A 字段，因为这个访问在编译器进行，如果采用调用方法获取则获取的是子类的A字段(**也就是字段不属于多态**)。3. 静态方法不具有多态。

    ```java
    public class TestPolymorphism {
        static class Super {
            int field = 0;
        }
        static class Sub extends Super {
            int field = 1;
            int getField() {
                return field;
            }
            int getSuperField() {
                return super.field;
            }
        }
        public static void main(String[] args) {
            Super su = new Sub();
            System.out.println(su.field); //直接获取得到的是父类的0，编译时决定
            System.out.println(su.getField()); //方法调用得到的是子类的1，运行时决定
        }
    }
    ```

* 构造器 准则 "用尽可能简单的方法使对象进入正常状态，如果可以的话避免调用其他方法，在构造器能安全的调用的方法只有当前类的 fina l和 private 方法因为它们不能被继承"，如果不遵守可能会出现下面的问题。

    ```java
    public class PracticeFifteen {
    
      public static void main(String[] args) {
           Circle circle = new Circle();
      }
      static class Shape {
            Shape() {
                print();
            }
            void print() {
                println("Shape print");
            }
      }
      static class Circle extends Shape {
            int radius = 2;
            Circle() {
            }
            void print() {
                println("Circle print" + radius); 
            }
        }
      }
    }
    // outputs: Circle print 0
    ```

    因为在堆中创建了空间后所有值都被置于 0，而在 Shape 构造器执行的时候 Circle 的成员变量还没赋值。

* 抽象类，如果一个类的父类不是抽现类，而该类声明了abstract并且没有任何抽象方法则该类不能被实例化。

* 接口，接口的所有字段都是 public static final 的，方法都是 public abstract 的，所以要求实现类的方法也必须是 public 的。接口中的字段不能是空 final 的(声明的时候不赋值，而在构造器赋值)。 接口可以继承另一个接口(也可以继承多个接口中间用逗号分割)，一个类可以实现多个接口中间用逗号分割。在写自己的类库时方法最好是需要传入一个实现了该类库中的接口的实例(声明为你可以用任何你想用的对象来调用我，只要你的对象遵循我的接口)。
* 内部类 分为成员内部类、局部内部类、静态内部类，接口中定义的内部类一定是静态内部类，**非静态内部类**对象含有一个外部类引用，所有能够访问外部类的所有成员包括 private 字段、方法（可以通过外部类类名.this.xxx 来明确使用外部类的 xxx)。要创建一个内部类对象必须要先有一个外部类对象（假设外部类名为 Outer 对象为 outer 内部类为 Inner），然后使用outer.new Inner()，外部类对象无法访问内部类的任何字段。可以在，类、方法(称作局部内部类)、任意的{}作用域，定义内部类。但是不管内部类定义在哪里在编译时会一并被编译成 .class 文件。**匿名内部类**如果用到了使用了一个在外部定义的变量，则该变量必须是final的，也就是说必须是常量。**内部类也是可以被继承的**，但是构造方法必须传入外部类对象并且要调用父类对象.super()。
* 静态内部类创建不需要外部类引用，无法访问外部类的非静态成员，内部可以创建 static 变量和方法，其实就相当于一个与外部类没关系的类了，普通内部类内部不能创建static方法和字段。

* 打印容器不需要循环打印直接 println(Collection) ，容器内不能存储基本数据类型只能存储对象，程序中不应该使用 Vector、Hashtable。
* Collections 包括以下几种：
    1. List，包括 ArrayList 强随读取，弱插入移除；LinkedList 强插入移除，弱随机读取，其还能当做队列和栈使用。
    2. Set，包括 HashSet 无序最快查找基于散列表实现；TreeSet 有序，但是查找稍慢、LinkedHashSet 两者折中 )。
    3. Queue，FIFO，poll 、peek ，offer ，PriorityQueue 内部使用堆实现，分为小顶堆和大顶堆，解决 TOP N 问题会使用。
    4. Stack FILO，push、pop、peek、isEmpty。

* Map 包括以下几种实现：
    1. HashMap 最快无序查找 O(1)。
    2. TreeMap 基于红黑树，一般查找 O(logn)，按插入序。
    3. LinkedHashMap 快查找 O(1)，按插入序或者 LRU。

* Iterator 迭代器 hasNext、next、remove，当一个类需要能够迭代时可以创建一个迭代器成员变量。

* ListIterator List 迭代器，通过 listIterator(index) 设置锚点起始值，可以做到倒序 比Iterator多的方法：hasPrevious、Previous、set、nextIndex、previousIndex。

* 增强 for 循环（forEach）可以用于数组、任何实现了 Iterable 的类 注:可以自己实现Iterable用于定制，比如生成逆序迭代。

    ```java
    public Iterable<Integer> reverseIterable() {
        //返回一个逆序的Iterable
        return new Iterable<Integer>() {
            int index = PracticeThirtyTwo.this.size();
            @Override
            public Iterator<Integer> iterator() {
                return new Iterator<Integer>() {
                    @Override
                    public boolean hasNext() {
                        return index > 0;
                    }
                    @Override
                    public Integer next() {
                        return get(--index);
                    }
                };
            }
        };
    }
    ```

* Exception printStackTraces 打印栈的轨迹可以传入 PrintStream 或者 WriteStream 在执行中会调用 getMessage。fillInStackTrace 一般用于捕获了一个异常然后再次将其抛出。finally 会在 try catch 执行完后执行，如果在 try catch 中执行了return，会先执行 return，但是方法还没结束方法会**等到 finally 执行完后**才结束。

* RuntimeException 继承于该类的异常 （NullPointException、ArrayIndexOutOfBoundsException 等等）属于不受检查的异常，方法不需要声明 throws，不进行 catch 也能够通过编译，如果调用链全都没处理，那么最后会传到 main，然后把异常传给System.err 进行打印。

* 两种情况会导致异常丢失，一种是在 finally 语句中使用return，会丢失前面抛出的未捕获异常，另一种是在子 try 代码块中抛出一个异常没有 catch，并且 finally 也抛出一个异常，这样在外层 catch 中仅仅能捕获到 finally 中抛出的异常，子 try 代码块中的异常将被忽略。

* 子类构造器不能捕获父类构造器抛出的异常，因为子类构造器中没法用 try catch 包裹 super()，super() 必须位于第一行。

* 异常匹配 当 try 代码块中抛出异常后，会寻找最近一个匹配的 catch 语句然后就认为异常处理结束，所以 Exception 要放到最后，因为它能匹配(父类可以匹配子类)所有异常。

* String 对象不可变（只读），任何看起来修改 String 的方法都只是返回了一个新的 String 对象，String 中的 +/+= 是 java 中唯一的两个重载操作符，StringBuilder/SE5新加的（常用方法append、delete、toString）线程不安全，StringBuffer线程安全。

* String +/+= 本质上是编译器通过 StringBuilder 实现的，在循环体里面记得不要使用 +/+=，因为这会创建很多个 StringBuilder 对象，应该在循环体外手动创建一个 StringBuilder 对象。

* String Pool，在类链接时，会将字节码中的常量池进行遍历，一个个与 String Pool 中的字符串进行比较，如果没有就新建一个字符串对象，将其引用保存在 String Pool 中。

* 正则表达式

    1. ^  输入序列的首个字符

    2. $  输入序列的最后一个字符

    3. \G  前一个匹配的结束??

    4. .  匹配任意除了\r\n以外的所有字符

    5. \*  0次或多次匹配前面的字符或者表达式

    6. \+  1次或多次匹配前面的字符或者表达式

    7. [A-Z]  匹配所有大写字母

    8. [a-zA-Z]  匹配所有字母

    9. [abc[de]]  等价于[abcde]，并集

    10. [a-z&&[abc]]  匹配a或b或c，交集

    11. ?  一个或零个

    12. |  或

    13. [xyz]  匹配里面包含的任一字符

    14. [\^xyz]  匹配除了里面包含的任一字符

    15. B  指定字符B

    16. \t 制表符

    17. \n 换行符

    18. \r 回车符

    19. \f 换页符

    20. \d  数字[0-9]

    21. \D  非数字[\^0-9]

    22. \W  非词字符等价于[\^\w]

    23. \w  词字符[a-zA-Z0-9]

    24. \s 空白符(空格、tab、换行、回车、换页)

    25. \S 非空白符

    26. \b  词的边界

    27. \B  非词的边界

    28. X{n}  恰好n次X

    29. X{n,} 至少n次X

    30. X{n.m} n<=X出现次数<=m

    31. \xhh 十六进制值为0xhh的字符

    32. \uhhhh 十六进制值为0xhh的unicode字符

    33. \\" 匹配双引号因为引号需要转义

    34. 标记模式 前面的常量用于Pattern.compile （第二个参数可以使用|分割以支持多种标记），?! 用于正则表达式，效果一致

        1. Pattern.CASE_INSENSITIVE(?!) 匹配忽略字母大小写
        2. Pattern.DOTALL(?x) inputCharSequence 中的空格已经以#开头行将被忽略
        3. Pattern.MULTILINE(?m) 在该模式下^、$分别匹配一行的开头和结束，默认两个分别匹配 inputCharSequence 的首字符和尾字符

    35. 逻辑操作符 X|Y。

    36. 量词 普通贪婪型、勉强型、占有型尾部加+ 包括(?、*、+、{4,}、{4,7})这几种默认是普通贪婪型的即匹配最多字符，如果在尾部加上? 则变成勉强型匹配最少字符，因为上述几种匹配字符数都不确定只是一个范围所以需要有量词。

    37. ^|$ 可以匹配第-1位元素和s.length位元素所以 "Hello".replaceAll("^|$", "#") 会变成#Hello#。

    38. XYZ+ 表示 XY 字符加上一个或多个Z，(XYZ)+ 表示一个或多个XYZ。

    39. **将价格格式化为每三个数子带一个 , 可以使用该正则表达式，这里的 ?!^ 不代表忽略大小写，代表的是忽略首位，?= 代表的是匹配的首个字符与前一个字符之间。**

        ```java
        private void convert() {
            String regex = "(?!^)(?=(\\d{3})+$)"
            "1234567".replaceAll(regex); // 1,234,567
        }
        ```

* 反射 getXXX 与 getDeclaredXXX 的区别时前者返回本类及其父类的 public 域，后者返回当前类的所有域，getEnclosingClass 获取外部类的 Class 对象。

* Collection.nCopies() 可以创建一个 List 里面每个 Item 都一样，这个 List 不能 set、add 因为其没重写 AbstractList 对应的方法，可以将其传入别的容器的构造器或者传入 addAll 方法，比较二路快排和三路快排有点用。

* Collections.fill()，将 List 中的所有元素都替换成指定元素，如果里面没元素则不进行任何操作。

* Collection.toArray()、Collection.toArray(T[]) 前者返回一个 Object 数组，后者返回一个与传入数组类型一致的数组，如果类型会抛出异常，传入的数组 length 如果小于 size 则返回 size 大小的数组，否则将 T[] 中空余部分填充为 null。

* Collections.max、Collections.min 要求也是必须实现 Comparable 接口或者传入一个Comparator。

* Collections.unmodifiableList，用于创建一个不可更改的 List，也就是只读的，调用其他方法将抛出 UnSupportOperationException。

* Collections.swap(List, i, j)，交换元素。

* Collections.synchronizedXXX() 转化为线程安全的容器。

* getPath 获取的是相对路径也就是短路径，getAbsolutePath 获取的绝对路径也就是长路径。

* canRead、canWrite返回是否可以读写。

* renameTo 可以用来修改文件名也可以用来移动文件比如剪切。

* mkdir 创建一层路径，如果目标路径的 parent 路径不存在则失败，mkdirs 创建 n 层路径，如果 parent 路径不存在则创建，调用了这两个方法那么该 File 对象就代表了一个路径。

* Java 中的流包括4个基类 InputStream、OutputStream、Reader、Writer，前两个操作的是字节，后两个操作的是字符。以及4个装饰流基类 FilterInputStream、FilterOutputStream、FilterReader、FilterWriter。

* InputStream、OutputStream转化为 Reader、Writer 通过 InputStreamReader 和OutputStreamWriter 转化。

* FileOutputStream 默认情况下写入一个已经存在的文件会先把文件清空然后再写入，可以通过给构造器传入第二个参数表明是否采用 append 方式。

* Java 中提供了 4 个源注解，用于修饰其它的注解，分别是：

    1. @Documented 注解是否包含在 JavaDoc 中。
    2. @Retention 什么时候使用该注解。
    3. @Target 注解用于什么地方。
    4. @Inherited 是否允许子类继承该注解。

    @Retension 取值为：

    1. RetensionPolicy.SOURCE：在编译阶段丢弃，比如 @Override、@SuppressWarnings
    2. RetensionPolicy.CLASS：在类加载时丢弃，处理字节码文件有用。
    3. RetensionPolicy.RUNTIME：始终不会丢弃，运行时可以使用反射获取。一般自定义时使用这种。

    @Target 取值为：

    1. ElementType.CONSTRUCTOR: 用于描述构造器
    2. ElementType.FIELD: 成员变量（包括enum实例）
    3. ElementType.LOCAL_VARIABLE: 用于描述局部变量
    4. ElementType.METHOD: 用于描述方法
    5. ElementType.PACKAGE: 用于描述包
    6. ElementType.PARAMETER: 用于描述参数
    7. ElementType.TYPE: 用于描述类、接口、枚举

* 并行与并发 并发一个时间片执行任务 A，下个时间片执行任务 B。并发同一时间片同时执行任务 A 和任务 B。
* Thread.yield 表明当前线程暂时不需要 CPU 可以让给其他线程，不保证一定会让出 CPU，只是建议可以把 CPU 让给其他相同优先级的线程，不会让出锁（sleep 方法也一样不会让出锁），wait 方法会让出锁（与notify和notifyAll一样只能在同步方法和同步代码块中调用，不然运行时会抛出异常）。

* ExecutorService 不用了要记得 shutdown 不然会等到里面的所有线程都死了才会关闭，构造器同时接受一个 ThreadFactory 对象，可以用来设置优先级，后台线程，名称等。
* SingleThreadExecutor 等于FixedThreadExecutor(1)，里面加入的所有任务会按照顺序执行。
* 父线程无法捕获子线程抛出的异常，子线程抛出了异常没捕获不会影响到父线程（**Android 是个例外如果子线程抛出了一个未捕获的 RuntimeException 会导致程序 Crash，因为默认异常处理器，发现任何线程抛出异常都会强行终止app**）。
* 可以对线程对象调用 thread.setUncaughtExceptionHandler，当该线程抛出来一个未捕获的异常时会回调该方法，也可以调用 Thread 类的静态方法 Thread.setDefaultUncaughtExceptionHandler,当 thread 没有调用 setUncaughtExceptionHandler 时会调用默认的 DefaultUncaughtExceptionHandler，如果设置了就不调用，就近原则(有具体的调用具体，没有具体的调用默认的)。
* 后台线程（Daemon）在还有非后台线程在运行的情况下可以一直运行，当没有非后台线程在运行了会自动退出，只要在调用 Thread.start 之前调用 setDaemon，一个后台线程启动的所有线程都是后台线程，并且在非后台线程全都停止运行了以后会立刻关闭所有的后台线程，就算后台线程有 finally 语句也不会得到执行。
* 实现 Runnable 与继承 Thread 的区别，前者还可以继承其他类，后者不能再继承了。
* Thread.join 表示当前线程需要等待其它线程执行完后才能继续运行，也可以传入一个 timeout，表示最多等待多少毫秒。
* 自增自减操作不是原子的，包括读取、增加或减少、赋值。
* 如果要唤醒处于 wait 的线程首先必须获得 wait 方法所锁住的对象锁，然后调用该对象的 notify方法（调用该方法不会立即释放锁，而会等到所在的同步代码块执行完后才会释放锁）。
* notify 只会随机唤醒一个处于wait的线程，notifyAll 会唤醒所有处于等待的线程(并且线程间公平竞争锁)。
* 线程一共有以下 5 个状态

    1. 新建状态（New）
    2. 就绪状态（Runnable）
    3. 运行状态（Running）
    4. 阻塞状态（Blocked）
    5. 死亡状态（Dead）
* 进入阻塞状态的几大原因
    1. 调用Thread.sleep()，任务在指定时间内不会得到运行
    2. 调用wait()，直到线程得到了 notify、notifyAll、signal、signalAll 的消息线程才会进入就绪状态
    3. 线程在等待某个输入/输出完成
    4. 任务试图调用某个同步代码块或者同步方法，而又得不到锁，会进行阻塞
* synchronized 提供原子性可见性，volatile 提供可见性和有序性。

