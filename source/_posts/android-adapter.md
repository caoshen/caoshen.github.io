---
title: Android 源码的适配器模式
date: 2019-11-24 17:41:52
tags: 设计模式
categories: Android
---

# Android 源码的适配器模式

## 适配器模式介绍

适配器模式在我们的开发中使用率极高，从代码中随处可见的 Adapter 就可以判断出来。从最早的 ListView、GridView 到现在最新的 RecyclerView 都需要使用 Adapter。

## 适配器模式的定义

适配器模式把一个类的接口变换成客户端所期待的另一种接口，从而使原本因接口不匹配而无法在一起工作的两个类能够一起工作。

## Android 源码中的适配器模式

在开发过程中，ListView 的 Adapter 是我们最常见的类型之一。

ListView 中并没有 Adapter 相关的成员变量，其实 Adapter 在 ListView 的父类 AbsListView 中，AbsListView 是一个列表控件的抽象。

```java
public abstract class AbsListView extends AdapterView<ListAdapter> implements TextWatcher,
        ViewTreeObserver.OnGlobalLayoutListener, Filter.FilterListener,
        ViewTreeObserver.OnTouchModeChangeListener,
        RemoteViewsAdapter.RemoteAdapterConnectionCallback {
    ...
    ListAdapter mAdapter;
    ...
    // 关联到 Window 时调用，获取调用 Adapter 中的 getCount 方法等
    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();

        final ViewTreeObserver treeObserver = getViewTreeObserver();
        treeObserver.addOnTouchModeChangeListener(this);
        if (mTextFilterEnabled && mPopup != null && !mGlobalLayoutListenerAddedFilter) {
            treeObserver.addOnGlobalLayoutListener(this);
        }
        // 给适配器注册一个观察者
        if (mAdapter != null && mDataSetObserver == null) {
            mDataSetObserver = new AdapterDataSetObserver();
            mAdapter.registerDataSetObserver(mDataSetObserver);

            // Data may have changed while we were detached. Refresh.
            mDataChanged = true;
            mOldItemCount = mItemCount;
            // 获取 Item 的数量，调用的是 mAdapter 的 getCount 方法
            mItemCount = mAdapter.getCount();
        }
    }

    ...

    // 子类需要覆写 layoutChildren 函数来布局 child view，也就是 item view
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);

        mInLayout = true;

        final int childCount = getChildCount();
        if (changed) {
            for (int i = 0; i < childCount; i++) {
                getChildAt(i).forceLayout();
            }
            mRecycler.markChildrenDirty();
        }

        // 布局 Child View
        layoutChildren();

        mOverscrollMax = (b - t) / OVERSCROLL_LIMIT_DIVISOR;

        // TODO: Move somewhere sane. This doesn't belong in onLayout().
        if (mFastScroll != null) {
            mFastScroll.onItemCountChanged(getChildCount(), mItemCount);
        }
        mInLayout = false;
    }
    ...
}
```

ListView 实现了 layoutChildren 方法，代码如下：

```java
    @Override
    protected void layoutChildren() {
        ...

        try {
            super.layoutChildren();

            invalidate();

            ...
            // 根据布局模式来布局 item view
            switch (mLayoutMode) {
            ...
            case LAYOUT_FORCE_TOP:
            case LAYOUT_FORCE_BOTTOM:
            case LAYOUT_SPECIFIC:
            case LAYOUT_SYNC:
                break;
            case LAYOUT_MOVE_SELECTION:
            default:
                ...
            }
        ...
    }
```

ListView 覆写了 AbsListView 中的 layoutChildren 函数，在该函数中根据布局模式来布局 item view，例如，默认情况是从上到下开始布局，但是，也有从下到上开始布局的，例如 QQ 聊天窗口的气泡布局，最新的消息就会布局到窗口的最底部。

fillDown 从上往下填充 item view。

```java
    private View fillDown(int pos, int nextTop) {
        View selectedView = null;

        int end = (mBottom - mTop);
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            end -= mListPadding.bottom;
        }

        while (nextTop < end && pos < mItemCount) {
            // is this the selected item?
            boolean selected = pos == mSelectedPosition;
            // 通过 makeAndAddView 获取 item view
            View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

            nextTop = child.getBottom() + mDividerHeight;
            if (selected) {
                selectedView = child;
            }
            pos++;
        }

        setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
        return selectedView;
    }
```

fillUp 从下往上填充布局。

```java
    private View fillUp(int pos, int nextBottom) {
        View selectedView = null;

        int end = 0;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            end = mListPadding.top;
        }

        while (nextBottom > end && pos >= 0) {
            // is this the selected item?
            boolean selected = pos == mSelectedPosition;
            // 通过 makeAndAddView 获取 item view
            View child = makeAndAddView(pos, nextBottom, false, mListPadding.left, selected);
            nextBottom = child.getTop() - mDividerHeight;
            if (selected) {
                selectedView = child;
            }
            pos--;
        }

        mFirstPosition = pos + 1;
        setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
        return selectedView;
    }
```
makeAndAddView 方法如下：

```java
    private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
            boolean selected) {
        if (!mDataChanged) {
            // Try to use an existing view for this position.
            final View activeView = mRecycler.getActiveView(position);
            if (activeView != null) {
                // Found it. We're reusing an existing child, so it just needs
                // to be positioned like a scrap view.
                setupChild(activeView, position, y, flow, childrenLeft, selected, true);
                return activeView;
            }
        }

        // 获取一个 item view
        // Make a new view for this position, or convert an unused view if
        // possible.
        final View child = obtainView(position, mIsScrap);

        // 将 item view 设置到对应的地方
        // This needs to be positioned and measured.
        setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

        return child;
    }
```

makeAndAddView 分为两个步骤，第一个是根据 position 获取一个 item view，然后将这个 view 布局到特定的位置。获取一个 item view 调用的是 obtainView 方法。这个方法在 AbsListView 中。

obtainView 方法如下：

```java
    View obtainView(int position, boolean[] outMetadata) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "obtainView");

        outMetadata[0] = false;

        // Check whether we have a transient state view. Attempt to re-bind the
        // data and discard the view if we fail.
        final View transientView = mRecycler.getTransientStateView(position);
        if (transientView != null) {
            final LayoutParams params = (LayoutParams) transientView.getLayoutParams();

            // If the view type hasn't changed, attempt to re-bind the data.
            if (params.viewType == mAdapter.getItemViewType(position)) {
                final View updatedView = mAdapter.getView(position, transientView, this);

                // If we failed to re-bind the data, scrap the obtained view.
                if (updatedView != transientView) {
                    setItemViewLayoutParams(updatedView, position);
                    mRecycler.addScrapView(updatedView, position);
                }
            }

            outMetadata[0] = true;

            // Finish the temporary detach started in addScrapView().
            transientView.dispatchFinishTemporaryDetach();
            return transientView;
        }
        // 1. 从缓存的 item view 中获取，ListView 的复用机制就在这里
        final View scrapView = mRecycler.getScrapView(position);
        // 2. 注意，这里将 scrapView 设置给了 Adapter 的 getView 函数
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                // Failed to re-bind the data, return scrap to the heap.
                mRecycler.addScrapView(scrapView, position);
            } else if (child.isTemporarilyDetached()) {
                outMetadata[0] = true;

                // Finish the temporary detach started in addScrapView().
                child.dispatchFinishTemporaryDetach();
            }
        }
        ...
        return child;
    }
```

obtainView 方法定义了列表控件的 item view 的复用逻辑，首先会从 RecyclerBin 中获取一个缓存的 View。如果有缓存则将这个缓存的 View 传递到 Adapter 的 getView 的第二个参数中，这也就是我们对 Adapter 的最常见的优化方式，即判断 getView 的 convertView 是否为空。如果为空则从 xml 中创建视图，否则使用缓存的 View。这样避免了每次都从 xml 加载布局的消耗，能够显著提升 ListView 的效率。

在 ListView 的适配器模式中，target 角色就是 View，Adapter 就是将 item view 输出为 view 抽象的角色，adaptee 就是需要被处理的 item view。通过增加 adapter 一层来将 item view 的操作抽象起来，listView 等集合视图通过 adapter 对象获得 item 的个数、数据、item view 等，从而达到适配各种数据、各种 item 视图的效果。