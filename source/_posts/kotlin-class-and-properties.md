---
title: Kotlin 类和属性
date: 2019-10-17 23:20:28
tags: Kotlin
categories: Kotlin
---

# Kotlin 类和属性

## 类

在 Kotlin 定义一个类似 JavaBean 的 Person 类如下：

```kotlin
class Person(val name: String)
```

它等价于 Java 的以下代码，可以由 Intellij 转换过来。

```java
public class Person {
    private final String name;
    
    public Person(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
}
```

像 Person 这种只含数据的类，可以被叫做值对象。

## 属性

在类中声明一个属性和声明一个变量一样，可以使用 val 或者 var。

```kotlin
class Person (val name: String, var isMarried: Boolean)
```

使用 Person 类如下：

```kotlin
fun main() {
    val person = Person("zhangsan", true)
    println(person.name)
    println(person.isMarried)
}
```

## 自定义 getter

Kotlin 的类可以声明自定义的 getter 方法。

```kotlin
class Rectangle(val height: Int, val width: Int) {
    val isSquare: Boolean
        get() = width == height
}
```

属性 isSquare 是根据 height 和 width 得到的，它没有自己的值。

```kotlin
fun main() {
    val rect = Rectangle(10, 10)
    println(rect.isSquare)
}
```

## 目录和包

和 Java 类似，不同包的文件使用时需要先 import。

```kotlin
package geometry.shapes

import java.util.*

class Rectangle(val height: Int, val width: Int) {
    val isSquare
        get() = width == height
}

fun createRandomRectangle(): Rectangle {
    val random = Random()
    return Rectangle(random.nextInt(), random.nextInt())
}
```

Kotlin 文件中，类和函数可以并存，不区分导入的是类还是函数。

```kotlin
package geometry.examples

import geometry.shapes.createRandomRectangle

fun main() {
    println(createRandomRectangle().isSquare)
}
```

Kotlin 文件中可以有多个类和顶层函数，而且不要求文件名和类名一致，文件名可以随意选择。