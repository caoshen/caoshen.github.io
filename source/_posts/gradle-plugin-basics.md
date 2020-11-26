---
title: Gradle 插件基础
date: 2019-10-13 00:39:33
tags: Gradle
categories: Android
---

# Gradle 插件基础

Gradle 中的插件可以分为 2 类：
1. 脚本插件
2. 对象插件

## 脚本插件

脚本插件直接在 gradle 文件中编写，其他 gradle 文件通过 apply from 引用这个脚本插件。它本质上是一个 gradle 配置文件。

在项目根目录下新建一个 config.gradle

```
project.task('showconfig').doLast {
    println "$project.name :showconfig"
}
```

然后在 app 模块的 build.gradle 引用 config.gradle

```
apply from: '../config.gradle'
```

执行 gradlew showconfig 命令。
执行结果如下：

```
> Task :app:showconfig
app :showconfig
```

## 对象插件

对象插件指的是实现了 org.gradle.api.Plugin 接口的类。

Plugin 接口如下：

```
public interface Plugin<T> {
    /**
     * Apply this plugin to the given target object.
     *
     * @param target The target object
     */
    void apply(T target);
}
```

对象插件有 3 种编写方式：

1. 直接在 gradle 脚本编写
2. 在 buildSrc 目录编写
3. 在独立的项目编写

### gradle 脚本编写

在 app 的 build.gradle 定义 Plugin 如下：

```
class CustomPluginInGradle implements Plugin<Project> {
    @Override
    void apply(Project target) {
        target.task('customplugin').doLast {
            println 'CustomPluginInGradle'
        }
    }
}
```

然后在 app 的 build.gradle 添加 apply 引用。

```
apply plugin: CustomPluginInGradle
```

执行 gradlew customplugin，结果如下：

```
> Task :app:customplugin
CustomPluginInGradle
```

### buildSrc 编写

在项目中新建一个叫做 buildSrc 的 Java library 模块，然后将 src/main/java 改为 src/main/groovy。

在 groovy 目录中新建一个 BuildSrcPlugin 类。它在 com.android.buildsrc 包中。

```
package com.android.buildsrc;

import org.gradle.api.Plugin;
import org.gradle.api.Project;

class BuildSrcPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        project.task("buildsrcplugin").doLast {
            println 'apply buildsrc plugin'
        }
    }
}
```

然后修改 buildSrc 模块的 build.gradle，添加 gradle 和 groovy 依赖：

```
apply plugin: 'groovy'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation gradleApi()
    implementation localGroovy()
}

sourceCompatibility = "8"
targetCompatibility = "8"
```

最后在 app 模块的 build.gradle 添加插件依赖：

```
apply plugin: com.android.buildsrc.BuildSrcPlugin
```

也可以先 import 再 apply plugin

```
import com.android.buildsrc.BuildSrcPlugin

apply plugin: BuildSrcPlugin
```

执行 gradlew buildsrcplugin，输出如下：

```
> Task :app:buildsrcplugin
apply buildsrc plugin
```

因为 buildSrc 目录是 gradle 的默认目录之一，这个目录下的代码会在编译时自动打包添加到 buildScript 的 classpath 路径下，所以不需要额外的配置就可以被其他模块引用。


上面的引用方式是类名引用，也可以改成 id 引用。id 引用需要先映射一个 id。

在 buildSrc\src\main 目录下新建 resources\META-INF\gradle-plugins 目录。然后添加一个叫做 myplugin.properties 的配置文件，那么插件的 id 就是 myplugin。

myplugin.properties 的内容如下：

```
implementation-class=com.android.buildsrc.BuildSrcPlugin
```

表示映射到 BuildSrcPlugin 类。

最后在 app 的 build.gradle 引用插件：

```
apply plugin: 'myplugin'
```

### 独立工程编写

在 buildSrc 下创建的 plugin 只能在它的项目中复用。如果想要在其他的项目中也用到这个插件，需要在一个独立的工程或者模块中编写插件，然后将编好的插件上传到 maven 仓库。

为了不增加复杂度，直接在原项目下新增一个 standalone 模块，它的结构和 buildSrc 类似。

```
package com.android.standalone;

import org.gradle.api.Plugin;
import org.gradle.api.Project;

class StandAlonePlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        project.task("standalone").doLast {
            println 'apply stand alone plugin'
        }
    }
}
```

为了让其他项目能用到插件，需要上传插件到 maven 仓库，然后在项目的 build.gradle 的 buildScript 中添加对插件的依赖。

为了上传 maven，在 standalone 的 build.gradle 添加 maven 依赖：

```
apply plugin: 'groovy'
apply plugin: 'maven'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation gradleApi()
    implementation localGroovy()
}

group = 'com.android.standalone'
version = '1.0.0'
uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: uri('../repo'))
        }
    }
}

sourceCompatibility = "8"
targetCompatibility = "8"
```

执行 gradlew uploadArchives 这个 task，会上传一个插件在项目的 repo 目录下。这里上传到的目录 repo 是本地仓库。

在 repo\com\android\standalone\standalone\1.0.0 目录下有生成的 standalone-1.0.0.jar 文件和 md5、sha1、pom 等文件。有了 jar 之后，其他项目就能通过 id 引用插件的 jar 包了。

在需要引用插件的项目的 build.gradle 添加 maven 仓库的地址和插件的路径：

```
buildscript {
    repositories {
        maven {
            url uri('repo')
        }
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.standalone:standalone:1.0.0'
        ...
    }
}
```

然后在 app 模块的 build.gradle 使用插件：

```
apply plugin: 'standalone'
```

执行 gradlew standalone 任务：

```
gradlew standalone
```

输出如下：

```
> Task :app:standalone
apply stand alone plugin
```