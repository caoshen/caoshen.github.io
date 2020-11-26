---
title: Android Gradle 版本参数优化
date: 2020-10-28 13:02:33
tags: Gradle
categories: Android
---

# Android Gradle 版本参数优化

在 Gradle 项目结构中，每一个 Module 都对应一个 build.gradle。有时每个 Module 都会需要配置相同的版本号或者相同的版本依赖。为了解决相同参数重复配置的问题，可以在项目的根目录下增加一个公用的配置文件 common_config.gradle，在公用配置文件提供 Android app 模块、Android library 模块、java library 模块的公用配置。

## common_config.gradle 

每一个模块的 build.gradle 都对应一个 project 对象，可以将 project 传递给 common_config 定义的 setAppDefaultConfig 闭包，从而实现参数配置。common_config.gradle 可以根据具体情况修改，如果项目不使用 kotlin，可以在 common_config 去掉 kotlin 依赖。

common_config.gradle 如下：

```gradle
project.ext {

    versions = [
            "compileSdkVersion": 29,
            "buildToolsVersion": "29.0.3",

            "minSdkVersion"    : 19,
            "targetSdkVersion" : 29,
            "versionCode"      : 1,
            "versionName"      : "1.0.0",

            "junit": "4.13",

            "kotlin": "1.3.72"
    ]

    dependencieLibs = [
            "appcompat"         : "androidx.appcompat:appcompat:1.1.0",

            // kotlin
            "kotlin-stdlib-jdk7": "org.jetbrains.kotlin:kotlin-stdlib-jdk7:${versions["kotlin"]}",
            "kotlin-reflect"    : "org.jetbrains.kotlin:kotlin-reflect:${versions["kotlin"]}",

            // test
            "junit"             : "junit:junit:${versions["junit"]}"
    ]

    setAppDefaultConfig = { extension ->
        extension.apply plugin: 'com.android.application'
        extension.apply plugin: 'kotlin-android'
        extension.description 'app'

        setAndroidConfig extension.android

        setDependencies extension.dependencies
    }

    setLibDefaultConfig = { extension ->
        extension.apply plugin: 'com.android.library'
        extension.apply plugin: 'kotlin-android'
        extension.description 'lib'

        setAndroidConfig extension.android

        setDependencies extension.dependencies
    }

    setJavaLibDefaultConfig = { extension ->
        extension.apply plugin: 'java-library'
        extension.apply plugin: 'kotlin'
        extension.description 'javalib'

        setDependencies extension.dependencies

        extension.sourceCompatibility = JavaVersion.VERSION_1_8
        extension.targetCompatibility = JavaVersion.VERSION_1_8
    }

    setAndroidConfig = { extension ->
        extension.compileSdkVersion versions['compileSdkVersion']
        extension.buildToolsVersion versions['buildToolsVersion']

        extension.defaultConfig {
            minSdkVersion versions['minSdkVersion']
            targetSdkVersion versions['targetSdkVersion']
            versionCode versions['versionCode']
            versionName versions['versionName']

            testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        }

        extension.compileOptions {
            targetCompatibility = JavaVersion.VERSION_1_8
            sourceCompatibility = JavaVersion.VERSION_1_8
        }

        extension.kotlinOptions {
            jvmTarget = JavaVersion.VERSION_1_8
        }
    }

    setDependencies = { extension ->
        extension.implementation fileTree(include: ['*.jar'], dir: 'libs')
        extension.implementation dependencieLibs['kotlin-stdlib-jdk7']
        extension.implementation dependencieLibs['appcompat']
        extension.testImplementation dependencieLibs['junit']
    }
}

```

## 配置说明

### versions

版本号定义

### dependencieLibs

所有模块都用到的公有依赖

### setAppDefaultConfig

Android application 模块的配置闭包

### setLibDefaultConfig

Android library 模块的配置闭包

### setJavaLibDefaultConfig

Java library 模块的配置闭包

### setAndroidConfig

android 的 配置闭包，也就是 build.gradle 的 android {} 配置

### setDependencies

依赖的配置闭包，也就是 dependencies {} 配置

## 专有配置

如果某个模块除了公有配置之外，还有它自己所需的依赖，可以在 setAppDefaultConfig 之后添加专有的 dependencies {} 依赖。

app 模块的 build.gradle 如下：

```gradle
apply from: "${rootProject.rootDir}/common_config.gradle"

project.ext.setAppDefaultConfig project

dependencies {
    implementation 'androidx.recyclerview:recyclerview:1.1.0'
}
```