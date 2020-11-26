# Gradle 脚本的执行时序

Gradle 脚本的执行分为三个过程：

1. 初始化。分析有哪一些 module 需要被构建，为每个 module 创建 project 实例。这时候 settings.gradle 文件会被解析。

2. 配置。处理所有 module 的 build.gradle，处理依赖、属性等。这个时候每个模块的 build.gradle 文件会被解析并且配置，会构建整个 task 的依赖链。

3. 执行。根据 task 链来执行某个特定的 task，这个 task 依赖的其他 task 都会被提前执行。

## 执行时序

下面以一个名叫 AndroidSample 的工程为例。

AndroidSample 有 3 个 module：app、annotations、processor。app 模块依赖 annotations 和 processor。processor 依赖 annotations。

给每个 gradle 文件的开始和结尾分别加上打印日志：

settings.gradle

```
println "settings start"
include ':app', ':annotations', ':processor'
println "settings end"
```

AndroidSample 的 build.gradle

```
// Top-level build file where you can add configuration options common to all sub-projects/modules.
println "AndroidSample start"

buildscript {
    ...
}

...
task clean(type: Delete) {
    delete rootProject.buildDir
}

println "AndroidSample end"
```

app 的 build.gradle

```
println "app start"
apply plugin: 'com.android.application'
...

dependencies {
    ...
    // annotation
    implementation project(':annotations')
    annotationProcessor project(':processor')
    ...
}

println "app end"
```

processor 的 build.gradle

```
println "processor start"

apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation project(':annotations')
    ...
}

sourceCompatibility = "8"
targetCompatibility = "8"

println "processor end"
```

annotations 的 build.gradle

```
println "annotations start"

apply plugin: 'java-library'

dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')
}

sourceCompatibility = "8"
targetCompatibility = "8"

println "annotations end"
```

然后执行 gradlew clean，查看日志输出：

```
C:\Users\m\AndroidStudioProjects\AndroidSample>gradlew clean
Starting a Gradle Daemon, 1 incompatible Daemon could not be reused, use --status for details
settings start
settings end

> Configure project :
AndroidSample start
AndroidSample end

> Configure project :annotations
annotations start
annotations end

> Configure project :app
app start
app end

> Configure project :processor
processor start
processor end

```

通过控制台日志可以看出，先执行初始化，即解析 settings.gradle 获取模块信息，然后进入配置阶段。首先执行项目 AndroidSample 的 build.gradle 文件，因为 app 和 processor 都依赖了 annotations 模块，所以 annotations 模块的 build.gradle 先执行，然后再执行 app 和 processor 的 build.gradle。

注意到 app 使用了 processor 却不是 implementation 依赖。app 的配置在 processor 之前。

```
annotationProcessor project(':processor')
```

## afterEvaluate

如果给 app 模块的 build.gradle 加上 afterEvaluate 方法，它可以在 app 配置之后执行。

```
...
project.afterEvaluate {
    println "afterEvaluate start"
    println "afterEvaluate end"
}
println "app end"
```

输出如下:

```
> Configure project :app
app start
app end
afterEvaluate start
afterEvaluate end
```

可以看出 app 配置完毕后才会执行 afterEvaluate。

## 总结

Gradle 的执行时序有如下规则：

1. 解析 settings.gradle 获取模块信息，进入初始化阶段
2. 配置项目的 build.gradle
3. 配置各个模块的 build.gradle，注意配置的时候不会执行 task
4. 配置完成后如果有 project.afterEvaluate，会执行 afterEvaluate 表示配置完毕
5. 执行 gradlew 指定的 task，比如 gradlew clean，用来删除 build 目录