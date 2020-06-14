---
title: JavaScript 基础
date: 2017-11-13 11:12:36
type: "JavaScript"
---
### 数据类型

JS 一共有六种数据类型，包括基本数据类型：String、Number、Boolean、Null、Undefined，以及引用类型 ：Object ，可以使用关键字 typeof 判断数据类型。<!-- more --> 

#### Number

不区分整数和浮点数，所有的数组类型都是 Number ，NaN表示Not a number当无法计算时会出现如：0/0，Infinity表示无限大，当超出最大计算范围时会出现如：2/0。

#### String

字符串可以使用单引号或者双引号，但是不能一个单引号一个双引号，如：'abc', "abc"。跟 Java 一样，JS 里面的字符串也是不可变的，toUpperCase、toLowCasr、subString 都只是返回一个新的字符串，此外ES6 支持模块字符串如：console.log('My name is ${name}')。

#### Boolean

只有 true、false 两个取值。

#### Null

只有 null 一个取值，用于表示空对象。

#### Undefined

只有 undefined 一个取值，当声明变量后没进行初始化就是改值。

#### Object

Object 类型包括普通的对象，由 {} 括起来的一组键值对如：var obj = {username: '123', password: '12345678'}，还包括数组如：var array = []。

获取对象中的属性有三种方式，其中第三种解构赋值一般在取多个属性时使用。

1. obj.username
2. obj['username']
3. let {username} = obj 

可以使用关键字判断对象是否含有某个属性：'username' in obj。 

将对象转换为 Json 字符串可以使用：JSON.stringify(arr)。

JS 中数组拥有很多方法，记住几个常用的：

1. map 接受一个方法，对每个数组元素做修改，然后返回一个数组对象。
2. slice 截取原数组指定位置，返回一个新数组。
3. push 在数组尾部添加元素。
4. pop 在数组尾部删除元素，JS 栈就能靠 push、pop 实现。
5. shift 删除数组第一个元素。
6. unShift 在数组第一个元素前添加一个元素。
7. sort 对数组进行排序。
8. splice 删除数组位置元素，并使用新元素填充。
9. concat 用于拼接数组，返回一个新数组。



### 方法

方法可以不声明任何参数，因为方法内部可以获取 arguments 属性，里面有该方法传入的所有的参数。

方法可以不声明返回类型，因为 JS 是弱类型，统一用 var 接收就行。

方法参数不需要类型，还是因为弱类型。

方法前面可以使用后面声明的变量，因为 JS 会将所有方法中申明的变量提前到方法顶部，不过不要这么做。



#### 作用域

ES6 前只有全局作用域以及方法作用域，ES6 引入 let 关键字，使用它声明的就是块级作用域。



#### 小技巧

JS 中交换两个值非常简单一行代码就搞定了 [x, y] = [y, x]。



