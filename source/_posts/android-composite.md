---
title: Android 源码的组合模式
date: 2019-11-23 23:59:01
tags: 设计模式
categories: Android
---

# Android 源码的组合模式

## 组合模式介绍

组合模式（Composite Pattern）也称为部分整体模式（Part-Whole Pattern），结构型设计模式之一。

组合模式比较简单，它将一组相似的对象看作一个对象处理，并根据一个树状结构来组合对象，然后提供一个统一的方法去访问相应的对象，以此忽略掉对象与对象集合之间的差别。

## 组合模式的定义

将对象组合成树形结构以表示“部分-整体”的层次结构，使得用户对单个对象和组合对象的使用具有一致性。

## Android 源码中的组合模式实现

Android 中的 View 和 ViewGroup 的嵌套组合是一个典型的组合模式实现。

在 Android 的这个视图层级中，容器一定是 ViewGroup，而且只有 ViewGroup 才能包含其他的 View，比如 LinearLayout 能包含 TextView、Button、CheckBox 等，但是反过来 TextView 是不能包含 LinearLayout 的，因为 TextView 直接继承于 View，其并非一个容器。

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    ...
    public void addView(View child) {
        addView(child, -1);
    }
    ...
    @Override
    public void removeView(View view) {
        if (removeViewInternal(view)) {
            requestLayout();
            invalidate(true);
        }
    }
    ...
    public View getChildAt(int index) {
        if (index < 0 || index >= mChildrenCount) {
            return null;
        }
        return mChildren[index];
    }
    ...
}
```
## 为什么 ViewGroup 有容器的功能

ViewGroup 是继承于 View 类的。

```java
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    ...
}
```

从继承的角度来说，ViewGroup 拥有 View 类所有的非私有方法。既然如此，两者的差别就在于 ViewGroup 所实现的 ViewParent 和 ViewManager 接口上，而事实也是如此。

ViewManager 定义了 addView、removeView 等对子视图操作的方法。

```java
public interface ViewManager
{
    /**
     * Assign the passed LayoutParams to the passed View and add the view to the window.
     * <p>Throws {@link android.view.WindowManager.BadTokenException} for certain programming
     * errors, such as adding a second view to a window without removing the first view.
     * <p>Throws {@link android.view.WindowManager.InvalidDisplayException} if the window is on a
     * secondary {@link Display} and the specified display can't be found
     * (see {@link android.app.Presentation}).
     * @param view The view to be added to this window.
     * @param params The LayoutParams to assign to view.
     */
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

而 ViewParent 则定义了刷新容器的接口 requestLayout 和其他一些焦点事件的处理的接口。

```java
public interface ViewParent {
    // 请求重新布局
    public void requestLayout();

    // 是否已经请求布局。这里需要注意，当我们调用 requestLayout 请求布局后，这一过程并非是立即执行的，Android 会将请求布局的操作以消息的形式发送至主线程的 Handler 并由其分发处理。因此在调用 requestLayout 方法请求布局到布局真正接收到重新布局的命令时需要一段时间间隔
    public boolean isLayoutRequested();

    ...
    // 获取当前 View 的 ViewParent
    public ViewParent getParent();
    ...
}
```

其中有一些方法比较常见，比如 requestLayout 和 bringChildToFront 等。

ViewGroup 除了所实现的这两个接口与 View 不一样外，还有重要的一点就是 ViewGroup 是抽象类，将 View 的 onLayout 重置为抽象方法。容器子类必须实现 onLayout 来布局定位。

除此之外，在 View 中比较重要的两个测绘流程的方法 onMeasure 和 onDraw 在 ViewGroup 中都没有被重写，相对于 onMeasure 方法，在 ViewGroup 中增加了一些计算子 View 的方法，如 measureChildren、measureChildrenWithMargins 等；而对于 onDraw 方法，ViewGroup 定义了一个 dispatchDraw 方法来调用其每一个子 View 的 onDraw 方法，由此可见，ViewGroup 真的就象一个容器一样，其职责只是负责对子元素的操作而非具体的个体行为。
