---
title: VirtualAPK 插件化框架介绍
date: 2019-09-21 15:32:55
tags: Android
categories: Android
---

# VirtualAPK 插件化框架介绍

[VirtualAPK](https://github.com/didi/VirtualAPK) 是一个 Android 插件化框架。如果一个 APK 有很多功能，其中一些功能使用的场景比较少，那么可以在这些功能被使用的时候动态加载，而不是一次性打包在整个 APK 中。插件化不仅可以缩小 APK 体积，也方便各个插件特性的动态更新。

使用 VirtualAPK 需要对主 APK （宿主）和插件 APK（插件）做一些修改定制。这里以 VirtualAPK 0.9.8 版本为例。

## 主 APK 配置

在主 APK 的项目 build.gradle 配置 VirtualAPK 的插件如下:

```
buildscript {
    ....
    dependencies {
        classpath 'com.android.tools.build:gradle:3.2.0'
        classpath 'com.didi.virtualapk:gradle:0.9.8.6'
    ...
    }
}
```

这里使用的是 gradle:3.2.0 的插件，如果使用 3.5 版本的 gradle，会出现 android 的 gradle 插件和 VirtualAPK 插件不兼容的情况，具体可以看 VirtualAPK 在 github 的 issue。

如果主 APK 使用了 AndroidX，也建议 android 的 gradle 插件版本在 3.2.0 以上，这样才能将 VirtualAPK 源码中对应 support 包的引用自动转换为 AndroidX。

然后配置主模块 app 的 build.gradle：

配置插件：

```
apply plugin: 'com.didi.virtualapk.host'
```

配置依赖：
```
dependencies {
    ...
    // virtualapk
    implementation 'com.didi.virtualapk:core:0.9.8'
}
```

配置 proguard 混淆

```
-keep class com.didi.virtualapk.internal.VAInstrumentation { *; }
-keep class com.didi.virtualapk.internal.PluginContentResolver { *; }

-dontwarn com.didi.virtualapk.**
-dontwarn android.**
-keep class android.** { *; }
````

最后在 Application 的 attachBaseContext 方法中初始化 VirtualAPK

```java
@Override
protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    PluginManager.getInstance(base).init();
}
```

因为宿主 APK 需要加载插件，所以需要读取外部存储的权限（READ_EXTERNAL_STORAGE）。同时在 AndroidManifest 声明权限如下：

```
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
```

以宿主 APK 的 HostActivity 启动插件 APK 的 MainActivity（com.android.plugin.MainActivity）如下：

```java
public class HostActivity extends Activity {
    private static final int REQ_WRITE_STORAGE = 1000;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            int result = checkSelfPermission(Manifest.permission.READ_EXTERNAL_STORAGE);
            if (result != PackageManager.PERMISSION_GRANTED) {
                requestPermissions(new String[]{Manifest.permission.READ_EXTERNAL_STORAGE}, REQ_WRITE_STORAGE);
            } else {
                launchPlugin();
            }
        } else {
            launchPlugin();
        }
    }

    private void launchPlugin() {
        Log.d("virtualapk:", "launchPlugin");
        String pluginPath = Environment.getExternalStorageDirectory().getAbsolutePath().concat("/Test.apk");
        File plugin = new File(pluginPath);
        try {
            Intent intent = new Intent();
            intent.setClassName("com.android.plugin", "com.android.plugin.MainActivity");
            LoadedPlugin loadedPlugin = PluginManager.getInstance(this).getLoadedPlugin(intent);
            if (loadedPlugin == null) {
                PluginManager.getInstance(this).loadPlugin(plugin);
            }
            startActivity(intent);
        } catch (Exception e) {
            Log.e("virtualapk:", "e:" + e);
            e.printStackTrace();
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == REQ_WRITE_STORAGE) {
            ...
        }
    }
}
```

launchPlugin 首先从外部存储中加载插件 APK，如果插件 APK 已经加载过了，就直接跳转，否则先调用 loadPlugin 方法加载插件。

如果正常运行，宿主 APK 会先启动 HostActivity，然后立即跳转到插件 APK 的 MainActivity，显示插件 APK 的内容。

## 插件 APK 配置

在插件 APK 的项目 build.gradle 配置 VirtualAPK 的插件如下:

```
buildscript {
    ...
    dependencies {
        classpath 'com.android.tools.build:gradle:3.1.0'
        classpath 'com.didi.virtualapk:gradle:0.9.8.6'
    ...
    }
}
```

接着配置 app 模块的 build.gradle：

配置插件：

```
apply plugin: 'com.didi.virtualapk.plugin'
```

可以看出插件 APK 只需要配置 com.didi.virtualapk.plugin，而不用配置依赖，因为插件 APK 这时只是作为被依赖的。

最后需要配置 virtualApk 的定制配置：

```
virtualApk {
    packageId = 0x6f             // The package id of Resources.
    targetHost='C:/Users/m/AndroidStudioProjects/AndroidSample/app' // The path of application module in host project.
    applyHostMapping = true      // [Optional] Default value is true.
}
```

需要注意的是第二项配置，targetHost 表示主 APK 的 app 模块的路径，可以是绝对路径，可以是相对路径。这里用的是绝对路径。

如果插件 APK 的资源和主 APK 的资源名称相同，建议修改为其他名称和主 APK 的命名区分开。

比如 R.layout.activity_main 改名为 R.layout.activity_plugin_main

最后编译插件 APK 如下：

```
gradlew clean assemblePlugin
```

注意插件 APK 必须要先配置好签名，不能用 debug 模式编译。

生成的插件 APK 位于 Plugin\app\build\outputs\plugin\release 目录：

```
com.android.plugin_20190921151552.apk
```

然后将插件 APK 重命名为 Test.apk push 到手机的 /mnt/sdcard/ 目录，方便主 APK 从外部存储加载插件 APK。

## 总结

VirtualAPK 是一个 Android 插件化框架，它支持四大组件以及资源的插件化。同时 VirtualAPK 有它自己的 Gradle 插件，宿主 APK 和 插件 APK 需要用 VirtualAPK 的 Gradle 插件编译。