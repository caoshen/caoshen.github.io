---
title: Dagger2 依赖注入框架介绍
date: 2019-09-18 23:53:34
tags: 依赖注入
categories: Android
---

# Dagger2 依赖注入框架介绍

Dagger2 是一个标准的依赖注入框架。Dagger2 是 Dagger1（Square 公司开发）的分支，由谷歌接手开发。

## 添加依赖

这里以 dagger 2.24 版本为例，在 app 的 build.gradle 添加 dagger 依赖如下:

```
// Add Dagger dependencies
dependencies {
  api 'com.google.dagger:dagger:2.24'
  annotationProcessor 'com.google.dagger:dagger-compiler:2.24'
}
```

## @Inject 和 @Component

假设由一个 Watch 类，需要调用 Watch 类的 work 方法。

```java
public class Watch {

    @Inject
    public Watch() {

    }

    public void work() {
        Log.d("dagger2", "work");
    }
}
```

为了注入 Watch 依赖，需要给它的构造方法添加 @Inject 注解，表明 dagger 使用 Watch 的构造方法构建对象。

接下来使用 @Component 注解来完成依赖注入。

定义 MainDaggerActivityComponent 接口如下，表示需要将依赖注入给 MainDaggerActivity。

定义的接口通常为类名 + Component 的形式，dagger 会自动生成 Dagger + 类名 + Component 形式的辅助类。

```java
@Component
public interface MainDaggerActivityComponent {
    void inject(MainDaggerActivity activity);
}
```

最后在 MainDaggerActivity 调用编译生成的文件 DaggerMainDaggerActivityComponent，调用它的 inject 方法完成依赖注入。

```java
public class MainDaggerActivity extends Activity {

    @Inject
    Watch watch;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dagger);
        DaggerMainDaggerActivityComponent.create().inject(this);
        watch.work();
    }
}
```

以上代码会调用 watch 对象的 work 方法。输出如下：

```
D/dagger2: work
```

## @Module 和 @Provides

如果在项目中使用了第三方开源框架，比如 Gson，那么不能直接对 Gson 使用 @Inject 注解。因为 Gson 没有一个被 @Inject 注解的构造方法。

具体报错如下：

```
错误: [Dagger/MissingBinding] com.google.gson.Gson cannot be provided without an @Inject constructor or an @Provides-annotated method.
```

所以需要一个工厂类来提供生成 Gson 对象的方法。

```java
@Module
public class GsonModule {

    @Provides
    public Gson providesGson() {
        return new Gson();
    }
}
```

@Module 标注的类是一个工厂类。@Provides 标记的方法是一个工厂方法，用来生成各种类的对象。

在 Component 接口定义中新增 modules 如下：

```java
@Component(modules = GsonModule.class)
public interface MainDaggerActivityComponent {
    void inject(MainDaggerActivity activity);
}
```

以上代码说明 inject 注入时，可以使用 GsonModule 来提供注入对象。

最后在 MainDaggerActivity 注入 Gson 依赖：

```java
public class MainDaggerActivity extends Activity {

    @Inject
    Watch watch;

    @Inject
    Gson gson;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dagger);
        DaggerMainDaggerActivityComponent.create().inject(this);
        String jsonD = "{'name': 'zhangwuji', 'age': 20}";
        Man man = gson.fromJson(jsonD, Man.class);
        Log.d("dagger2", "man:" + man);
    }

    class Man {
        String name;
        int age;

        public Man(String name, int age) {
            this.name = name;
            this.age = age;
        }

        @Override
        public String toString() {
            return "Man{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    '}';
        }
    }
}
```

以上代码输出如下：

```
D/dagger2: man:Man{name='zhangwuji', age=20}
```

类似的，如果需要注入的对象是抽象类，@Inject 注解也无法被使用，因为抽象类不能被实例化，如下所示：

```java
public abstract class Engine {
    public abstract String work();
}
```

以上代码定义了 Engine 抽象类，接着定义它的实现类 GasolineEngine:

```java
public class GasolineEngine extends Engine {

    @Inject
    public GasolineEngine() {

    }

    @Override
    public String work() {
        return "GasolineEngine: work";
    }
}
```

最后在 Car 类引用 Engine：

```java
public class Car {

    private final Engine engine;

    @Inject
    public Car(Engine engine) {
        this.engine = engine;
    }

    public String run() {
        return engine.work();
    }
}
```

如果在 MainDaggerActivity 注入 Car 对象，编译会报错：

```java
    @Inject
    Car car;
```

具体报错如下：

```
错误: [Dagger/MissingBinding] com.caoshen.androidsample.dagger.Engine cannot be provided without an @Provides-annotated method.
```

因为 Car 的构造方法传入的是抽象类 Engine，Engine 类没有构造方法。

这时可以使用 @Module 和 @Provides 注解提供工厂方法。

编写 GasolineEngine 实现类：

```java
public class GasolineEngine extends Engine {
    @Override
    public String work() {
        return "GasolineEngine: work";
    }
}
```

编写 EngineModule 类：

```java
@Module
public class EngineModule {
    @Provides
    public Engine providesEngine() {
        return new GasolineEngine();
    }
}
```

编写 MainDaggerActivityComponent 类：

```java
@Component(modules = {GsonModule.class, EngineModule.class})
public interface MainDaggerActivityComponent {
    void inject(MainDaggerActivity activity);
}
```

注入 Car 依赖：

```java
public class MainDaggerActivity extends Activity {
...
    @Inject
    Car car;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dagger);
        DaggerMainDaggerActivityComponent.create().inject(this);
        Log.d("dagger2", "car:" + car.run());
    }
```

输出如下：

```
D/dagger2: car:GasolineEngine: work
```

## @Named 和 @Qualifier

@Qualifier 是限定符，@Named 则是 @Qualifier 的一种实现。当有两个相同的依赖时，它们都继承同一个父类或者均实现同一个接口。当它们被提供给 Component 时，就会不知道要提供哪一个依赖对象。

假设提供了新的 Engine：DieselEngine。

```java
public class DieselEngine extends Engine {
    @Override
    public String work() {
        return "DieselEngine: work";
    }
}
```

在 EngineModule 提供如下 2 个方法：

```java
@Module
public class EngineModule {
    @Provides
    public Engine providesGasoline() {
        return new GasolineEngine();
    }

    @Provides
    public Engine providesDiesel() {
        return new DieselEngine();
    }
}
```

编译会报错：

```
错误: [Dagger/DuplicateBindings] com.caoshen.androidsample.dagger.Engine is bound multiple times:
@Provides com.caoshen.androidsample.dagger.Engine com.caoshen.androidsample.dagger.EngineModule.providesDiesel()
@Provides com.caoshen.androidsample.dagger.Engine com.caoshen.androidsample.dagger.EngineModule.providesGasoline()
```

编译错误表明 Engine 可能有多个实现，不知道用哪个构造方法。

这时可以使用 @Named 注解，如下所示：

```java
@Module
public class EngineModule {
    @Provides
    @Named("Gasoline")
    public Engine providesGasoline() {
        return new GasolineEngine();
    }

    @Provides
    @Named("Diesel")
    public Engine providesDiesel() {
        return new DieselEngine();
    }
}
```

然后在 Car 类指定依赖：

```java
public class Car {

    private final Engine engine;

    @Inject
    public Car(@Named("Diesel") Engine engine) {
        this.engine = engine;
    }

    public String run() {
        return engine.work();
    }
}
```

输出结果如下：

```
D/dagger2: car:DieselEngine: work
```

以上的代码通过 @Named 来指定采用 provides 的方式来生成实例。也可以用 @Qualifier 来实现，@Qualifier 用来定义注解。


定义 Gasoline 注解：

```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
public @interface Gasoline {
}
```

定义 Diesel 注解：

```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
public @interface Diesel {
}
```

定义 EngineModule：

```java
@Module
public class EngineModule {
    @Provides
    @Gasoline
    public Engine providesGasoline() {
        return new GasolineEngine();
    }

    @Provides
    @Diesel
    public Engine providesDiesel() {
        return new DieselEngine();
    }
}
```

注入 Diesel 依赖：

```java
public class Car {

    private final Engine engine;

    @Inject
    public Car(@Diesel Engine engine) {
        this.engine = engine;
    }

    public String run() {
        return engine.work();
    }
}

```

输出如下：

```java
D/dagger2: car:DieselEngine: work
```

## @Singleton 和 @Scope

@Scope 是用来自定义注解的，而 @Singleton 则是用来配合实现局部单例和全局单例的。需要注意的是，@Singleton 本身并不具备创建单例的能力。

如果我们要使用两次 Gson，会这么做：

```java
    @Inject
    Gson gson1;

    @Inject
    Gson gson2;
```

由于 gson1 和 gson2 的内存地址不一样，所以是 2 个 Gson 对象。如果想让 Gson 是一个单例，可以在 GsonModule 中添加 @Singleton 标签。如下所示：

```java
@Module
public class GsonModule {

    @Singleton
    @Provides
    public Gson providesGson() {
        return new Gson();
    }
}
```

然后在 Component 接口中添加 @Singleton：

```java
@Singleton
@Component(modules = {GsonModule.class})
public interface MainDaggerActivityComponent {
    void inject(MainDaggerActivity activity);
}
```

在 MainDaggerActivity 打印日志如下：

```java
    @Inject
    Gson gson1;

    @Inject
    Gson gson2;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dagger);
        DaggerMainDaggerActivityComponent.create().inject(this);
        Log.d("dagger2", gson1.equals(gson2) ? "same" : "different");
    }
```

输出如下：

```
D/dagger2: same
```

可以看出 gson1 和 gson2 是同一个对象。

但是 gson1 和 gson2 只是在 MainDaggerActivity 相同，如果再创建一个 Activity2，Activity2 里面的 gson 就和 gson1、gson2 不同，也就是说 gson1 和 gson2 只能保证局部单例。

如果想实现全局单例，就需要保证对应的 Component 只有一个实例。

查看 @Singleton 注解的代码：

```java
@Scope
@Documented
@Retention(RUNTIME)
public @interface Singleton {}
```

可以看出 @Singleton 注解是被 @Scope 注解的。

为了实现 Gson 的全局单例，可以使用 @Scope 配合 Application。

首先定义 @ApplicationScope 注解:

```java
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface ApplicationScope {
    
}
```

接着再 GsonModule 中使用 @ApplicationScope 注解：

```java
@Module
public class GsonModule {

    @ApplicationScope
    @Provides
    public Gson providesGson() {
        return new Gson();
    }
}
```

在 Component 中添加注解如下：

```java
@ApplicationScope
@Component(modules = {GsonModule.class})
public interface MainDaggerActivityComponent {
    void inject(MainDaggerActivity activity);
    void inject(MainDaggerActivity2 activity);
}
```

自定义 Application 继承 Application：

```java
public class SampleApplication extends Application {

    private MainDaggerActivityComponent activityComponent;

    @Override
    public void onCreate() {
        super.onCreate();
        activityComponent = DaggerMainDaggerActivityComponent.builder().build();
    }

    public static SampleApplication get(Context context) {
        return (SampleApplication) context.getApplicationContext();
    }

    public MainDaggerActivityComponent getActivityComponent() {
        return activityComponent;
    }
}
```

由于 Application 是单例的，所以保证了 MainDaggerActivityComponent 也是单例。

用同一个 MainDaggerActivityComponent 在 2 个 Activity inject 依赖如下：

```java
public class MainDaggerActivity extends Activity {

    @Inject
    Gson gson1;

    @Inject
    Gson gson2;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dagger);
        SampleApplication.get(this).getActivityComponent().inject(this);
        Log.d("dagger2", "gson1: hashcode:" + gson1.hashCode());

        launch2();
    }

    private void launch2() {
        startActivity(new Intent(this, MainDaggerActivity2.class));
    }

}
```

```java
public class MainDaggerActivity2 extends Activity {

    @Inject
    Gson gson1;

    @Inject
    Gson gson2;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dagger);
        SampleApplication.get(this).getActivityComponent().inject(this);
        Log.d("dagger2", "gson2: hashcode:" + gson2.hashCode());
    }

}
```

日志打印可以发现 activity1 的 gson1 和 activity2 的 gson2 是同一个：

```
2019-09-18 21:05:19.263 9575-9575/com.caoshen.androidsample D/dagger2: gson1: hashcode:133084731
2019-09-18 21:05:19.868 9575-9575/com.caoshen.androidsample D/dagger2: gson2: hashcode:133084731
```

通常来说一个应用会有很多 Component，为了管理这些 Component，可以使用自定义 Scope 注解。他可以更好地管理 Component 和 Module 之间的匹配关系。比如编译器会检查 Component 管理的 Module。如果发现 Module 创建实例的 Scope 注解和 Component 的 Scope 注解不一致就会编译报错。


## @Component 的 dependencies

@Component 也可以用 dependencies 依赖其他 component。

定义 Swordman 类如下：

```java
public class Swordman {
    @Inject
    public Swordman() {

    }

    public String fight() {
        return "fighting";
    }
}
```

定义 Module 如下：

```java
@Module
public class SwordmanModule {

    @Provides
    public Swordman providesSwordman() {
        return new Swordman();
    }
}
```

定义 Component 如下：

```java
@Component(modules = SwordmanModule.class)
public interface SwordmanComponent {
    Swordman getSwordman();
}
```

在 MainDaggerActivityComponent 添加 Component 依赖：

```java
@ApplicationScope
@Component(modules = {GsonModule.class}, dependencies = SwordmanComponent.class)
public interface MainDaggerActivityComponent {
    void inject(MainDaggerActivity activity);
    void inject(MainDaggerActivity2 activity);
}
```

然后在 application 中引入依赖的 Module

```java
    @Override
    public void onCreate() {
        super.onCreate();
        activityComponent = DaggerMainDaggerActivityComponent.builder()
                .swordmanComponent(DaggerSwordmanComponent.builder().build())
                .build();
    }
```

可以看出 DaggerMainDaggerActivityComponent 构造时多了 DaggerSwordmanComponent 的构造过程。

最后在 activity 注入 Swordman 依赖：

```java
public class MainDaggerActivity extends Activity {

    @Inject
    Gson gson1;

    @Inject
    Gson gson2;

    @Inject
    Swordman swordman;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dagger);
        SampleApplication.get(this).getActivityComponent().inject(this);
        Log.d("dagger2", "swordman:" + swordman.fight());
    }
}
```

输出如下：

```java
D/dagger2: swordman:fighting
```

可以看出依赖的 swordman Component 也注入成功。

## 懒加载

懒加载模式即 @Inject 不初始化，使用时才初始化。改写上面的 swordman component 如下：

```java
public class MainDaggerActivity extends Activity {

    @Inject
    Gson gson1;

    @Inject
    Gson gson2;

    @Inject
    Lazy<Swordman> swordman;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_dagger);
        SampleApplication.get(this).getActivityComponent().inject(this);
        Log.d("dagger2", "lazy swordman:" + swordman.get().fight());
    }
}
```

可以看出 @Inject 了 Lazy 对象，使用时需要先 get() 得到真正的 swordman 对象。

输出如下：

```java
D/dagger2: lazy swordman:fighting
```

## 总结

Dagger2 通过注解的方式构造实例对象，避免使用者在调用对象的时候一层层构造依赖。因为使用者不需要了解它所依赖的对象的细节，而只需要调用依赖的方法，所以可以通过注解的方式将依赖组建的构造方式交给 Dagger2 的 Component 去处理，避免了大量重复的构建工作。

常用的注解有 @Inject、@Component、@Module、@Provides、@Named、@Qualifier、@Singleton、@Scope 等，也可以使用 Lazy 懒加载的方式在使用时创建对象。