# Dart 语法简介

## 数据类型

int 整数（-2^63 ~ 2^63-1）

double 双精度浮点数

num 既可以表示整数，也可以表示浮点数

String 字符串，采用 UTF-16 编码

bool 布尔值

List 列表

Set 集合

Map 键值对

Runes 表示采用 UTF-32 的字符串

## 操作符

?? 操作符

a ?? b 表示如果 a 是 null，那么表达式返回 b，否则返回 a。
等价于
```
if (a == null) {
    return b;
} else {
    return a;
}
```

??= 操作符

a ??= b 等价于 a = a ?? b

## 函数

函数也是对象，它的类型是 Function。

返回值 函数名（参数） {

}

=> 箭头函数

=> 后面只能跟表达式，不能跟多个语句。表达式可以是函数或者值。

等价于

```
void fun() {
    return expr;
}
```

## 类

每个对象都是类的实例。所有类都继承自 Object 类。