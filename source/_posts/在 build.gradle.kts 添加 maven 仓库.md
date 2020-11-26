# 在 build.gradle.kts 添加 maven 仓库

使用 kotlin script DSL 配置 build.gradle.kts 时，添加 maven 仓库的方式如下：

```kotlin
repositories {
    maven {
        setUrl("http://maven.aliyun.com/nexus/content/groups/public/")
    }
    mavenCentral()
}
```

它等价于以下 gradle 配置：

```groovy
repositories {
    maven {url 'http://maven.aliyun.com/nexus/content/groups/public/'}
    mavenCentral()
}
```

## 参考资料

[如何使用kotlinscript DSL（build.gradle.kts）通过url添加maven存储库？](https://cloud.tencent.com/developer/ask/142262)

[Provide maven(url: String, name: String? = null, ...) shortcut](https://github.com/gradle/kotlin-dsl-samples/issues/256)
