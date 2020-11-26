# Android 源码的备忘录模式

## 备忘录模式介绍

备忘录模式是一种行为模式，该模式用于保存对象当前状态，并且在之后可以再次恢复到此状态，这有点像我们平时说的“后悔药”。备忘录模式实现的方式需要保证被保存的对象状态不能被对象从外部访问，目的是为了保护好被保存的这些对象状态的完整性以及内部实现不向外暴露。

## 备忘录模式的定义

在不破坏封闭的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态。这样以后就可将该对象恢复到原先保存的状态。

## Android 源码中的备忘录模式

在 Android 开发中，状态模式应用是 Activity 中的状态保存，也就是里面的 onSaveInstanceState 和 onRestoreInstanceState。当 Activity 不是正常方式退出，且 Activity 在随后的时间内被系统杀死之前会调用这两个方法让开发人员可以有机会存储 Activity 相关的信息。

### onSaveInstanceState

onSaveInstanceState 方法的代码如下：

```java
    protected void onSaveInstanceState(Bundle outState) {
        // 存储当前窗口的视图树的状态
        outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
        
        outState.putInt(LAST_AUTOFILL_ID, mLastAutofillId);
        // 存储 fragment 的状态
        Parcelable p = mFragments.saveAllState();
        if (p != null) {
            outState.putParcelable(FRAGMENTS_TAG, p);
        }
        // 存储自动填充的字段
        if (mAutoFillResetNeeded) {
            outState.putBoolean(AUTOFILL_RESET_NEEDED, true);
            getAutofillManager().onSaveInstanceState(outState);
        }
        // 如果用户还设置了 Activity 的 ActivityLifecycleCallbacks，那么调用这些 ActivityLifecycleCallbacks 的 onSaveInstanceState 进行存储状态
        getApplication().dispatchActivitySaveInstanceState(this, outState);
    }

```

Window 类的具体实现是 PhoneWindow，PhoneWindow 的 saveHierarchyState 方法如下：

```java
    @Override
    public Bundle saveHierarchyState() {
        Bundle outState = new Bundle();
        if (mContentParent == null) {
            return outState;
        }
        // 通过 SparseArray 类来存储，这相当于一个 key 为整型的 map
        SparseArray<Parcelable> states = new SparseArray<Parcelable>();
        // 调用 mContentParent 的 saveHierarchyState 方法，这个 mContentParent 就是调用 Activity 的 setContentView 函数设置的内容视图，它是内容视图的根节点，在这里存储整棵视图树的结构。
        mContentParent.saveHierarchyState(states);
        // 将视图树结构放到 outState 中
        outState.putSparseParcelableArray(VIEWS_TAG, states);

        // 保存当前界面中获取了焦点的 View
        // Save the focused view ID.
        final View focusedView = mContentParent.findFocus();
        if (focusedView != null && focusedView.getId() != View.NO_ID) {
            // 持有焦点的 View 必须要设置 id，否则重新进入该界面时不会恢复它的焦点状态
            outState.putInt(FOCUSED_ID_TAG, focusedView.getId());
        }
        // 存储整个面板的状态
        // save the panels
        SparseArray<Parcelable> panelStates = new SparseArray<Parcelable>();
        savePanelState(panelStates);
        if (panelStates.size() > 0) {
            outState.putSparseParcelableArray(PANELS_TAG, panelStates);
        }
        // 存储 ActionBar 的状态
        if (mDecorContentParent != null) {
            SparseArray<Parcelable> actionBarStates = new SparseArray<Parcelable>();
            mDecorContentParent.saveToolbarHierarchyState(actionBarStates);
            outState.putSparseParcelableArray(ACTION_BAR_TAG, actionBarStates);
        }

        return outState;
    }

```

在 saveHierarchyState 中，主要时存储了与当前 UI、ActionBar 相关的 View 状态，这里用 mContentParent 来分析。这个 mContentParent 就是我们通过 Activity 的 setContentView 函数设置的内容视图，它是整个内容视图的根节点，存储它层级结构中的 View 状态也就存储了用户界面的状态。mContentParent 是一个 ViewGroup 对象，但是，saveHierarchyState 并不是在 ViewGroup 中，而是在 ViewGroup 的父类 View。

View 的 saveHierarchyState 方法如下：

```java
    public void saveHierarchyState(SparseArray<Parcelable> container) {
        dispatchSaveInstanceState(container);
    }
```

saveHierarchyState 调用了 dispatchSaveInstanceState 方法。

dispatchSaveInstanceState 方法如下：

```java
    protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
        // 注意：如果 View 没有设置 id，那么这个 View 的状态将不会被存储。
        if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
            mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
            // 调用 onSaveInstanceState 获取自身的状态
            Parcelable state = onSaveInstanceState();
            if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
                throw new IllegalStateException(
                        "Derived class did not call super.onSaveInstanceState()");
            }
            if (state != null) {
                // Log.i("View", "Freezing #" + Integer.toHexString(mID)
                // + ": " + state);
                // 将自身状态放到 container 中，key 为 id、value 为自身状态。

                container.put(mID, state);
            }
        }
    }
```

View 的 onSaveInstanceState 方法如下：

```java
    @CallSuper
    @Nullable protected Parcelable onSaveInstanceState() {
        mPrivateFlags |= PFLAG_SAVE_STATE_CALLED;
        ...
        return BaseSavedState.EMPTY_STATE;
    }
```

View 类默认存储的状态为空。

ViewGroup 的 dispatchSaveInstanceState 如下：

```java
    @Override
    protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
        super.dispatchSaveInstanceState(container);
        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            View c = children[i];
            if ((c.mViewFlags & PARENT_SAVE_DISABLED_MASK) != PARENT_SAVE_DISABLED) {
                c.dispatchSaveInstanceState(container);
            }
        }
    }
```

dispatchSaveInstanceState 会首先调用 super 的方法存储自身的状态。然后调用每个子视图的 dispatchSaveInstanceState。

### TextView 的 onSaveInstanceState

TextView 复写了 onSaveInstanceState 方法。

```java
    @Override
    public Parcelable onSaveInstanceState() {
        Parcelable superState = super.onSaveInstanceState();
        ...
        // 存储 TextView 的 start、end 以及文本内容
        if (freezesText || hasSelection) {
            SavedState ss = new SavedState(superState);

            if (freezesText) {
                if (mText instanceof Spanned) {
                    final Spannable sp = new SpannableStringBuilder(mText);

                    if (mEditor != null) {
                        removeMisspelledSpans(sp);
                        sp.removeSpan(mEditor.mSuggestionRangeSpan);
                    }

                    ss.text = sp;
                } else {
                    ss.text = mText.toString();
                }
            }

            if (hasSelection) {
                // XXX Should also save the current scroll position!
                ss.selStart = start;
                ss.selEnd = end;
            }
        ...

        return superState;
    }

```

存储完 Window 的视图树状态后，会存储每个 Fragment 的状态，调用它们的 onSaveInstanceState 方法。最后调用 ActivityLifecycleCallbacks 的 onSaveInstanceState。

### onSaveInstanceState 和 performStopActivity

P 版本（Android 9）之前，onSaveInstanceState 会在 onStop 之前调用。

P 版本（Android 9）之后，onSaveInstanceState 会在 onStop 之后调用。


ActivityThread 的 performStopActivity 会调用 callActivityOnStop。

callActivityOnStop 代码如下：

```java
    private void callActivityOnStop(ActivityClientRecord r, boolean saveState, String reason) {
        // Before P onSaveInstanceState was called before onStop, starting with P it's
        // called after. Before Honeycomb state was always saved before onPause.
        final boolean shouldSaveState = saveState && !r.activity.mFinished && r.state == null
                && !r.isPreHoneycomb();
        final boolean isPreP = r.isPreP();
        if (shouldSaveState && isPreP) {
            callActivityOnSaveInstanceState(r);
        }

        try {
            r.activity.performStop(false /*preserveWindow*/, reason);
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException(
                        "Unable to stop activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
            }
        }
        r.setState(ON_STOP);

        if (shouldSaveState && !isPreP) {
            callActivityOnSaveInstanceState(r);
        }
    }
```

可以看出如果 isPreP 是 false，那么 onStop 之后才会 onSaveInstanceState。

callActivityOnSaveInstanceState 方法会将状态信息存储到ActivityClientRecord对象的state字段中。

```java
    private void callActivityOnSaveInstanceState(ActivityClientRecord r) {
        r.state = new Bundle();
        r.state.setAllowFds(false);
        if (r.isPersistable()) {
            r.persistentState = new PersistableBundle();
            mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state,
                    r.persistentState);
        } else {
            mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state);
        }
    }
```
在 ActivityThread 类的 performLaunchActivity 方法会回调 onCreate，将 ActivityClientRecord 对象的 state 字段传递给 onCreate。

```java
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
        ...
        return activity;
    }
```

在 ActivityThread 类的 handleStartActivity 方法中会调用 callActivityOnRestoreInstanceState 恢复 InstanceState。

```java
    @Override
    public void handleStartActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions) {
        ...
        // Restore instance state
        if (pendingActions.shouldRestoreInstanceState()) {
            if (r.isPersistable()) {
                if (r.state != null || r.persistentState != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                            r.persistentState);
                }
            } else if (r.state != null) {
                mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
            }
        }
        ...
    }
```