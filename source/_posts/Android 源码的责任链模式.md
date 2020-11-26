# Android 源码的责任链模式

## 责任链模式介绍

责任链模式是行为型设计模式之一。多个节点首尾相连接所构成的模型称为链。对于链式结构，每个节点都可以被拆开再连接，因此，链式结构也具有很好的灵活性。将这样一种结构应用于编程领域，将每一个节点看作是一个对象，每一个对象拥有不同的处理逻辑，将一个请求从链式的首端发出，沿着链的路径依次传递给每一个节点对象，直至有对象处理这个请求为止。我们将这种模式称为责任链模式。

## 责任链模式的定义

使多个对象都有机会处理请求，从而避免了请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，只到有对象处理它为止。

## ViewGroup 的 dispatchTouchEvent

责任链模式在 Android 源码中比较类似的实现是 View 的 MotionEvent 事件分发。每当用户接触屏幕的时候，Android 都会将对应的事件包装成一个事件对象从 ViewTree 的顶部自上而下地分发传递。

ViewGroup 的 dispatchTouchEvent 方法如下：

```java
    public boolean dispatchTouchEvent(MotionEvent ev) {
        ...
        // 对于辅助功能的事件处理
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // 处理原始的 ACTION_DOWN 事件
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // 这里主要是在新事件开始时，处理完上一个事件
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // 检查事件拦截
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    // 恢复事件防止其改变
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            // 如果事件被拦截了，则进行正常的事件分发
            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // 检查事件是否取消
            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // 如果有必要的话，为 DOWN 事件检查所有的目标对象
            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            // 如果事件未被取消并未被拦截
            if (!canceled && !intercepted) {

                // 如果有辅助功能的参与，则直接将事件投递到对应的 View，否则将事件分发给所有的子 View
                // If the event is targeting accessibility focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;
                // 如果事件为起始事件
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    // 如果 TouchTarget 为空并且子元素不为 0
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // 由上至下去寻找一个可以接收该事件的子 View
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        // 遍历子元素
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            
                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }
                            // 如果这个子元素无法接收 Pointer Event 或这个事件点压根就没有落在子元素的边界范围内，那么跳出该次循环继续遍历
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }
                            // 找到 Event 改由哪个子元素持有
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            // 投递事件执行触摸操作
                            // 如果子元素还是一个 ViewGroup，则递归调用重复此过程
                            // 如果子元素是一个 View，那么则会调用 View 的 dispatchTouchEvent，并最终由 onTouchEvent 处理
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // 子 View 在其边界范围内接收事件
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

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    // 如果发现没有子元素可以持有该次事件
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }

```

## ViewGroup 的 dispatchTransformedTouchEvent

这里我们主要看看 dispatchTransformedTouchEvent 方法是如何调度子元素 dispatchTouchEvent 方法的：

```java
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
        // 如果事件被取消
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            // 如果没有子元素
            if (child == null) {
                // 那么就直接调用父类的 dispatchTouchEvent，注意，这里的父类终会为 View 类
                handled = super.dispatchTouchEvent(event);
            } else {
                // 如果有子元素则传递 cancel 事件
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // 计算即将被传递的点的数量
        // Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // 如果事件没有相应的点，那么就丢弃该事件
        // If for some reason we ended up in an inconsistent state where it looks like we
        // might produce a motion event with no pointers in it, then drop the event.
        if (newPointerIdBits == 0) {
            return false;
        }

        // 声明临时变量保存坐标转换后的 MotionEvent
        // If the number of pointers is the same and we don't need to perform any fancy
        // irreversible transformations, then we can reuse the motion event for this
        // dispatch as long as we are careful to revert any changes we make.
        // Otherwise we need to make a copy.
        final MotionEvent transformedEvent;
        // 如果事件点的数量一致
        if (newPointerIdBits == oldPointerIdBits) {
            // 子元素为空或子元素有一个单位矩阵
            if (child == null || child.hasIdentityMatrix()) {
                // 再次区分子元素为空的情况
                if (child == null) {
                    // 为空则调用父类 dispatchTouchEvent
                    handled = super.dispatchTouchEvent(event);
                } else {
                    // 否则尝试获取 xy 方向上的偏移量（如果通过 scrollTo 或者 scrollBy 对子视图进行滚动的话）
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    // 将 MotionEvent 进行坐标变换
                    event.offsetLocation(offsetX, offsetY);
                    // 再将变换后的 MotionEvent 传递给子元素
                    handled = child.dispatchTouchEvent(event);
                    // 复位 MotionEvent 以便之后再次使用
                    event.offsetLocation(-offsetX, -offsetY);
                }
                // 如果通过以上的逻辑判断，当前事件被持有则可以直接返回
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // 下述是一样的不再赘述
        // Perform any necessary transformations and dispatch.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }
```

ViewGroup 事件投递的递归调用就类似于一条责任链，一旦其寻找到责任者，那么将由责任者持有并消费掉该次事件，具体地体现再 View 地 onTouchEvent 方法中返回值的设置，如果 onTouchEvent 返回 false，那么意味着当前 View 不会是该次事件的责任人，将不会对其持有；如果为 true 则相反，此时 View 会持有该事件并不再向外传递。