# Android 源码的装饰者模式

## 装饰者模式介绍

装饰者模式（Decorator Pattern）也称为包装模式 （Wrapper Pattern），结构型设计模式之一，其使用一种对客户端透明的方式来动态地扩展对象地功能，同时它也是继承关系的一种替代方案之一。

## 装饰者模式的定义

动态地给一个对象添加一些额外的职责。就增加功能来说，装饰者模式相比生成子类更为灵活。

## 装饰模式的使用场景

需要透明且动态地扩展类的功能时使用。

## Android 源码中的装饰模式

Context 类在 Android 中被称为“上帝对象”，它本质是一个抽象类，其在我们装饰者模式里相当于抽象组件，而在其内部定义了大量的抽象方法，比如我们经常会用到的 startActivity 方法。

```java
    ...
    public abstract void startActivity(@RequiresPermission Intent intent);
    ...
    public abstract void startActivity(@RequiresPermission Intent intent,
            @Nullable Bundle options);
    ...
```

真正的实现是在 ContextImpl 中完成的，ContextImpl 继承自 Context 抽象类，并实现了 Context 中的抽象方法。

```java
    @Override
    public void startActivity(Intent intent) {
        warnIfCallingFromSystemProcess();
        startActivity(intent, null);
    }

    /** @hide */
    @Override
    public void startActivityAsUser(Intent intent, UserHandle user) {
        startActivityAsUser(intent, null, user);
    }

    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
        // maintain this for backwards compatibility.
        final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && (targetSdkVersion < Build.VERSION_CODES.N
                        || targetSdkVersion >= Build.VERSION_CODES.P)
                && (options == null
                        || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                            + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                            + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
```

这里 ContextImpl 就相当于组件具体实现类，那么谁来承担装饰者的身份呢？

Activity 从类层次上来说本质是一个 Context。Activity 并非直接继承于 Context，而是继承于 ContextThemeWrapper。

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback,
        AutofillManager.AutofillClient {
...
}
```

而这个 ContextThemeWrapper 又是继承于 ContextWrapper。

```java
public class ContextThemeWrapper extends ContextWrapper {
...
}
```

最终这个 ContextWrapper 才继承于 Context。

为什么类层次会这么复杂呢？其实这里就是一个典型的装饰模式，ContextWrapper 就是我们要找的装饰者，在 ContextWrapper 中有一个 Context 的引用。

```java
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    /**
     * Set the base context for this ContextWrapper.  All calls will then be
     * delegated to the base context.  Throws
     * IllegalStateException if a base context has already been set.
     * 
     * @param base The new base context for this wrapper.
     */
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    ...
```

ContextWrapper 类的 startActivity 方法如下：

```java
    @Override
    public void startActivity(Intent intent) {
        mBase.startActivity(intent);
    }
```

可以看出它调用了 ContextImpl 中对应的方法。

装饰模式应用的套路都是很相似的，对于具体方法的包装扩展则由 ContextWrapper 的具体子类完成，比如我们的 Activity、Service 和Application。