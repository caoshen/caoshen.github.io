Android 布局 ViewGroup 置灰

如果对 button 控件 setEnabled(false)，会把 button 控件置灰，同时使得 button 无法点击。

如果需要置灰（enable false）的控件是一个 ViewGroup，不仅需要对 ViewGroup setEnabled(false)，而且要对它的每个子 view 都 setEnabled(false)。

ViewGroup setEnabled 代码如下：

```java
    public static void setViewAndChildrenEnabled(View view, boolean isEnabled) {
        view.setEnabled(isEnabled);
        if (view instanceof ViewGroup) {
            ViewGroup viewGroup = (ViewGroup) view;
            int childCount = viewGroup.getChildCount();
            for (int i = 0; i < childCount; ++i) {
                View child = viewGroup.getChildAt(i);
                setViewAndChildrenEnabled(child, isEnabled);
            }
        }
    }
```

可以看出 setViewAndChildrenEnabled 方法采用递归的方式对它的每一个子视图置灰。因为 View 本身是一个树形结构，所以可以采用递归的方式。

首先会调用 view 的 setEnabled 方法。如果它同时是一个 ViewGroup(ViewGroup 继承 View)，对它的每个子 View 递归调用 setViewAndChildrenEnabled 方法。