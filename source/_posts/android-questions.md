---
title: Android 问题记录
date: 2019-03-17 10:12:08
tags: 面试
categories: Android
---

# Android 问题记录

## 核心知识点

### Android 四大组件

#### Activity

同一个 task 中的同一个 stack，有三个普通的 activity，A 启动 B，B 启动 C。切到后台后被系统杀死。再次从最近应用列表里打开，会出现什么情况？task 中有几个 activity？谁的 onCreate 会执行？

两个 activity A, B，A 跳转到 B，并传递了 bean 数据，按 home 键，过一段时间如果应用被系统杀死，重新进入应用，B 恢复。请问 bean 会为空吗？为什么？

### View

#### 自定义 View

[Android自定义View系列](https://www.jianshu.com/p/22c2444c83fa)

如何设计一个抽奖动画？

#### View 事件分发

事件是怎么确定到某个 view 上的？如：内外层 view，点击内层 view，内层 view 会触发事件，这是怎么个流程？

## 开源框架

rxjava 操作符有哪些？

okhttp 源码

glide 源码

eventbus 源码

## 热修复

## 插件化

[占位式插件化之加载Activity](https://blog.csdn.net/mingyunxiaohai/article/details/99478783)

## JVM

### Java 垃圾回收

## 网络

Socket

[手把手教你写 Socket 长连接](https://mp.weixin.qq.com/s/035SiIDhmbHzbtI1fF4h1g)