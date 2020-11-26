# Kotlin 扩展函数和扩展属性

Kotlin 的扩展函数是定义在类外面的成员函数。

## 扩展函数

假如现有的类缺少一个你想要的方法，但是又无法改变它的内部结构，可以使用扩展函数给它添加一个方法，这一点类似 Java 一些工具类的静态方法。

下面为 String 类添加一个 lastChar 扩展方法，用来查找最后一个字符。

```kotlin
package string

fun String.lastChar() = get(length - 1)
```

使用方法如下：

```kotlin
fun main() {
    println("Kotlin".lastChar())
}
```

输出

```
n
```

在扩展函数中，可以直接访问被扩展的类的其他方法和属性，比如 String 类的 get 方法和 length 属性。

## 导入扩展函数

可以直接导入扩展函数

```kotlin
import string.lastChar

fun main() {
    println("Kotlin".lastChar())
}
```

也可以导入包

```kotlin
import string.*

fun main() {
    println("Kotlin".lastChar())
}
```

也可以重命名函数

```kotlin
import string.lastChar as last

fun main() {
    println("Kotlin".last())
}
```

调用 last 等于调用 string.lastChar。

## 从 Java 调用 Kotlin 的扩展函数

Java 类也可以使用 Kotlin 的扩展函数，但是使用方法有些区别。

```kotlin
import string.StringKt;

public class StringTest {

    public static void main(String[] args) {
        System.out.println(StringKt.lastChar("Kotlin"));
    }
}
```

Kotlin 的扩展函数被转换成一个静态方法，它的第一个参数相当于 Kotlin 的调用者 "Kotlin"。然后 string.kt 文件被转换成 StringKt 类，lastChar 是 StringKt 的静态方法。

## 作为扩展函数的工具函数

先看一个标准库扩展函数 joinToString，它相当于一个工具函数，用来拼接字符串。

```kotlin
public fun <T> Iterable<T>.joinToString(separator: CharSequence = ", ", prefix: CharSequence = "", postfix: CharSequence = "", limit: Int = -1, truncated: CharSequence = "...", transform: ((T) -> CharSequence)? = null): String {
    return joinTo(StringBuilder(), separator, prefix, postfix, limit, truncated, transform).toString()
}
```

joinToString 调用了 joinTo 函数。

```kotlin
public fun <T, A : Appendable> Iterable<T>.joinTo(buffer: A, separator: CharSequence = ", ", prefix: CharSequence = "", postfix: CharSequence = "", limit: Int = -1, truncated: CharSequence = "...", transform: ((T) -> CharSequence)? = null): A {
    buffer.append(prefix)
    var count = 0
    for (element in this) {
        if (++count > 1) buffer.append(separator)
        if (limit < 0 || count <= limit) {
            buffer.appendElement(element, transform)
        } else break
    }
    if (limit >= 0 && count > limit) buffer.append(truncated)
    buffer.append(postfix)
    return buffer
}
```

joinTo 函数把集合的每个元素一个个 append 到字符串中。

可以利用 joinToString 定义一个 String 集合的 join 函数。

```kotlin
fun Collection<String>.join(separator: String = ",", prefix: String = "", postfix: String = "") =
    joinToString(separator, prefix, postfix)
```

```kotlin
fun main() {
    println(arrayListOf("one", "two", "three").join(" "))
}
```

输出

```
one two three
```

## 不可重写的扩展函数

因为扩展函数是跟随某个类的，因此它具有静态性，不能被子类重写。

```kotlin
open class View

class Button: View()

fun View.showOff() = println("View showOff")
fun Button.showOff() = println("Button showOff")

fun main() {
    val view:View = Button()
    view.showOff()
}
```

输出如下

```
View showOff
```

假如改成：

```kotlin
fun main() {
    val view:Button = Button()
    view.showOff()
}
```

输出如下：

```
Button showOff
```

扩展函数不存在重写，它相当于被扩展类的静态方法。

第一种情况相当于 Java 的 View.showOff(view)。

第二种情况相当于 Java 的 Button.showOff(view)。

如果在 IntelliJ IDEA 中观察代码：

```kotlin
view.showOff()
```

可以发现 showOff 是斜体的，说明 IDE 在提示它是一个静态方法。

## 扩展属性

扩展属性可以给类提供一种方法，用来扩展类的 API。但是尽管它被成为属性，它也没有给现有的对象添加额外的字段。


```kotlin
val String.lastChar: Char
    get() = get(length - 1)

fun main() {
    println("kotlin".lastChar)
}
```

定义扩展属性时，需要提供一个 get 方法用来访问它。


也可以声明一个可变的扩展属性，比如 StringBuilder。

```kotlin
var StringBuilder.lastChar: Char
    get() = get(length - 1)
    set(value: Char) {
        setCharAt(length - 1, value)
    }

fun main() {
    val stringBuilder = StringBuilder("kotlin?")
    stringBuilder.lastChar = '!'
    println(stringBuilder)
}
```

输出如下：

```
kotlin!
```