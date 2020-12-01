---
title: ContentProvider 难点
date: 2019-08-19 23:13:31
tags: 四大组件
categories: Android
---

# ContentProvider 难点

## ContentProvider 的生命周期

ContentProvider 只有 onCreate 这个生命周期方法。

## ContentProvider 的 onCreate 和 CRUD

ContentProvider 运行在哪个线程？他们是线程安全的吗？

onCreate 方法运行在主线程。如果是 ContentProvider 同进程通信，CRUD 运行在主线程。如果是 ContentProvider 跨进程通信，CRUD 运行在 Binder 线程。

## ContentProvider 的内部存储有哪些

ContentProvider 的内部存储只能是 sqlite 吗？

ContentProvider 的内部存储不一定是 sqlite，可以是多种形式的存储。比如 sqlite、SharedPreference、文件、内存。

## ContentProvider 的 onCreate 和 Application 的 onCreate 先后顺序

ContentProvider 的 onCreate 早于 Application 的 onCreate 执行。在 Application 的 onCreate 之前，有 installProvider 和 publishProvider 的过程。


