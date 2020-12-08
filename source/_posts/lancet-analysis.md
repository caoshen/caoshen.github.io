---
title: Android AOP 框架 Lancet 应用与解析
date: 2020-12-08 21:12:00
tags: Gradle
categories: Android
---

# Android AOP 框架 Lancet 应用与解析

Lancet 是一个轻量级 Android AOP 框架。它可以用来替换某个方法的代码实现，或者在方法执行前后插入代码。

## Lancet 应用举例

待修改的 MainActivity 如下：

```java
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // onCreatelancet
        Log.i(TAG, "onCreate");
    }

}
```

假设需要在 MainActivity 的每一个 Log 日志之前都加上 "lancet" 后缀，可以使用 Lancet 的 @Proxy 注解。

```java
public class HookClass {
    @Proxy("i")
    @TargetClass("android.util.Log")
    public static int anyName(String tag, String msg){
        msg = msg + "lancet";
        return (int) Origin.call();
    }
}
```

输出如下：

```
MainActivity: onCreatelancet
```

假设需要在 MainActivity 的 onStop 方法执行之前打印日志到标准输出，可以使用 Lancet 的 @Insert 注解。

```java
public class HookClass {

    @TargetClass(value = "androidx.appcompat.app.AppCompatActivity", scope = Scope.LEAF)
    @Insert(value = "onStop", mayCreateSuper = true)
    protected void onStop() {
        System.out.println("hello onStop");
        Origin.callVoid();
    }

}
```

当 MainActivity 触发 onStop 时，输出如下：

```
System.out: hello onStop
```

## Lancet 注入

使用 jadx-gui 工具反编译 apk，可以看出 MainActivity 注入了一个 _lancet 子类。

```java
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";

    class _lancet {
        private _lancet() {
        }

        @Proxy("i")
        @TargetClass("android.util.Log")
        static int cn_okclouder_lancet_use_HookClass_anyName(String str, String str2) {
            return Log.i(str, str2 + "lancet");
        }

        @TargetClass(scope = Scope.LEAF, value = "androidx.appcompat.app.AppCompatActivity")
        @Insert(mayCreateSuper = true, value = "onStop")
        static void cn_okclouder_lancet_use_HookClass_onStop(MainActivity mainActivity) {
            System.out.println("hello onStop");
            mainActivity.onStop$___twin___();
        }
    }

    /* access modifiers changed from: private */
    /* access modifiers changed from: public */
    private void onStop$___twin___() {
        super.onStop();
    }

    /* access modifiers changed from: protected */
    @Override // androidx.appcompat.app.AppCompatActivity, androidx.fragment.app.FragmentActivity
    public void onStop() {
        _lancet.cn_okclouder_lancet_use_HookClass_onStop(this);
    }

    /* access modifiers changed from: protected */
    @Override // androidx.core.app.ComponentActivity, androidx.appcompat.app.AppCompatActivity, androidx.fragment.app.FragmentActivity
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        _lancet.cn_okclouder_lancet_use_HookClass_anyName(TAG, "onCreate");
    }
}
```

_lancet 子类按照 HookClass 中定义的 anyName 和 onStop 方法生成了 2 个静态方法。

anyName 方法直接调用了 Log.i，只是改变了方法的第二个参数，即 message 参数，为 message 参数加上了 "lancet" 后缀。

onStop 方法首先输出 HookClass 中定义的 system.out，然后调用 MainActivity onStop 方法的 twin 方法。twin 方法是 MainActivity 默认的 onStop 方法。因为 @Insert 注解会改变原有的 onStop，直接在 _lancet 子类的 onStop 方法无法调用 MainActivity 的 super.onStop()，调用 MainActivity 的 onStop 会导致循环调用，因此使用了 twin 方法间接调用默认 onStop。

在 MainActivity 原有的调用 Log.i 和 onStop 方法的地方使用 _lancet 子类的静态方法替换，从而实现了 HookClass 定义的代码替换。
