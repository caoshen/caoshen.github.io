---
title: Java 注解与注解处理器
date: 2019-09-08 23:23:55
tags: 注解
categories: Java
---

# Java 注解与注解处理器

从 JDK 5 开始，Java 增加了注解，注解是代码里面的特殊标记，这些标记可以在编译、类加载、运行时被读取，并执行一些相应的处理。通过使用注解，开发人员可以在不改变原有逻辑的情况下，在源文件中嵌入一些补充的信息。代码分析工具、开发工具和部署工具可以通过这些补充信息进行验证、处理或者进行部署。

## 注解

注解可以分为标准注解和元注解。

标准注解是 JDK 自带的注解。元注解是用来注解其他注解的注解。

### 标准注解

标准注解由以下 4 种：

1. @Override 重写注解。对覆盖超类中的方法进行标记，如果被标记的方法没有实际覆盖超类中的方法，编译器会发出警告。
2. @Deprecated 过时注解。对不鼓励使用或者已经过时的方法添加注解。当编程人员在使用这些方法时，会在编译时显示提示信息。
3. @SuppressWarnings 取消警告注解。选择性地取消特定代码段中的警告，比如 lint 警告。
4. @SafeVarargs JDK 7 新增，用来声明使用了可变长度参数的方法，其在与泛型类一起使用时不会出现类型安全的问题。

### 元注解

除了标准注解，还有元注解，它用来注解其他注解，从而创建新的注解。元注解有以下几种：

- @Target 注解所修饰的对象范围
- @Inherited 表示注解可以被继承
- @Documented 表示这个注解应该被 JavaDoc 工具记录
- @Retention 用来声明注解的保留策略
- @Repeatable JDK 8 新增，允许一个注解在同一个声明类型（类、属性或者方法）上多次使用

@Target 注解取值是一个 ElementType 类型的数组，其中有以下几种取值，对应不同的对象范围。

- ElementType.TYPE 修饰类、接口或者枚举类型
- ElementType.FIELD 修饰成员变量
- ElementType.METHOD 修饰方法
- ElementType.PARAMETER 修饰参数
- ElementType.CONSTRUCTOR 修饰构造方法
- ElementType.LOCAL_VARIABLE 修饰局部变量
- ElementType.ANNOTATION_TYPE 修饰注解
- ElementType.PACKAGE 修饰包
- ElementType.TYPE_PARAMETER 类型参数声明
- ElementType.TYPE_USE 使用类型

@Retention 注解有 3 种类型，分别表示不同级别的保留策略。

- RetentionPolicy.SOURCE 源码级注解。注解信息只会保留在 java 源码中，源码在编译后注解信息被丢弃，不会保留在 class 文件
- RetentionPolicy.CLASS 编译时注解。注解信息会保留在 java 源码以及 class 文件中。当运行 java 程序时，JVM 会丢弃该注解信息，不会保留在 JVM 中
- RetentionPolicy.RUNTIME 运行时注解，当运行 java 程序时，JVM 也会保留该注解信息，可以通过反射获取该注解信息。

### 定义注解

定义注解使用 @interface 关键字。

```java
public @interface Swordsman {
    String name() default "zhaomin";

    int age() default 18;
}
```

定义注解后，可以在程序中使用该注解。

```java
@Swordsman(name = "zhangwuji", age = 20)
public class AnnotationTest {
}
```

可以看出注解只有成员变量，没有方法。注解的成员变量在注解定义中以无形参的方法表示。返回值定义了成员变量的类型。

使用注解时，可以给成员变量指定值。也可以在定义注解时使用 default 关键字指定默认值。如果有默认值，可以在使用注解时不为成员变量赋值。

### 定义运行时注解

可以用 @Retention 来设定注解的保留策略。这 3 个策略的生命周期长度为 SOURCE < CLASS < RUNTIME 。生命周期短的能起作用的地方，生命周期长的也一定能起作用。一般如果需要在运行时去动态获取注解信息，那么只能用 RetentionPolicy.RUNTIME；如果要在编译时进行一些预处理操作，比如生产一些辅助代码，就用 RetentionPolicy.CLASS；如果只是做一些检查的操作，就用 RetentionPolicy.SOURCE。

@Override 注解是源码级别注解（RetentionPolicy.SOURCE）：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

Retrofit 的 @GET 注解是运行时注解（Retention(RUNTIME)）：

```java
@Documented
@Target(METHOD)
@Retention(RUNTIME)
public @interface GET {
  String value() default "";
}
```

### 定义编译时注解

如果将 @Retention 的保留策略设定为 RetentionPolicy.CLASS，这个注解就是编译时注解，如下所示：

```java
@Retention(RetentionPolicy.CLASS)
public @interface Swordsman {
    String name() default "zhaomin";

    int age() default 18;
}
```

## 注解处理器

如果没有处理注解的工具，那么注解也不会有太大的作用。对于不同的注解有不同的注解处理器。虽然注解处理器的编写千变万化，但是也有处理标准。

针对运行时注解会采用反射机制处理，针对编译时注解会采用 AbstractProcessor 来处理。

### 运行时注解处理器

处理运行时注解需要用到反射机制。定义运行时注解如下：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface Get {
    String value() default "";
}
```

接着使用 Get 注解如下：

```java
public class AnnotationTest {

    @Get(value = "http://ip.taobao.com/59.108.54.37")
    public String getIpMsg() {
        return "";
    }

    @Get(value = "http://ip.taobao.com/")
    public String getIp() {
        return "";
    }
}
```

接下来写一个简单的注解处理器，用来获取 Get 注解的值。

```java
public class AnnotationProcessor {

    public static void main(String[] args) {
        Method[] methods = AnnotationTest.class.getDeclaredMethods();
        for (Method m : methods) {
            Get get = m.getAnnotation(Get.class);
            System.out.println(get.value());
        }
    }
}
```

以上代码通过 getDeclaredMethods 获取类的方法，通过 getAnnotation 获取方法的注解，最后调用 Get 的 value 方法返回注接的成员变量的值。

输出如下：

```
http://ip.taobao.com/59.108.54.37
http://ip.taobao.com/
```

### 编译时注解处理器

#### 定义注解

首先新建一个叫做 annotations 的 Java Library 存放注解：

```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface BindView {
    int value() default 1;
}
```

#### 编写注解处理器

然后新建一个叫做 processor 的 Java Library 存放注解处理器：

```java
public class ClassProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        Messager messager = processingEnv.getMessager();
        for (Element ele : roundEnvironment.getElementsAnnotatedWith(BindView.class)) {
            if (ele.getKind() == ElementKind.FIELD) {
                messager.printMessage(Diagnostic.Kind.NOTE, "printMessage:" + ele.toString());
            }
        }
        return true;
    }

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotations = new LinkedHashSet<>();
        annotations.add(BindView.class.getCanonicalName());
        return annotations;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }
}
```

- init 被注解处理工具调用，并输入 processingEnvironment 参数。processingEnvironment 提供了很多工具类，如 Elements、Types、Filer 和 Messenger 等。
- process 注解处理的主函数，这里处理扫描、评估和处理注解的代码，以及生产 Java 文件。
- getSupportedAnnotationTypes 指明注解处理器是处理哪些注解的。
- getSupportedSourceVersion 指明使用的 Java 版本，通常返回 SourceVersion.latestSupported。

在 Java 7 以后，也可以用注解的形式代替 getSupportedAnnotationTypes 和 getSupportedSourceVersion。即 @SupportedAnnotationTypes 和 @SupportedSourceVersion 注解。

```java
@SupportedSourceVersion(SourceVersion.RELEASE_8)
@SupportedAnnotationTypes("com.caoshen.annotations.BindView")
public class ClassProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        ...
    }

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
    }
}
```

并在 processor 的 build.gradle 配置依赖：

```
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation project(':annotations')
}
```

#### 注册注解处理器

在 processor 的 main 目录下新建 resources 目录，然后添加一个 META-INF/services 目录。

在 META-INF/services 目录下新建一个名叫 javax.annotation.processing.Processor 的文件。

文件内容如下：

```
com.caoshen.processor.ClassProcessor
```

ClassProcessor 用来表示之前定义的注解处理器。

#### auto-service

如果觉得上一步的注册 service 步骤麻烦，可以使用 google 的 auto service 自动注册。

在 processor 的 build.gradle 配置依赖：

```
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation project(':annotations')
    // auto service
    implementation 'com.google.auto.service:auto-service:1.0-rc6'
    annotationProcessor 'com.google.auto.service:auto-service:1.0-rc6'
}
```

注意这里的依赖要加上 annotationProcessor 这一行，不然无法生成 META-INF/services 目录以及里面的注解处理器。

```
annotationProcessor 'com.google.auto.service:auto-service:1.0-rc6'
```

然后在注解处理器类添加 @AutoService 注解如下：

```java
@AutoService(Processor.class)
@SupportedSourceVersion(SourceVersion.RELEASE_8)
@SupportedAnnotationTypes("com.caoshen.annotations.BindView")
public class ClassProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
       ...
    }
...
}
```

生成的 javax.annotation.processing.Processor 文件在以下目录

```
processor\build\classes\java\main\META-INF\services
```

这样就可以避免手动注册。

#### 应用注解

在 app 模块的 build.gradle 依赖注解和注解处理器

```java
dependencies {
    ...
    // annotation
    implementation project(':annotations')
    annotationProcessor project(':processor')
}
```

在 AnnotationActivity 使用注解如下：

```java
public class AnnotationActivity extends Activity {

    @BindView(value = R.id.tv_text)
    TextView textTv;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_annotations);
    }
}
```

重新 build 项目，在 Run 窗口会打印出 @BindView 注解对应的 TextView 的名称，即 textTv。

输出如下：

```
注: printMessage:textTv
```