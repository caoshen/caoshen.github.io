---
title: Gradle 自定义 task
date: 2019-10-12 00:23:25
tags: Gradle
categories: Android
---

# Gradle 自定义 task

task 是 gradle 的执行单元，gradle 通过一个个 task 完成具体任务的执行。

build.gradle 中自带的 clean task 如下：

```
task clean(type: Delete) {
    delete rootProject.buildDir
}
```

执行 gradlew clean 命令时会删除 build 目录。

## doFirst 和 doLast

自定义 task 如下

```
task doMytask {
    println "doMytask"
}
```

执行 gradlew doMytask 时，会在 gradle 的配置阶段打印 "doMytask"

```
> Configure project :app
doMytask
```

很多时候需要 task 闭包的代码仅仅在 task 的执行阶段运行。这时可以用 doFirst 或者 doLast 操作。

- doFirst 表示 task 执行时，最开始的操作
- doLast 表示 task 执行时，最后的操作

```
task doMytask {
    println "doMytask"
}

doMytask.doFirst {
    println "doMytask doFirst"
}

doMytask.doLast {
    println "doMytask doLast"
}
```

这时可以在 gradle 的执行阶段运行：

```
> Task :app:doMytask
doMytask doFirst
doMytask doLast
```

## TaskAction

通过 @TaskAction 注解也可以指定 task 执行时做的事情。

下面定义两个 task1、task2，它们都是 ActionTask 类型。

```
class ActionTask extends DefaultTask {
    String message = "task"

    @TaskAction
    def printHello() {
        println "hello, ${message}"
    }
}

task task1(type: ActionTask)

task task2(type: ActionTask) {
    message = "task2"
}
```

执行 gradlew task1 task2。结果如下

```
> Task :app:task1
hello, task

> Task :app:task2
hello, task2
```

可以看出 task2 的 message 变量被替换成 "task2"。

## type

gradle 有一些预置的 task，比如 Copy 复制 task。

如果需要将 app-release.apk 重命名为 AndroidSample.apk，可以在项目的 build.gradle 定义如下 task:

```
task renameApk(type: Copy, dependsOn: ':app:assembleRelease') {
    from 'app/build/outputs/apk/release'
    into 'app/build/outputs/apk/release'
    include '*.apk'
    rename {
        String fileName -> fileName = "AndroidSample.apk"
    }
    duplicatesStrategy 'exclude'
}

project.tasks.findByPath(':app:assembleRelease').finalizedBy renameApk
```

renameApk task 是 Copy 类型，它依赖 assembleRelease 任务完成。通过 findByPath 方法找到 assembleRelease 任务，然后通过 finalizedBy 指定接下来的重命名 task，即 renameApk task。