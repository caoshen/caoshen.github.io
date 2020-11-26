---
title: Kotlin 带接收者的 lambda ：with 和 apply
date: 2019-10-19 00:11:43
tags: Kotlin标准库
categories: Kotlin
---

# Kotlin 带接收者的 lambda ：with 和 apply

Kotlin 标准库（Standard.kt）中提供了一些函数，使用它们可以方便地改写原有的代码。

with 和 apply 方法是其中的 2 个函数，它们都可以接受一个 lambda 函数体，而且在 lambda 函数体可以调用对象。

## with

假如有以下代码，输出 A - Z 和一段文字。

```kotlin
fun alphabet(): String {
    val result = StringBuilder()
    for (letter in 'A'..'Z') {
        result.append(letter)
    }
    result.append("\nNow I know the alphabet!")
    return result.toString()
}
```

使用 with 函数改写如下：

```kotlin
fun alphabetWith(): String {
    val result = StringBuilder()
    return with(result) {
        for (letter in 'A'..'Z') {
            this.append(letter)
        }
        this.append("\nNow I know the alphabet!")
        this.toString()
    }
}
```

这里的 this 指向 result。

代码中的 this 可以省略，改写如下：

```kotlin
fun alphabetWith(): String {
    val result = StringBuilder()
    return with(result) {
        for (letter in 'A'..'Z') {
            append(letter)
        }
        append("\nNow I know the alphabet!")
        toString()
    }
}
```

可以将函数体转换成表达式函数：

```kotlin
fun alphabetWith() = with(StringBuilder()) {
    for (letter in 'A'..'Z') {
        append(letter)
    }
    append("\nNow I know the alphabet!")
    toString()
}
```

这样的写法对比最初的写法少了很多 result 变量的调用。假如一个类有很多属性，可以直接使用 with 对它进行操作。


### 方法名称冲突

如果 alphabetWith 想调用外部类的同名方法 toString。可以使用 this@OuterClass.toString 如下：

```kotlin
class Main {
    fun alphabetWith() = with(StringBuilder()) {
        for (letter in 'A'..'Z') {
            append(letter)
        }
        append("\nNow I know the alphabet!")
        this@Main.toString()
    }

    override fun toString(): String {
        return "Main toString"
    }
}
```

输出

```
Main toString
```

可以看出调用的是 Main 类的 toString。

## apply

apply 和 with 很类似。with 的返回值是整个 lambda 的返回值，而不是 with 第一个参数本身。

如果想完成和 with 相同的作用，但是不想传递一个对象或者想返回调用者本身，就用 apply。

```kotlin
fun alphabetApply() = StringBuilder().apply {
    for (letter in 'A'..'Z') {
        append(letter)
    }
    append("\nNow I know the alphabet!")
}.toString()
```

可以看出 StringBuilder 对象直接调用 apply，apply 返回了一个 StringBuilder，最后调用 StringBuilder 的 toString。

上述代码等价于

```kotlin
fun alphabetBuildString() = buildString {
    for (letter in 'A'..'Z') {
        append(letter)
    }
    append("\nNow I know the alphabet!")
}
```

buildString 是 StringBuilder.kt 提供的标准函数。

它的定义如下：

```kotlin
public inline fun buildString(builderAction: StringBuilder.() -> Unit): String =
    StringBuilder().apply(builderAction).toString()
```

可以看出 buildString 封装了 apply。

## with 和 apply 的实现

with 函数：

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

apply 函数：

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

with 和 apply 都调用了传入的 block 函数。它们区别在于：

1. with 有一个 receiver 参数来表示接收者，而 apply 是一个扩展函数，调用者可以直接调用 apply。
2. with 的返回值是 block 函数的返回值，而 apply 的返回值是调用者本身。