# ButterKnife View 注解框架介绍

ButterKnife 是 Android 系统的 View 注入框架，它可以减少大量的 findViewById 以及 setOnClickListener 代码，简化代码并提升开发效率。

但是，Google 官方提倡的 View Binding (MVVM) 架构也可以起到相同的作用。

正如 [ButterKnife](https://github.com/JakeWharton/butterknife) 的作者所说：

```
Attention: Development on this tool is winding down. Please consider switching to view binding in the coming months.
```

大致意思是 ButterKnife 的开发将变缓，后续请考虑迁移到 [view binding 架构](https://developer.android.google.cn/topic/libraries/view-binding)。

## 添加 ButterKnife 依赖

在 app 模块的 build.gradle 文件添加依赖如下：

```
dependencies {
...
    // butterknife
    implementation 'com.jakewharton:butterknife:10.2.0'
    annotationProcessor 'com.jakewharton:butterknife-compiler:10.2.0'
}
```

## 绑定控件

使用 @BindView 绑定控件，代码如下：

```java
public class ButterKnifeActivity extends Activity {

    @BindView(R.id.butter_text)
    TextView textView;

    @BindString(R.string.app_name)
    String appName;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_butter_knife);
        ButterKnife.bind(this);
        textView.setText(appName);
    }
}
```

注意这里的 textView 不能定义成 private 或者 static ，否则会编译报错：

```
错误: @BindView fields must not be private or static. 
```

也可以使用 @BindViews 绑定多个控件 id，用数组的形式装载 view。代码如下：

```java
public class ButterKnifeActivity extends Activity {

    @BindViews({R.id.butter_text, R.id.butter_text2})
    List<TextView> textViews;

    @BindString(R.string.app_name)
    String appName;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_butter_knife);
        ButterKnife.bind(this);
        textViews.get(0).setText(appName);
        textViews.get(1).setText(appName + "-2");
    }
}
```

## 绑定资源

ButterKnife 可以绑定资源：

- BindString 绑定字符串
- BindArray 绑定字符串数组
- BindBool 绑定布尔值
- BindColor 绑定颜色
- BindDimen 绑定距离大小
- BindDrawable 绑定 Drawable
- BindBitmap 绑定 Bitmap

以 BindString 为例，BindString 可以绑定字符串，而不用 getString 方法，代码如下：

```java
public class ButterKnifeActivity extends Activity {
...
    @BindString(R.string.app_name)
    String appName;
...
}
```

## 绑定监听事件

ButterKnife 可以绑定监听事件：

- onClick 绑定点击事件
- onLongClick 绑定长按点击
- onItemClick 绑定列表项点击
- onTouch 绑定触摸事件
- onTextChanged 绑定 ExitText 的文字变化事件

onClick 使用如下：

```java
    @OnClick(R.id.butter_button1)
    public void showToast() {
        Toast.makeText(this, "onClick", Toast.LENGTH_SHORT).show();
    }
```

onLongClick 使用如下：

```java
    @OnLongClick(R.id.butter_button2)
    public boolean showToast2() {
        Toast.makeText(this, "onLongClick", Toast.LENGTH_SHORT).show();
        return true;
    }
```

onItemClick 使用如下：

```java
    @OnItemClick(R.id.butter_lv)
    public void showToast3(int position) {
        Toast.makeText(this, "OnItemClick:" + position, Toast.LENGTH_SHORT).show();
    }
```

## 可选绑定

@BindView 或者其他的注解操作，如果不能找到目标资源，则会引发异常，为了防止异常，可以添加 @Nullable 注解：

```java
    @Nullable
    @BindView(R.id.butter_lv)
    ListView listView;
```

## 在 Fragment 和 Adapter 中使用 ButterKnife

除了在 Activity 中使用 ButterKnife，还可以在 Fragment 和 Adapter 中使用 ButterKnife。

在 Fragment 中使用 ButterKnife 如下：

```java
    @BindView(R.id.frag_text)
    TextView text;
...
    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_rx_bus, container, false);
        ButterKnife.bind(view);
        return view;
    }
```

也可以在 Adapter 的 ViewHolder 中绑定控件资源，使用方法和 Fragment 类似。

## 总结

1. ButterKnife View 注入框架可以绑定字符串、颜色、距离、图片的等资源
2. ButterKnife 可以绑定点击事件和各种控件的监听事件
3. ButterKnife 除了在 Activity 使用，也可以在 Fragment、Adapter ViewHolder、View 中使用