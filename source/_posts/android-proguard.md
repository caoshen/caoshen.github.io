# Android Proguard 混淆

Android 项目可以在 build.gradle 开启 proguard 代码混淆。

## 开启混淆的好处

1. 降低代码的可读性，缩短类和成员的名称，使反编译后的代码不容易被其他人阅读或破解。比如 APP \ SDK 对外发布正式版本时，通常需要做代码混淆。
2. 代码压缩。开启混淆后，项目中没有被任何地方执行到的代码会被 Proguard 优化，减少 APP\SDK 包体积。
3. 资源压缩。开启混淆后，项目中没有被使用的图片、字符串、布局等资源会从项目中移出，减少 APP\SDK 包体积。

## 如何开启混淆

Android 项目可以在模块的 build.gradle 文件配置开启混淆。

```
buildTypes {
    debug {
        minifyEnabled false
        shrinkResources false
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
    release {
        minifyEnabled true
        shrinkResources true
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
}
```

minifyEnabled true 表示开启混淆。shrinkResources 表示资源压缩。

需要注意的是 Library 项目无法使用 shrinkResources。shrinkResources 只能用于 APK 资源压缩。这可能是因为工具无法判断 Library 中的资源是否会被其他模块使用到，所以只有在构建整个 APK 的时候才能知道哪些资源是无用资源，才可以启用资源压缩。

## 编写 Proguard 混淆规则

混淆规则有 2 部分组成，Android SDK 自带的默认规则 proguard-android.txt 和项目自定义的 proguard-rules.pro。

### 自定义规则

项目自定义的 proguard-rules.pro。

不混淆四大组件和 Application，在 AndroidManifest.xml 注册的组件：

```
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Application
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider
```

不混淆 ww 目录下的所有类：

```
-keep public class xx.yy.zz.ww.* {*;}
```

不混淆 ww 目录下的所有子目录和类，循环子目录下的所有内容：

```
-keep public class xx.yy.zz.ww.** {*;}
```

不混淆 AA 类的名称，但是混淆 AA 类的成员：

```
-keep public class xx.yy.zz.ww.AA
```

不混淆 AA 类的名称和成员：

```
-keep public class xx.yy.zz.ww.AA {*;}
```

不混淆 AA 类的某个 public 方法，可能被反射调用或者被 Native 代码回调的方法：

```
-keep class xx.yy.zz.ww.AA {
    public void callbackByNavtiveMethodXXXX();
}
```

不混淆 Serializable 子类：

```
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
    public <fields>;
}
```

### 系统规则

系统默认的 proguard-android.txt，位于 $ANDROID_HOME\tools\proguard\proguard-android.txt。在 Android Gradle Plugin (AGP)2.2 以上版本会使用编译期间生成的 proguard-android.txt，比如 build\intermediates\proguard-files\proguard-android.txt-3.3.2，而不是 SDK 里面的 proguard-android.txt。

proguard-android.txt-3.3.2 的规则如下：

```
# This is a configuration file for ProGuard.
# http://proguard.sourceforge.net/index.html#manual/usage.html
#
# Starting with version 2.2 of the Android plugin for Gradle, this file is distributed together with
# the plugin and unpacked at build-time. The files in $ANDROID_HOME are no longer maintained and
# will be ignored by new version of the Android plugin for Gradle.

# Optimization is turned off by default. Dex does not like code run
# through the ProGuard optimize steps (and performs some
# of these optimizations on its own).
# Note that if you want to enable optimization, you cannot just
# include optimization flags in your own project configuration file;
# instead you will need to point to the
# "proguard-android-optimize.txt" file instead of this one from your
# project.properties file.
# 不启用优化
-dontoptimize
# 混淆时不使用大小写混写的类名
-dontusemixedcaseclassnames
# 不跳过库中非 public 的类
-dontskipnonpubliclibraryclasses
# 打印处理日志
-verbose

# Preserve some attributes that may be required for reflection.
# 保留注解、泛型、内部类、封闭方法
-keepattributes *Annotation*,Signature,InnerClasses,EnclosingMethod
# 不混淆指定的类
-keep public class com.google.vending.licensing.ILicensingService
-keep public class com.android.vending.licensing.ILicensingService
-keep public class com.google.android.vending.licensing.ILicensingService
# 不打印潜在的错误或者疏漏的注释
-dontnote com.android.vending.licensing.ILicensingService
-dontnote com.google.vending.licensing.ILicensingService
-dontnote com.google.android.vending.licensing.ILicensingService

# For native methods, see http://proguard.sourceforge.net/manual/examples.html#native
# 不混淆 Native 方法
-keepclasseswithmembernames class * {
    native <methods>;
}

# Keep setters in Views so that animations can still work.
# 不混淆 View 子类的 set 和 get 方法
-keepclassmembers public class * extends android.view.View {
    void set*(***);
    *** get*();
}

# We want to keep methods in Activity that could be used in the XML attribute onClick.
# 不混淆 Activity 子类的参数为 View 的 public 方法
-keepclassmembers class * extends android.app.Activity {
    public void *(android.view.View);
}

# For enumeration classes, see http://proguard.sourceforge.net/manual/examples.html#enumerations
# 不混淆枚举类的 values 和 valueOf 方法
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
# 不混淆 Parceable 子类的 CREATOR 常量
-keepclassmembers class * implements android.os.Parcelable {
    public static final ** CREATOR;
}
# 不混淆 R 类的所有 public static 成员
-keepclassmembers class **.R$* {
    public static <fields>;
}

# Preserve annotated Javascript interface methods.
# 不混淆 JavascriptInterface 注解的方法
-keepclassmembers class * {
    @android.webkit.JavascriptInterface <methods>;
}

# The support libraries contains references to newer platform versions.
# Don't warn about those in case this app is linking against an older
# platform version. We know about them, and they are safe.
# 不要警告 support 和 androidx 的报错
-dontnote android.support.**
-dontnote androidx.**
-dontwarn android.support.**
-dontwarn androidx.**

# This class is deprecated, but remains for backward compatibility.
# 不要警告 FloatMath 的报错
-dontwarn android.util.FloatMath

# Understand the @Keep support annotation.
# 不混淆 @Keep 注解的类、成员
-keep class android.support.annotation.Keep
-keep class androidx.annotation.Keep

-keep @android.support.annotation.Keep class * {*;}
-keep @androidx.annotation.Keep class * {*;}

-keepclasseswithmembers class * {
    @android.support.annotation.Keep <methods>;
}

-keepclasseswithmembers class * {
    @androidx.annotation.Keep <methods>;
}

-keepclasseswithmembers class * {
    @android.support.annotation.Keep <fields>;
}

-keepclasseswithmembers class * {
    @androidx.annotation.Keep <fields>;
}

-keepclasseswithmembers class * {
    @android.support.annotation.Keep <init>(...);
}

-keepclasseswithmembers class * {
    @androidx.annotation.Keep <init>(...);
}

# These classes are duplicated between android.jar and org.apache.http.legacy.jar.
# 不警告 org.apache.http 和 android.net.http
-dontnote org.apache.http.**
-dontnote android.net.http.**

# These classes are duplicated between android.jar and core-lambda-stubs.jar.
# 不警告 java.lang.invoke
-dontnote java.lang.invoke.**
```

## 参考资料

1. [缩减、混淆处理和优化应用](https://developer.android.com/studio/build/shrink-code)
2. [shrinkResources true can't be used on Instant Apps Feature?](https://stackoverflow.com/questions/46219987/shrinkresources-true-cant-be-used-on-instant-apps-feature)
3. [Android ProGuard 代码混淆那些事儿](https://www.jianshu.com/p/69878ed712be)
4. [Android ProGuard 混淆 详解](https://blog.csdn.net/chen930724/article/details/49687067)
5. [Android 混淆从入门到精通](https://juejin.im/entry/6844903444826832904)
