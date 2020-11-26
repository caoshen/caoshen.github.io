---
title: Groovy 语法简介
date: 2019-10-09 01:01:48
tags: Gradle
categories: Android
---

# Groovy 语法简介

Groovy 是一个 JVM 语言，它可以和 Java 兼容，编译成 class 文件在 JVM 上运行。相比较 Java ，Groovy 语法更为简洁。Gradle 构建工具使用 Groovy 语言编写配置。

## Groovy 的 String

Groovy 的 String 类型和 Java 的 String 类似，但是它可以使用 $ 字符串模板来拼接字符串。

```groovy
b = "hello"
println "${b} world"
```

输出如下

```
hello world
```

## Groovy 的 Closure 闭包

Groovy 有一种特殊类型，叫做 Closure 闭包。它可以使用大括号 {} 将一段代码包裹起来，类似于一个函数或者一段代码块。

闭包一般声明如下

```
{ parameters ->
   code
}
```

可以看出具有 {参数 -> 代码} 的形式。

举例如下：

```groovy
def test = {
    a, b -> println "a=${a} b=${b}, a closure"
}

test 1, 2
```

输出如下

```
a=1 b=2, a closure
```

如果闭包没有指定参数，它会有一个隐含的参数 it

```groovy
def test = {
    println "default=${it}, a closure"
}

test 1
```

输出如下

```
default=1, a closure
```

## Groovy 的 List（列表）

List 使用中括号定义如下：

```groovy
def aList = [1, 2.0, "hello", false]

println aList[3]
println aList[-1]
println aList[4]
```

输出如下

```
false
false
null
```

可以看出 Groovy 的 List 可以多种类型混合。

通过下标获取 List 的元素时，可以正向（下标为正数），或者逆向（下标为负数）获取元素。

当下标越界时，认为获取的元素不存在，返回 null。

## Groovy 的 Map（字典）

Groovy 的 Map 定义如下：

```groovy
def aMap = ["id": 1, "name": "groovy", "isJava": false]

aMap.each {
    println "key:${it.key}, value:${it.value}"
}
```

输出如下：

```
key:id, value:1
key:name, value:groovy
key:isJava, value:false
```

可以看出 Groovy 的 map 是键值对的形式，即保存了 String : Object 的元素。

使用 each 方法遍历 map，each 接受一个闭包，如果闭包没有参数，默认有一个 it 参数，可以用来循环迭代 map。

## Groovy File 操作

Groovy 的 File 操作非常简洁。

以下代码会读取当前目录下的 file.groovy 文件，并且将它的每一行打印出来。

```groovy
def file = new File("file.groovy")

file.eachLine {line -> println "${line}"}
```

输出如下：

```
package groovysamples

def file = new File("file.groovy")
file.eachLine {line -> println "${line}"}

println("this is file.groovy")
```

可以看出 file.groovy 文件的每一行代码都被打印出来。

## Groovy 的 == 和 is

Groovy 的 == 类似于 Java 的 equals，即两者的值相等。Groovy 的 is 相当于 Java 的 ==，即同一个对象。

```groovy
class People {
    String name

    People(String name) {
        this.name = name
    }

    boolean equals(o) {
        if (this.is(o)) return true
        if (getClass() != o.class) return false

        People people = (People) o

        if (name != people.name) return false

        return true
    }

    int hashCode() {
        return (name != null ? name.hashCode() : 0)
    }
}

def p1 = new People("groovy")
def p2 = new People("groovy")

println p1 == p2
println p1.is(p2)
```

输出如下：

```
true
false
```