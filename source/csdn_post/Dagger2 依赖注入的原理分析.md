# Dagger2 依赖注入的原理分析

因为 Dagger2 的使用情况很多，这里只对基本的使用方法进行分析。

以一个简单依赖注入举例如下：


创建 Watch 类如下：

```java
public class Watch {

    public Watch() {

    }

    public String work() {
        return "work";
    }
}
```

创建 WatchModule 类如下：

```java
@Module
public class WatchModule {

    @Provides
    public Watch providesWatch() {
        return new Watch();
    }
}
```

创建 Component 接口如下：

```java
@Component(modules = WatchModule.class)
public interface WatchComponent {
    void inject(WatchActivity activity);
}
```

最后在 WatchActivity 注入依赖如下：

```java
public class WatchActivity extends Activity {
    @Inject
    Watch watch;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        DaggerWatchComponent.create().inject(this);
        Log.d("dagger2", "watch:" + watch.work());
    }
}
```

输出结果如下：

```
D/dagger2: watch:work
```

## Dagger2 辅助类

Rebuild 项目之后，会在 app\build\generated\source\apt\debug 生成 Java 辅助类：

- DaggerWatchComponent： Component 注入类
- WatchActivity_MembersInjector：WatchActivity 成员变量注入类
- WatchModule_ProvidesWatchFactory： Module 注入类

因为 Watch 类的注入是从 DaggerWatchComponent.create().inject(this) 这一行代码开始的，所以首先分析 DaggerWatchComponent 类。

## DaggerWatchComponent 接口实现

DaggerWatchComponent 主要代码如下：

```java
public final class DaggerWatchComponent implements WatchComponent {
  private final WatchModule watchModule;

  private DaggerWatchComponent(WatchModule watchModuleParam) {
    this.watchModule = watchModuleParam;
  }

  public static Builder builder() {
    return new Builder();
  }

  public static WatchComponent create() {
    return new Builder().build();
  }

  @Override
  public void inject(WatchActivity activity) {
    injectWatchActivity(activity);}

  private WatchActivity injectWatchActivity(WatchActivity instance) {
    WatchActivity_MembersInjector.injectWatch(instance, WatchModule_ProvidesWatchFactory.providesWatch(watchModule));
    return instance;
  }

  public static final class Builder {
    private WatchModule watchModule;

    private Builder() {
    }

    public Builder watchModule(WatchModule watchModule) {
      this.watchModule = Preconditions.checkNotNull(watchModule);
      return this;
    }

    public WatchComponent build() {
      if (watchModule == null) {
        this.watchModule = new WatchModule();
      }
      return new DaggerWatchComponent(watchModule);
    }
  }
}
```

DaggerWatchComponent 主要做了 2 件事：

1. 使用建造者模式构造 DaggerWatchComponent
2. 调用构造好的 DaggerWatchComponent 的 WatchModule，将 WatchModule 提供的 Watch 对象注入给 WatchActivity

DaggerWatchComponent 实现了 WatchComponent 接口，实现了接口定义的 inject 方法。

DaggerWatchComponent 的 create 静态方法构造了 watchModule，并且将 watchModule 作为构造参数构造了 DaggerWatchComponent 对象。

DaggerWatchComponent 的 inject 方法调用了 injectWatchActivity 方法，通过 WatchActivity_MembersInjector 的 injectWatch 将 Watch 实例注入到 WatchActivity。

## ProvidesWatchFactory 提供实例对象

WatchModule_ProvidesWatchFactory 是一个工厂类：

```java
public final class WatchModule_ProvidesWatchFactory implements Factory<Watch> {
  private final WatchModule module;

  public WatchModule_ProvidesWatchFactory(WatchModule module) {
    this.module = module;
  }

  @Override
  public Watch get() {
    return providesWatch(module);
  }

  public static WatchModule_ProvidesWatchFactory create(WatchModule module) {
    return new WatchModule_ProvidesWatchFactory(module);
  }

  public static Watch providesWatch(WatchModule instance) {
    return Preconditions.checkNotNull(instance.providesWatch(), "Cannot return null from a non-@Nullable @Provides method");
  }
}
```

providesWatch 方法就是调用了 WatchModule 的 providesWatch 方法。providesWatch 方法里使用 new Watch() 构造 Watch 实例。

## MembersInjector 成员变量注入

WatchActivity_MembersInjector 是一个用来注入成员变量的辅助类：

```java
public final class WatchActivity_MembersInjector implements MembersInjector<WatchActivity> {
  private final Provider<Watch> watchProvider;

  public WatchActivity_MembersInjector(Provider<Watch> watchProvider) {
    this.watchProvider = watchProvider;
  }

  public static MembersInjector<WatchActivity> create(Provider<Watch> watchProvider) {
    return new WatchActivity_MembersInjector(watchProvider);}

  @Override
  public void injectMembers(WatchActivity instance) {
    injectWatch(instance, watchProvider.get());
  }

  public static void injectWatch(WatchActivity instance, Watch watch) {
    instance.watch = watch;
  }
}
```

它的 injectWatch 方法很简单，作用就是将参数 watch 赋值给 WatchActivity 的 watch 变量。所以通过 component 的 inject 就将 module 构造的 watch 实例赋值给了 activity。

## 总结

Dagger2 的 component 作为依赖注入的入口和桥梁，负责初始化 Module Factory 和 Activity MemberInject 辅助类，将 2 者串联起来完成依赖对象注入到目标类的作用。