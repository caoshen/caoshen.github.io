---
title: Gradle Android Transform API 编译修改 class
date: 2020-11-03 10:07:21
tags: Gradle
categories: Android
---

# Gradle Android Transform API 编译修改 class

## Gradle 插件

Android 的 Gradle 插件一般用作 Android 工程的编译构建流程。以 Android app 模块的 build.gradle 为例：

```groovy
apply plugin: 'com.android.application'
```

'com.android.application' 插件是 Android Gradle Plugin(AGP) 提供的用作 app 编译的插件。

Android library 模块也提供了类似的插件：

```groovy
apply plugin: 'com.android.library'
```

'com.android.library' 插件是 Android Gradle Plugin(AGP) 提供的用作 library 编译的插件。

要使用这些插件，必须先在根目录下的 build.gradle 添加编译依赖：

```groovy
buildscript {
    ...
    dependencies {
        classpath 'com.android.tools.build:gradle:4.0.2'
        ...
    }
}
```

如果不清楚如何编写一个 Gradle 插件，可以先看 [Gradle 插件基础](https://blog.csdn.net/caoshen2014/article/details/102528973)，了解如何写一个 Gradle 插件，并且应用到 Android 项目。

## Transform API

通常我们编写 Gradle 插件是为了对原有编译流程做修改，或者修改项目源码中某种类型的类实现做统一的编译期修改。

例如路由自动注册、项目无痕埋点、全局性能监控等使用场景。如果手动对代码的每一处都做修改，不仅工作量大，而且容易遗漏出错。当项目规模增大，涉及的开发人员变多，还会出现信息不同步、修改不一致等问题。

要修改原有编译流程，一般就会在自定义的 Gradle 插件中注册 Transform API。

```kotlin
class DemoPlugin: Plugin<Project> {
    override fun apply(target: Project) {
        target.extensions.findByType(AppExtension::class.java)?.run {
            registerTransform(AutoRegisterTransform(target))
        }
    }
}
```

Transform 是 Android build api 的一部分，它主要用来处理编译的中间过程。每一个 Transform 对应一个 Gradle task，一个 Transform 的输出会变为下一个 Transform 的输入。

```java
package com.android.build.api.transform;

/**
 * A Transform that processes intermediary build artifacts.
 *
 * <p>For each added transform, a new task is created. The action of adding a transform takes
 * care of handling dependencies between the tasks. This is done based on what the transform
 * processes. The output of the transform becomes consumable by other transforms and these
 * tasks get automatically linked together.
 * ...
 */
@SuppressWarnings("MethodMayBeStatic")
public abstract class Transform {
}
```

编写自定义的 Transfrom 时，一般需要重写 getName、getInputTypes、getScopes、isIncremental、transform 5 个方法。

### getName

getName() 用来指明 Transform 的名称。

```kotlin
    override fun getName(): String = "AutoRegisterTransform"
```

### getInputTypes

getInputTypes() 用来指明 Transform 的输入类型。常用的输入类型可以是 Classes 、Resources。 Classes 是 Jar 文件或者目录里面的 class 字节码。 Resources 是标准的 Java 资源文件。

```kotlin
    /**
     * 输入文件的类型
     * 可供我们去处理的有两种类型, 分别是编译后的java代码, 以及资源文件
     */
    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> = TransformManager.CONTENT_CLASS
```

TransformManager 的 getTaskNamePrefix 用来生成 transform task 的前缀。Task 的名称与输入的 InputTypes 以及 Transform 名称相关。

如果 Transform 的名称是 DemoTransform，那么编译过程新增的 task 就有 transformClassesWithDemoTransformForDebug。

```java
    static String getTaskNamePrefix(@NonNull Transform transform) {
        StringBuilder sb = new StringBuilder(100);
        sb.append("transform");

        sb.append(
                transform
                        .getInputTypes()
                        .stream()
                        .map(
                                inputType ->
                                        CaseFormat.UPPER_UNDERSCORE.to(
                                                CaseFormat.UPPER_CAMEL, inputType.name()))
                        .sorted() // Keep the order stable.
                        .collect(Collectors.joining("And")));
        sb.append("With");
        StringHelper.appendCapitalized(sb, transform.getName());
        sb.append("For");

        return sb.toString();
    }
```

### getScopes

getScopes 用来指明 Transform 作用的范围。常用 Transform 的范围有 SCOPE_FULL_PROJECT。SCOPE_FULL_PROJECT 是 PROJECT、SUB_PROJECTS、EXTERNAL_LIBRARIES 的集合，分别表示当前模块，依赖的子模块、外部依赖。

```kotlin
    /**
     * 指定作用范围
     */
    override fun getScopes(): MutableSet<in QualifiedContent.Scope> = TransformManager.SCOPE_FULL_PROJECT
```

### isIncremental

isIncremental() 用来指明是否开启增量编译。开启增量编译后，只会处理发生变更的文件，加快编译速度。

```kotlin
    /**
     * 是否支持增量
     * 如果支持增量执行, 则变化输入内容可能包含 修改/删除/添加 文件的列表
     */
    override fun isIncremental(): Boolean = true
```

### transform

transform(transformInvocation: TransformInvocation) 用来指明编译的具体动作。如果 transform 什么都不做，那么生成的 APK 里面是没有 dex 文件的。因此至少需要将 Transform 的输入复制到输出目录。同时可以遍历扫描每一个 class 文件，针对某一类文件做特殊处理，这种操作就是字节码插桩。

因为 Transform 的基础操作比如文件的复制、删除等都是通用操作，可以提取一个通用的 BaseTransform，并且添加基础操作到 transform 方法。

```kotlin
abstract class DemoTransform : Transform() {
    ...
    /**
     * transform的执行主函数
     */
    override fun transform(transformInvocation: TransformInvocation) {
        val outputProvider = transformInvocation.outputProvider
        println("有没有增量编译${transformInvocation.isIncremental}")
        for (input in transformInvocation.inputs) {
            with(input) {
                // 输入源为jar
                jarInputs.forEach { jarInput ->
                    val inputJar = jarInput.file
                    val outputJar = outputProvider.getContentLocation(
                        jarInput.name,
                        jarInput.contentTypes,
                        jarInput.scopes,
                        Format.JAR
                    )
                    if (transformInvocation.isIncremental) {
                        when (jarInput.status) {
                            NOTCHANGED -> {
                            }
                            ADDED, CHANGED -> transformJar(inputJar, outputJar)
                            REMOVED -> FileUtils.delete(outputJar)
                            else -> {
                            }
                        }
                    } else {
                        transformJar(inputJar, outputJar)
                    }
                }

                // 输入源为文件夹
                directoryInputs.forEach { di ->
                    val inputDir = di.file
                    val outputDir = outputProvider.getContentLocation(
                        di.name,
                        di.contentTypes,
                        di.scopes,
                        Format.DIRECTORY
                    )
                    if (transformInvocation.isIncremental) {
                        for (entry in di.changedFiles.entries) {
                            val inputFile = entry.key
                            when (entry.value) {
                                NOTCHANGED -> {
                                }
                                ADDED, CHANGED -> {
                                    if (!inputFile.isDirectory && inputFile.name.endsWith(
                                            SdkConstants.DOT_CLASS
                                        )
                                    ) {
                                        val out = toOutputFile(outputDir, inputDir, inputFile)
                                        transformFile(inputFile, out)
                                    }
                                }
                                REMOVED -> {
                                    val outputFile = toOutputFile(outputDir, inputDir, inputFile)
                                    FileUtils.deleteIfExists(outputFile)
                                }
                                else -> {
                                }
                            }
                        }
                    } else {
                        FileUtils.getAllFiles(inputDir)
                            .filter {
                                true == it?.name?.endsWith(SdkConstants.DOT_CLASS)
                            }.forEach { fileIn ->
                                val out = toOutputFile(outputDir, inputDir, fileIn)
                                transformFile(fileIn, out)
                            }
                    }

                }
            }
        }
    }
...
}
```

## ASM 修改字节码

修改字节码的方式有很多种，这里介绍使用 [ASM](https://asm.ow2.io/) 修改字节码的方式。相比其他方式，ASM 修改效率很高。

在 build.gradle 引入 ASM 依赖：

```groovy
    implementation 'org.ow2.asm:asm:6.0'
```

因为字节码的可读性较低，如果不熟悉 ASM API 和字节码的生成方式，可以先用 Java 编写想要转换的代码，然后使用一个 IDEA 插件 [ASM Bytecode Outline](https://plugins.jetbrains.com/plugin/5918-asm-bytecode-outline)，转换 Java 代码到 ASM 代码。

以如下 Java 代码为例：

```java
    public static List<String> getAllRoutes(){
        List<String> list = new ArrayList<String>();
        return list;
    }
```

以 ASM Bytecode Outline 插件转换的 ASMified 代码如下：

```java
        {
            mv = cw.visitMethod(ACC_PUBLIC + ACC_STATIC, "getAllRoutes", "()Ljava/util/List;", "()Ljava/util/List<Ljava/lang/String;>;", null);
            mv.visitCode();
            Label l0 = new Label();
            mv.visitLabel(l0);
            mv.visitLineNumber(10, l0);
            mv.visitTypeInsn(NEW, "java/util/ArrayList");
            mv.visitInsn(DUP);
            mv.visitMethodInsn(INVOKESPECIAL, "java/util/ArrayList", "<init>", "()V", false);
            mv.visitVarInsn(ASTORE, 0);
            Label l1 = new Label();
            mv.visitLabel(l1);
            mv.visitLineNumber(11, l1);
            mv.visitVarInsn(ALOAD, 0);
            mv.visitInsn(ARETURN);
            Label l2 = new Label();
            mv.visitLabel(l2);
            mv.visitLocalVariable("list", "Ljava/util/List;", "Ljava/util/List<Ljava/lang/String;>;", l1, l2, 0);
            mv.visitMaxs(2, 1);
            mv.visitEnd();
        }
```

当我们需要生成 Java 代码对应的 .class 字节码时，使用 ClassWriter 写入 ASMified 得到的代码，最后将 ClassWriter 转换成字节数组，并作为输入流写入文件即可。

```kotlin
    /**
     * 修改注册类
     */
    private fun modifyRegisterByte(ins: InputStream): ByteArray {
        val cw = ClassWriter(0)
        val cr = ClassReader(ins)
        val cn = ClassNode()
        cr.accept(cn, 0)
        cn.methods.removeIf { it.name == "getAllRoutes" && "()Ljava/util/List;" == it.desc }
        val mv = cn.visitMethod(
            ACC_PUBLIC + ACC_STATIC,
            "getAllRoutes",
            "()Ljava/util/List;",
            "()Ljava/util/List<Ljava/lang/String;>;",
            null
        )
        with(mv) {
            val labels = arrayListOf<Label>()
            visitCode()

            val label = Label()
            visitLabel(label)
            visitTypeInsn(NEW, "java/util/ArrayList")
            visitInsn(DUP)
            visitMethodInsn(INVOKESPECIAL, "java/util/ArrayList", "<init>", "()V", false)
            visitVarInsn(ASTORE, 0)

            ...
        }
        cn.accept(cw)
        return cw.toByteArray()
    }
```

修改目标 class 文件：

```kotlin
    private fun modifyTargetClass(it: File) {
        // 如果找到的目标文件是 .class 文件（对 ModuleRegister 来说，它是 .class 文件）
        if (it.name.endsWith(SdkConstants.DOT_CLASS)) {
            val rewrite = modifyRegisterByte((it.inputStream()))
            FileOutputStream(it).write(rewrite)
            ...
        }
    }
```

## class 文件读写

对 Transform API 而言，修改字节码有两种类型：

1. 直接修改文件夹的 .class 文件。对应 Transform 的 directoryInputs
2. 修改 .jar 文件里面的 .class 文件。对应 Transform 的 jarInputs

### 修改文件夹的 .class 字节码

如果修改 .class 文件，可以使用 FileInputStream 和 FileOutputStream。首先读取文件，然后将 ASM 转换的 ByteArray 写入到文件。

```kotlin
class AutoRegisterTransform(val p: Project): DemoTransform() {

    private fun modifyTargetClass(it: File) {
        // 如果找到的目标文件是 .class 文件
        if (it.name.endsWith(SdkConstants.DOT_CLASS)) {
            val rewrite = modifyRegisterByte((it.inputStream()))
            FileOutputStream(it).write(rewrite)
        } else if (it.name.endsWith(SdkConstants.DOT_JAR)) {
            // 如果找到的目标文件是 .jar 文件。dependencies {}、依赖的 android library 类型 module、
            // 每个 module 的 R 文件打包成的 jar，都属于 .jar 文件
            ...   
        }
    }
}
```

### 修改 Jar 文件的 .class 字节码

如果修改的是 .jar 文件里面的 .class 文件。需要遍历 .jar 文件的每一个 JarEntry，如果发现目标的 .class 文件，就使用 putNextEntry 修改它的内容。

dealJarFile 用来处理 jar 文件。将参数的 JarFile 作为输出文件，将 JarFile 重命名为 bakJarFile 作为输入文件，遍历 bakJarFile 的每一个 JarEntry。如果发现 JarEntry 的名称和需要替换的类名相同，就使用 ASM 修改它的字节码，然后加以一个新的 JarEntry。否则将原有的 JarEntry 复制到 JarFile。

为了修改 JarEntry，可以提取一个 addZipEntry 方法。当需要修改字节码时，将 ASM 转换得到的 ByteArray 转换成 ByteArrayInputStream 作为 InputStream 输入；当不需要修改字节码，只复制 JarEntry 时，将 bakJarFile.getInputStream(jarEntry) 作为 InputStream 输入。


目标文件是 Jar 文件，而且发现 JarEntry 的名称和需要替换的类名相同：

```kotlin
class AutoRegisterTransform(val p: Project): DemoTransform() {

    private fun modifyTargetClass(it: File) {
        // 如果找到的目标文件是 .class 文件
        if (it.name.endsWith(SdkConstants.DOT_CLASS)) {
            ...
        } else if (it.name.endsWith(SdkConstants.DOT_JAR)) {
            // 如果找到的目标文件是 .jar 文件。dependencies {}、依赖的 android library 类型 module、
            // 每个 module 的 R 文件打包成的 jar，都属于 .jar 文件
            ScanHelper.dealJarFile(it) { jarEntry, jos, jarFile ->
                val isRegisterClass = jarEntry.name == ScanConstants.AUTOREGISTER
                if (isRegisterClass) {
                    println("modifyTargetClass: isRegisterClass:${isRegisterClass}")

                    val inputStream = jarFile.getInputStream(jarEntry)
                    val byteArray = modifyRegisterByte(inputStream)
//                    jos.putNextEntry(JarEntry(jarEntry.name))
//                    jos.write(byteArray)
                    ScanHelper.addZipEntry(jos, JarEntry(jarEntry.name), ByteArrayInputStream(
                        byteArray
                    ))
                } else {
                    println("NOT register class:${jarEntry.name}")
                }
                isRegisterClass
            }
        }
    }
}
```

目标文件是 Jar 文件，而且发现 JarEntry 的名称和需要替换的类名不同：

```kotlin
object ScanHelper {

    fun dealJarFile(
        jarFile: File,
        callback: (jarEntry: JarEntry, jos: JarOutputStream, jarFile: JarFile) -> Boolean
    ) {
        val jarAbsolutePath = jarFile.absolutePath
        val bakFilePath = jarAbsolutePath.substring(0, jarAbsolutePath.length - 4) + "-" +
                System.currentTimeMillis() + SdkConstants.DOT_JAR
        println("jar absolute path: ${jarAbsolutePath}, bak file path:${bakFilePath}")
        val bakFile = File(bakFilePath)
        // JarFile 不是 File 的子类，需要将原始 JarFile(build/javac)，重命名为带时间戳的新 jar。
        // 原有的 jarFile 在磁盘上不再存在
        jarFile.renameTo(bakFile)
        // 获取备份的jar
        val bakJarFile = JarFile(bakFilePath)
        val jos = JarOutputStream(FileOutputStream(jarFile))
        for (jarEntry in bakJarFile.entries()) {
            println("name:${jarEntry.name}, size:${jarEntry.size}")
            // 只有找到了目标 class 的情况下才在 callback 方法添加 ZipEntry
            // 否则在这里的 JarOutputStream 添加 ZipEntry
            if (!callback(jarEntry, jos, bakJarFile)) {
                println("putNextEntry:${jarEntry.name}")

                val inputStream = bakJarFile.getInputStream(jarEntry)

                val newZipEntry = ZipEntry(jarEntry.name)

                addZipEntry(jos, newZipEntry, inputStream)
            }
        }

        with(jos) {
            flush()
            finish()
            close()
        }
        // 关闭 jarFile
        bakJarFile.close()
        // 删除备份文件
        bakFile.delete()
    }

    fun addZipEntry(
        jos: JarOutputStream,
        zipEntry: ZipEntry,
        inputStream: InputStream
    ) {
        jos.putNextEntry(zipEntry)
        val buffer = ByteArray(16384)
        var length: Int
        do {
            length = inputStream.read(buffer);
            if (length == -1) {
                break
            }
            jos.write(buffer, 0, length)
            jos.flush()
        } while (length != -1);
        inputStream.close()
        jos.closeEntry()
    }
}
```

## 参考资料

[每日一问 | 玩转 Gradle，可不能不熟悉 Transform，那么，我要开始问了。](https://www.wanandroid.com/wenda/show/15215)

[深入了解TransformApi](https://yutiantina.github.io/2019/04/24/%E6%B7%B1%E5%85%A5%E4%BA%86%E8%A7%A3TransformApi/)

[TransformDemo](https://github.com/YuTianTina/TransformDemo)

[Chapter-ASM](https://github.com/AndroidAdvanceWithGeektime/Chapter-ASM)
