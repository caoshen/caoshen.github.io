---
title: Android 源码的策略模式
date: 2019-10-31 23:55:08
tags: 设计模式
categories: Android
---

# Android 源码的策略模式

## 策略模式介绍

实现某一个功能可以有多种算法或者策略，我们根据实际情况选择不同的算法或者策略来完成该功能。例如，排序算法，可以使用插入排序、归并排序、冒泡排序等。

如果将这些算法或者策略抽象出来，提供一个统一的接口，不同的算法或者策略有不同的实现类，这样在程序客户端就可以通过注入不同的实现对象来实现算法或者策略的动态替换，这种模式的可扩展性、可维护性也就更高，也就是我们说的策略模式。

## 策略模式的定义

策略模式定义了一系列的算法，并将每一个算法封装起来，而且使它们还可以相互替换。策略模式让算法独立于使用它的客户而独立变化。

## 时间插值器

Android 的 TimeInterpolator 时间插值器可以根据时间流逝的百分比计算出当前属性值改变的百分比。系统预置的有线性插值器 LinearInterpolator、加速减速插值器 AccelerateDecelerateInterpolator、减速插值器 DecelerateInterpolator 和其他一些插值器。

### LinearInterpolator

线性插值器的代码如下：

```java
public class LinearInterpolator extends BaseInterpolator implements NativeInterpolatorFactory {
...
    public float getInterpolation(float input) {
        return input;
    }
...
}
```

LinearInterpolator 的 getInterpolation 方法直接把输入 input 返回，这样输出百分比就是输入百分比，不做任何处理，使得动画的速率不会发生变化。

### AccelerateInterpolator

加速插值器的代码如下：

```java
public class AccelerateInterpolator extends BaseInterpolator implements NativeInterpolatorFactory {
    private final float mFactor;
    private final double mDoubleFactor;

    public AccelerateInterpolator() {
        mFactor = 1.0f;
        mDoubleFactor = 2.0;
    }
...
    public float getInterpolation(float input) {
        if (mFactor == 1.0f) {
            return input * input;
        } else {
            return (float)Math.pow(input, mDoubleFactor);
        }
    }
...
}
```

可以看出加速插值器的 mFactor 如果是 1.0，那么随着时间的变化速率是按照平方变化的；如果 mFactor 不是 1.0，那么变化速率是按照 mDoubleFactor 的幂次方变化的。也就是说刚开始变化速度最慢，时间结束时的变化速度最快。

### BaseInterpolator

这些插值器都继承了 BaseInterpolator 类，而 BaseInterpolator 实现了 Interpolator 接口。

```java
public interface Interpolator extends TimeInterpolator {
    // A new interface, TimeInterpolator, was introduced for the new android.animation
    // package. This older Interpolator interface extends TimeInterpolator so that users of
    // the new Animator-based animations can use either the old Interpolator implementations or
    // new classes that implement TimeInterpolator directly.
}
```

Interpolator 接口继承了 TimeInterpolator。

```java
public interface TimeInterpolator {
    ...
    float getInterpolation(float input);
}
```

TimeInterpolator 声明了 getInterpolation 方法。通过 getInterpolation 方法来修改动画的流逝时间百分比，以此达到动画的加速、减速等效果

Interpolator 就是这个计算策略的抽象，LinearInterpolator、CycleInterpolator 等插值器就是具体的实现策略，通过注入不同的插值器实现不同的动态效果。