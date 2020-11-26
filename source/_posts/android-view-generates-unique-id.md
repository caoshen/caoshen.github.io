---
title: Android View 生成唯一 Id
date: 2020-08-31 23:06:48
tags: View
categories: Android
---

# Android View 生成唯一 Id

可以使用 Hook LayoutInflater 的方法替换 SystemService 原有的 LayoutInflater，在自定义的 LayoutInflater 遍历每一个 view，为它们生成 md5 作为 view 的唯一 id。

## Hook LayoutInflater

Hook LayoutInflater 的核心在于使用反射调用 registerService 方法，注册自定义的 LayoutInflater。

```java
public class LayoutInflaterHook {
    private static final String TAG = "LayoutInflaterHook";

    public static void hookLayoutInflater() throws Exception {
        Class<?> serviceFetcher = Class.forName("android.app.SystemServiceRegistry$ServiceFetcher");
        // 获取 ServiceFetcher 的实例 serviceFetcherImpl
        Object serviceFetcherImpl = Proxy.newProxyInstance(LayoutInflaterHook.class.getClassLoader(),
                new Class<?>[]{serviceFetcher},
                new ServiceFetcherHandler());

        // 获取 SystemServiceRegistry 的 registerService 方法
        Class<?> systemServiceRegistry = Class.forName("android.app.SystemServiceRegistry");

        // 无法反射调用 registerService，registerService 在 Android 10 版本以上都是 blacklist 级别的 api，
        // 反射调用会被系统拒绝，抛出 NoSuchMethodException。
        Method registerService = systemServiceRegistry.getDeclaredMethod("registerService",
                String.class,
                CustomLayoutInflater.class.getClass(),
                serviceFetcher);
        registerService.setAccessible(true);

        // 调用 registerService 方法，将自定义的 CustomLayoutInflater 设置到 SystemServiceRegistry
        registerService.invoke(systemServiceRegistry,
                Context.LAYOUT_INFLATER_SERVICE, CustomLayoutInflater.class, serviceFetcherImpl);

        // 测试
        Field systemServiceFetchers = systemServiceRegistry.getDeclaredField("SYSTEM_SERVICE_FETCHERS");
        systemServiceFetchers.setAccessible(true);
        Map systemServiceFetchersField = (Map) systemServiceFetchers.get(null);
        Set set = systemServiceFetchersField.keySet();
        Object service = systemServiceFetchersField.get(Context.LAYOUT_INFLATER_SERVICE);
        Log.w(TAG, "find layout inflater:" + service);
        for (Object next : set) {
            Object value = systemServiceFetchersField.get(next);
            Log.d(TAG, "key:" + next);
            Log.d(TAG,"value:" + value);
            if (Context.LAYOUT_INFLATER_SERVICE.equals(next)) {
                Log.e(TAG, "find Service for layout inflater:" + value);
            }
        }
    }

}

```

## 动态代理 ServiceFetcher

因为 ServiceFetcher 是 SystemServiceRegistry 的内部借口，因此需要使用动态代理的方式实现它的 invoke 方法。

```java
public class ServiceFetcherHandler implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 当调用 ServiceFetcherImpl 的 getService 的时候，会返回自定义的 LayoutInflater
        if ("toString".equals(method.getName())) {
            return "ServiceFetcherHandler";
        }
        return new CustomLayoutInflater((Context) args[0]);
    }
}
```

## 自定义 LayoutInflater

当调用 getService 时，会返回自定义的 CustomLayoutInflater。

```java
public class CustomLayoutInflater extends LayoutInflater {

    private static final String[] sClassPrefixList = {
            "android.widget.",
            "android.webkit."
    };

    private static int VIEW_TAG = 0x10000000;

    public CustomLayoutInflater(Context context) {
        super(context);
    }

    public CustomLayoutInflater(LayoutInflater original, Context newContext) {
        super(original, newContext);
    }

    @Override
    protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        for (String prefix : sClassPrefixList) {
            try {
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            } catch (InflateException e) {
                e.printStackTrace();
            }
        }
        return super.onCreateView(name, attrs);
    }

    @Override
    public LayoutInflater cloneInContext(Context newContext) {
        return new CustomLayoutInflater(this, newContext);
    }

    @Override
    public View inflate(int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        View viewGroup = super.inflate(resource, root, attachToRoot);
        View rootView = viewGroup;
        View tempView = viewGroup;
        // 向上遍历得到根 View
        while (tempView != null) {
            rootView = viewGroup;
            tempView = ((ViewGroup) tempView.getParent());
        }
        traversalViewGroup(rootView);
        return viewGroup;
    }

    private void traversalViewGroup(View rootView) {
        if (rootView == null || !(rootView instanceof ViewGroup)) {
            return;
        }
        // 如果 rootView 没有 tag，设置它的 view 值为 VIEW_TAG 计数值
        if (rootView.getTag() == null) {
            rootView.setTag(getViewTag());
        }
        ViewGroup viewGroup = (ViewGroup) rootView;
        int childCount = ((ViewGroup) rootView).getChildCount();
        for (int i = 0; i < childCount; ++i) {
            View childView = viewGroup.getChildAt(i);
            if (childView.getTag() == null) {
                childView.setTag(combineTag(getViewTag(), rootView.getTag().toString()));
            }
            Log.e("Hooker", "childView name=" + childView.getClass().getName()
                    + ", id = " + childView.getTag().toString());
            if (childView instanceof ViewGroup) {
                // 深度优先遍历
                traversalViewGroup(childView);
            }
        }
    }

    private static String combineTag(String tag1, String tag2) {
        return getMd5(getMd5(tag1) + getMd5(tag2));
    }

    private static String getViewTag() {
        return String.valueOf(VIEW_TAG++);
    }

    public static String getMd5(String str) {
        try {
            MessageDigest md5 = MessageDigest.getInstance("MD5");
            md5.update(str.getBytes());
            return new BigInteger(1, md5.digest()).toString(16);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        return "null";
    }
}
```

通过组合父 view 和自身 view 的 md5 得到唯一的 view id。

## Android 黑名单的限制

从 Android 9 开始，Android 系统限制了部分 System api 的调用。Hook LayoutInflater 用到的 registerService 方法就被列入了黑名单，反射调用直接抛出 NoSuchMethodException。因此 Hook LayoutInflater 的方法实际不可用，但是可以通过修改系统设置本地调试。

```
    adb shell settings put global hidden_api_policy  1
```

关于在 Android 10 中授予对非 SDK 接口的访问权限，可以查看 [Android 10 中有关限制非 SDK 接口的更新
](https://developer.android.google.cn/about/versions/10/non-sdk-q#enable-non-sdk-access)

## 源码

https://github.com/caoshen/AndroidEfficientAdvanced