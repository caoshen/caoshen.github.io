---
title: View 的绘制流程
date: 2019-08-19 23:10:08
tags: View
categories: Android
---

# View 的绘制流程

View 的绘制流程其实就是 View 的 measure\layout\draw 过程，分别对应测量、布局和绘制。

## ViewRoot

ViewRoot 对于 ViewRootImpl 类，他是连接 WindowManager 和 DecorView 的桥梁。View 的三大流程都是通过 ViewRoot 完成的。在 ActivityThread 类中，activity 创建完毕后，会将 Decorview 添加到 Window 中，同时创建 ViewRootImpl 对象，并将 ViewRootImpl 和 DecorView 关联。

View 的绘制流程是从 ViewRootImpl 的 performTraversals 开始的，经过了 measure、layout、draw 三个过程，才能最终将一个 View 绘制出来。

measure 过程结束后，可以通过 getMeasuredWith 和 getMeasuredHeight 拿到 View 的测量宽高，在几乎所有的情况下都等同于 View 的最终宽高。除了在 layout 过程中修改了左上右下四个顶点的位置，导致实际宽高和测量宽高不一致。

layout 过程结束后，可以通过 getTop、getBottom、getLeft、getRight 拿到四个顶点的坐标，通过 getWidth 和 getHeight 拿到实际宽高。

draw 过程决定了 View 的显示，只有 draw 方法完成后，才能显示到屏幕上。

## measure 测量过程

measure 测量过程会使用 MeasureSpec 测量规格来表示 View 的测量尺寸。父容器的 MeasureSpec 和子元素的 LayoutParams 共同决定了子元素的 MeasureSpec。

MeasureSpec 使用 32 位整数来表示，其中高 2 位用来表示测量模式（SpecMode），低 30 位用来表示测量大小（SpecSize）。

SpecMode 有 3 种：EXACTLY、AT_MOST 和 UNSPECIFIED。 

EXACTLY 表示父容器检测出子元素的精确大小，子元素的最终大小就是 SpecSize 的值。它对应 LayoutParams 的 match_parent 和 具体大小。

AT_MOST 表示父容器制定了一个可用大小的 SpecSize，子元素的大小不能超过这个值，具体是什么值要看不同 View 的实现。它对应 LayoutParams 的 wrap_content。

UNSPECIFIED 表示 View 的大小不受限制，要多大给多大，一般用于系统内部，表示一种测量的状态。

对于 DecorView 来说，它的 MeasureSpec 是由窗口的大小和自身的 LayoutParams 决定的；对于普通的 View 来说，它的 MeasureSpec 是由父容器的 MeasureSpec 和自身的 LayoutParams 决定的。这个过程分别对应 getRootMeasureSpec 方法和 getChildMeasureSpec 方法。

根据 getRootMeasureSpec 方法，可以得到 DecorView 的测量规格如下：

| LayoutParams | SpecSize | SpecMode |
| - | - | - |
| MATCH_PARENT | windowSize | EXACTLY | 
| WRAP_CONTENT | windowSize | AT_MOST |
| 具体值 dp px | rootDimension | EXACTLY |

根据 getChildMeasureSpec 方法，可以得到普通 View 的测量规格如下：

| childLayoutParams\parentSpecMode | EXACTLY | AT_MOST | UNSPECIFIED |
| - | - | - | - |
| 具体值 dp px | EXACTLY\childSize | EXACTLY\childSize | UNSPECIFIED\childSize |
| MATCH_PARENT | EXACTLY\parentSize | AT_MOST\parentSize | UNSPECIFIED\0 |
| WRAP_CONTENT | AT_MOST\parentSize | AT_MOST\parentSize | UNSPECIFIED\0 |

View 和 ViewGroup 的测量过程有区别。View 通过 measure 方法就完成了测量过程，但是 ViewGroup 需要遍历它的子元素的测量过程。

View 的测量过程由 measure 方法完成。measure 方法是一个 final 方法，自类不可以重写。但是 measure 方法会调用 onMeasure，可以重写 onMeasure。onMeasure 方法会调用 setMeasuredDimension，将测量规格转换成测量后的宽高值。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

ViewGroup 是一个抽象类，它没有重写 View 的 onMeasure 方法，但是它提供了 measureChildren 方法。

```java
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```

可以看出 measureChildren 会对每一个子元素调用 measureChild 方法。

```java
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

measureChild 方法就是根据 LayoutParams 和父容器的 MeasureSpec 得到子元素的 MeasureSpec，最后子元素调用 measure 方法。

## layout 布局过程

layout 的作用是 ViewGroup 用来确定子元素的位置。当 ViewGroup 的位置确定后，它会在 onLayout 方法中遍历所有的子元素并调用其 layout 方法。

layout 方法的流程如下：
首先会通过 setFrame 确定 View 的四个顶点的位置。初始化 View 左上右下四个值。然后调用 onLayout 方法，确定每个子元素的位置。

View 的 onLayout 是一个空方法。

```java
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }
```

ViewGroup 的 onLayout 是一个抽象方法。因为不同类型的 ViewGroup 对应的布局也不一样。

```java
    @Override
    protected abstract void onLayout(boolean changed,
            int l, int t, int r, int b);
```

在布局流程结束后，View 的位置确定了，可以获取到宽高。

```java
    public final int getWidth() {
        return mRight - mLeft;
    }

    public final int getHeight() {
        return mBottom - mTop;
    }
```

可以看出 View 的宽度就是右坐标减去左坐标，高度就是下坐标减去上坐标。

## draw 绘制过程

View 的绘制过程共有一下几步：

1. 绘制背景 drawBackground
2. 绘制自己 onDraw
3. 绘制子元素 dispatchDraw
4. 绘制装饰 onDrawScrollBars

```java
    public void draw(Canvas canvas) {
       ...
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            drawAutofilledHighlight(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // Step 7, draw the default focus highlight
            drawDefaultFocusHighlight(canvas);

            if (debugDraw()) {
                debugDrawFocus(canvas);
            }

            // we're done...
            return;
        }

        ...
    }
```

