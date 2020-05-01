# Kotlin 函数和变量

## Hello, World!

一个 Kotlin 的 Hello, World! 程序如下：

```kotlin
fun main(args: Array<String>) {
    println("Hello, World!")
}
```

## 函数

Kotlin 的函数声明如下：

```kotlin
fun max(a: Int, b: Int): Int {
    return if (a > b) a else b
}
```

函数声明以 fun 关键字开始。

max 是 函数名称。

`a: Int` 表示一个整数类型的参数 a。

`: Int` 表示函数的返回值类型是整型。

if (a > b) a else b 类似于 Java 的三元表达式，它是有返回结果的。 


### 表达式函数体

以上函数可以转换为一种表达式函数体。

```kotlin
fun max(a: Int, b: Int): Int = if (a > b) a else b
```

表达式函数体可以不写返回类型，由类型推导完成。

```kotlin
fun max(a: Int, b: Int) = if (a > b) a else b
```

## 变量

声明变量如下：

```kotlin
val answer: Int = 10
```

Int 类型也可以不写

```kotlin
val answer = 10
```

如果没有初始化也没有指定类型，编译报错。

```kotlin
val answer
```

报错是因为 Kotlin 无法推导出 answer 的类型。


### 可变变量和不可变变量

val 声明的是不可变变量。（value）

var 声明的可变变量。（variable）

val 类似 Java 的 final 变量。var 类似 Java 的普通非 final 变量。

需要注意的是，尽管 val 引用自身是不可变的，但是它指向的对象可能是变化的。类似于 Java 的 final。

```kotlin
fun main(args: Array<String>) {
    val langs = arrayListOf("Java")
    langs.add("Kotlin")
    println(langs)
}
```

虽然 langs 是 val 不可变量，但是它引用的 list 可变，可以添加元素。



var 变量可以改变它的值，但是无法改变类型。

```kotlin
fun main(args: Array<String>) {
    var answer = 42
    answer = "no ans"
}
```

以上代码编译报错，因为类型不匹配。


## 字符串模板

Kotlin 可以使用字符串模板来生成字符串。

```kotlin
fun doPrint(args: Array<String>) {
    val name = if (args.size > 0) args[0] else "Kotlin"
    println("Hello, $name!")
}
```

以上代码也可以这么写：

```kotlin
fun doPrint(args: Array<String>) {
    if (args.isNotEmpty()) {
        println("Hello, ${args[0]}!")
    } else {
        println("Kotlin")
    }
}
```

使用 ${} 将一个表达式插入到字符串模板。

如果想在字符串中使用双引号，可以用双引号嵌套。

```kotlin
fun doPrint(args: Array<String>) {
    println("Hello, ${if (args.isNotEmpty()) args[0] else "Kotlin"}!")
}
```