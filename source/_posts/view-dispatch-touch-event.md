---
title: view 的事件分发机制
date: 2019-08-19 23:09:01
tags: View
categories: Android
---

# view 的事件分发机制

当点击手机屏幕时，会产生点击事件，这个事件就是MotionEvent。view系统有一套事件分发机制，决定由哪一个 view 处理点击事件。

事件分发由三个重要的方法：
1. dispatchTouchEvent 事件分发
2. onInterceptTouchEvent 事件拦截
3. onTouchEvent 事件处理

事件的传递过程是由上至下传递。点击事件会依次传递给 Activity -> PhoneWindow -> DecorView -> ContentView -> ViewGroup。

## dispatchTouchEvent 方法

```java
    @Override

    public boolean dispatchTouchEvent(MotionEvent ev) {

        ...

        boolean handled = false;

        if (onFilterTouchEventForSecurity(ev)) {

            final int action = ev.getAction();

            final int actionMasked = action & MotionEvent.ACTION_MASK;



            // Handle an initial down.

            if (actionMasked == MotionEvent.ACTION_DOWN) {

                // Throw away all previous state when starting a new touch gesture.

                // The framework may have dropped the up or cancel event for the previous gesture

                // due to an app switch, ANR, or some other state change.

                cancelAndClearTouchTargets(ev);

                resetTouchState();

            }



            // Check for interception.

            final boolean intercepted;

            if (actionMasked == MotionEvent.ACTION_DOWN

                    || mFirstTouchTarget != null) {

                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;

                if (!disallowIntercept) {

                    intercepted = onInterceptTouchEvent(ev);

                    ev.setAction(action); // restore action in case it was changed

                } else {

                    intercepted = false;

                }

            } else {

                // There are no touch targets and this action is not an initial down

                // so this view group continues to intercept touches.

                intercepted = true;

            }

        ...

        return handled;
    }
```

首先判断 motionEvent 是不是 MotionEvent.ACTION_DOWN，即按下事件。如果是的，会进行初始化，把之前的触摸状态清空，mFirstTouchTarget = null。因为按下事件标志了一段手势的开始，所以会把之前的触摸状态清空，比如 up 或者 cancel 事件。

因为第一个事件是 ACTION_DOWN，所以会进入 if 语句。接下来判断子 View 是否不允许父 ViewGroup 拦截。如果 disallowIntercept 是 false，即允许 ViewGroup 拦截，那么会调用 onInterceptTouchEvent 方法，ViewGroup 在此方法中决定是否拦截 TouchEvent。

mFirstTouchTarget 表示第一个处理 Touch 事件的目标，如果它是 null，那么说明事件被 ViewGroup 拦截了，否则不是 null，交给子 View 处理。

当 ViewGroup 要拦截事件，后续的事件都会被拦截。因为此时不是 ACTION_DOWN ，而且 mFirstTouchTarget == null，走进 else 分支，intercepted = true。


```java
...
                        for (int i = childrenCount - 1; i >= 0; i--) {

                            final int childIndex = getAndVerifyPreorderedIndex(

                                    childrenCount, i, customOrder);

                            final View child = getAndVerifyPreorderedView(

                                    preorderedList, children, childIndex);

...
                            resetCancelNextUpFlag(child);

                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {

                                // Child wants to receive touch within its bounds.

                                mLastTouchDownTime = ev.getDownTime();

                                if (preorderedList != null) {

                                    // childIndex points into presorted list, find original index

                                    for (int j = 0; j < childrenCount; j++) {

                                        if (children[childIndex] == mChildren[j]) {

                                            mLastTouchDownIndex = j;

                                            break;

                                        }

                                    }

                                } else {

                                    mLastTouchDownIndex = childIndex;

                                }

                                mLastTouchDownX = ev.getX();

                                mLastTouchDownY = ev.getY();

                                newTouchTarget = addTouchTarget(child, idBitsToAssign);

                                alreadyDispatchedToNewTouchTarget = true;

                                break;

                            }
```
dispatchTransformedTouchEvent 里面如果有子 View，会调用子 View 的 dispatchTouchEvent 方法。如果 ViewGroup 没有子 View，会调用 super.dispatchTouchEvent 方法。

如果 OnTouchListener 不为空，会调用里面的 onTouch 方法。否则调用的是 onTouchEvent。

在 OnTouchEvent 方法中，如果有 clickable 标记或者 longClickable 标记，会在 ACTION_UP 的时候调用 performClick 方法。

如果在 performClick 方法中由设置 onClickListener，会执行 onClickListener 的 onClick 方法。



## onInterceptTouchEvent 方法。

```java
    public boolean onInterceptTouchEvent(MotionEvent ev) {

        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)

                && ev.getAction() == MotionEvent.ACTION_DOWN

                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)

                && isOnScrollbarThumb(ev.getX(), ev.getY())) {

            return true;

        }

        return false;

    }
```
可以看出除非 MotionEvent 是鼠标点击滚动条时，才会返回 true。默认情况下，直接返回 false，即不做拦截。如果想要 ViewGroup 拦截事件，必须在自定义的 ViewGroup 中重写这个方法。

值得注意的是，View 没有 onInterceptTouchEvent 方法。因为 View 是 View 树形结构的叶子节点，处于最底层，不能再拦截事件。只有处于中间和上层的 ViewGroup 才有这个方法。

