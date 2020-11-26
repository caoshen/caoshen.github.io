# Android 源码的访问者模式

## 访问者模式介绍

访问者模式是一种将数据操作与数据结构分离的设计模式，它是 23 种设计模式中最复杂的一个，但是它的使用频率并不高，正如《设计模式》的作者 GOF 对访问者模式的描述：大多数情况下，并不需要使用访问者模式，但是当你一旦需要使用它时，那你就是真的需要它了。

访问者模式的基本想法是，软件系统中拥有一个由许多对象构成的、比较稳定的对象结构，这些对象的类都拥有一个 accept 方法用来接受访问者对象的访问。访问者是一个接口，它拥有一个 visit 方法，这个方法对访问到的对象结构中不同类型的元素做出不同的处理。在对象结构的每一次访问过程中，我们遍历整个对象结构，对每一个元素都实施 accept 方法，在每一个元素的 accept 方法中会调用访问者的 visit 方法，从而使访问者得以处理对象结构的每一个元素，我们可以针对对象结构设计不同的访问者类来完成不同的操作，达到区别对待的效果。

## 访问者模式的定义

封装一些作用于某种数据结构中的各元素的操作，他可以在不改变这个数据结构的前提下定义作用于这些元素的新的操作。

## Android 源码中的访问者模式

Android 的编译时注解是一种访问者模式。编译时注解的核心原理依赖 APT （Annotation Processing Tools）实现。

编译时注解解析的基本原理是，在某些代码元素上（如类型、函数、字段等）添加注解，在编译时编译器会检查 AbstractProcessor 的子类，并且调用该类型的 process 函数，然后将添加了注解的所有元素都传递到 process 函数中，使得开发人员可以在编译期进行相应的处理。

编写注解处理器的核心是 AnnotationProcessorFactory 和 AnnotationProcessor 两个接口，后者表示的是注解处理器，而前者则是为某些注解类型创建注解处理器的工厂。

对于编译器来说，代码中的元素结构是基本不变的，例如，组成代码的基本元素由包、类、函数、字段、类型参数、变量。JDK 中为这些元素定义了一个基类，也就是 Element 类，它有如下几个子类：

1. PackageElement 包元素，包含了某个包下的信息，可以获取到包名等；
2. TypeElement 类型元素，如某个字段属于某种类型；
3. ExecutableElement 可执行元素，代表了函数类型的元素；
4. VariableElement 变量元素；
5. TypeParameterElement 类型参数元素。

因为注解可以指定作用在哪些元素上，因此，通过上述的抽象来对应这些元素，例如下面这个注解，指定的是只能作用于类上面，并且这个注解只能保留在 class 文件中（编译时注解）：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface Test {
    String value();
}
```

### Element

Element 接口位于 javax.lang.model.element。

```java
public interface Element extends javax.lang.model.AnnotatedConstruct {

    TypeMirror asType();
    // 获取元素类型
    ElementKind getKind();
    // 获取元素修饰符，如 public、static、final 等
    Set<Modifier> getModifiers();

    List<? extends Element> getEnclosedElements();
    // 接受访问者的访问
    <R, P> R accept(ElementVisitor<R, P> v, P p);
}
```

可以看到 Element 定义了一个代码元素的一些通用接口，其中很显眼的就是 accept 函数，这个函数接收一个 ElementVisitor 和类型为 P 的参数，ElementVisitor 就是访问者类型，而 P 则用于传递一些额外的参数给 visitor。这是一个典型的访问者模式。

ElementVisitor 定义如下：

```java
// 元素访问者
public interface ElementVisitor<R, P> {
    // 访问元素
    R visit(Element var1, P var2);

    R visit(Element var1);
    // 访问包元素
    R visitPackage(PackageElement var1, P var2);
    // 访问类型元素
    R visitType(TypeElement var1, P var2);
    // 访问变量元素
    R visitVariable(VariableElement var1, P var2);
    // 访问可执行元素
    R visitExecutable(ExecutableElement var1, P var2);
    // 访问参数元素
    R visitTypeParameter(TypeParameterElement var1, P var2);
    // 访问位置元素，为后续扩展预留的接口
    R visitUnknown(Element var1, P var2);
}
```

当 Visitor 对元素结构进行访问时，就可以针对不同的类型进行不同的处理。

例如 SimpleElementVisitor6 就是其中一个访问者，它基本上没做什么操作，直接返回了元素的默认值。

```java
@SupportedSourceVersion(SourceVersion.RELEASE_6)
public class SimpleElementVisitor6<R, P> extends AbstractElementVisitor6<R, P> {
    protected final R DEFAULT_VALUE;

    protected SimpleElementVisitor6() {
        this.DEFAULT_VALUE = null;
    }

    protected SimpleElementVisitor6(R var1) {
        this.DEFAULT_VALUE = var1;
    }

    protected R defaultAction(Element var1, P var2) {
        return this.DEFAULT_VALUE;
    }

    public R visitPackage(PackageElement var1, P var2) {
        return this.defaultAction(var1, var2);
    }

    public R visitType(TypeElement var1, P var2) {
        return this.defaultAction(var1, var2);
    }

    public R visitVariable(VariableElement var1, P var2) {
        return var1.getKind() != ElementKind.RESOURCE_VARIABLE ? this.defaultAction(var1, var2) : this.visitUnknown(var1, var2);
    }

    public R visitExecutable(ExecutableElement var1, P var2) {
        return this.defaultAction(var1, var2);
    }

    public R visitTypeParameter(TypeParameterElement var1, P var2) {
        return this.defaultAction(var1, var2);
    }
}
```

SimpleElementVisitor6 直接返回了默认值 null，也可以自己设定默认值。

### ElementKindVisitor6

另一个提取元素类型的访问者是 ElementKindVisitor6。

```java
@SupportedSourceVersion(SourceVersion.RELEASE_6)
public class ElementKindVisitor6<R, P> extends SimpleElementVisitor6<R, P> {
    protected ElementKindVisitor6() {
        super((Object)null);
    }

    protected ElementKindVisitor6(R var1) {
        super(var1);
    }

    public R visitPackage(PackageElement var1, P var2) {
        assert var1.getKind() == ElementKind.PACKAGE : "Bad kind on PackageElement";

        return this.defaultAction(var1, var2);
    }
    // 访问类型元素，比如类、注解、枚举、接口
    public R visitType(TypeElement var1, P var2) {
        ElementKind var3 = var1.getKind();
        switch(var3) {
        case ANNOTATION_TYPE:
            return this.visitTypeAsAnnotationType(var1, var2);
        case CLASS:
            return this.visitTypeAsClass(var1, var2);
        case ENUM:
            return this.visitTypeAsEnum(var1, var2);
        case INTERFACE:
            return this.visitTypeAsInterface(var1, var2);
        default:
            throw new AssertionError("Bad kind " + var3 + " for TypeElement" + var1);
        }
    }

    public R visitTypeAsAnnotationType(TypeElement var1, P var2) {
        return this.defaultAction(var1, var2);
    }

    public R visitTypeAsClass(TypeElement var1, P var2) {
        return this.defaultAction(var1, var2);
    }

    public R visitTypeAsEnum(TypeElement var1, P var2) {
        return this.defaultAction(var1, var2);
    }

    public R visitTypeAsInterface(TypeElement var1, P var2) {
        return this.defaultAction(var1, var2);
    }

    ...
}

```

ElementKindVisitor6 对于不同的类型进行不同的处理，提取各个元素的类型信息，例如，上述代码中对于 Type 类型的元素将分别进行处理，如类、枚举、接口、注解等。

首先，编译器将代码抽象成一个代码元素的树，然后再编译时对整棵树进行遍历访问，每个元素都有一个 accept 函数接受访问者的访问，每个访问者中都有对应的 visit 函数，例如，visitType 函数就是对类型元素的访问，在每个 visit 函数中对不同的类型进行不同的处理，这样就达到了差异处理的效果，同时将数据结构和数据操作分离，使得每个类型的职责单一，易于升级维护。JDK 还特意预留了 visitUnknown 接口来应对 Java 语言后续发展可能添加元素类型的问题，灵活地将访问者模式的缺点优化。