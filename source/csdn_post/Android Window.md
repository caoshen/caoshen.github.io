# Android Window

Window 是一个抽象类，它的实现是 PhoneWindow。Activity 的 setContentView 方法实际上就是通过 PhoneWindow 完成的。一般使用 WindowManager 来添加 View。WindowManager 是客户端用来添加 View 的接口，WindowManager 在 framework 层对应的是 WindowManagerService。

## 使用 WindowManager 添加 View

首先构造好需要添加的 View 和 LayoutParams，然后通过 WindowManager 的 addView 方法添加。

```java
    private void addWindow() {
        Button button = new Button(this);
        button.setText("button");
        WindowManager windowManager = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
        WindowManager.LayoutParams params = new WindowManager.LayoutParams();
        params.width = WindowManager.LayoutParams.WRAP_CONTENT;
        params.height = WindowManager.LayoutParams.WRAP_CONTENT;
        params.x = 100;
        params.y = 300;
        params.gravity = Gravity.START | Gravity.TOP;
        params.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE | WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
                | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED;
        params.type = WindowManager.LayoutParams.TYPE_SYSTEM_OVERLAY;
        windowManager.addView(button, params);
    }
```

这里介绍一下 WindowManager.LayoutParams 的几个属性。

width 表示 View 的宽度；

height 表示 View 的高度；

x 表示 View 的 x 轴相对坐标；

y 表示 View 的 y 轴相对坐标；

gravity 表示 View 的对齐方式，比如左对齐，上对齐。Gravity.Start 表示以开始的一边对齐，如果是从左向右（ltr）的阅读顺序，就是左对齐，如果是从右向左（rtl）的阅读顺序，就是右对齐。

flags 表示 Window 的标记。比如 FLAG_NOT_FOCUSABLE，开启后当前 Window 就收不到输入事件，事件会传给下方的 Window。同时它也会开启 FLAG_NOT_TOUCH_MODAL 标记。

```java
        /** Window flag: this window won't ever get key input focus, so the
         * user can not send key or other button events to it.  Those will
         * instead go to whatever focusable window is behind it.  This flag
         * will also enable {@link #FLAG_NOT_TOUCH_MODAL} whether or not that
         * is explicitly set.
         *
         * <p>Setting this flag also implies that the window will not need to
         * interact with
         * a soft input method, so it will be Z-ordered and positioned
         * independently of any active input method (typically this means it
         * gets Z-ordered on top of the input method, so it can use the full
         * screen for its content and cover the input method if needed.  You
         * can use {@link #FLAG_ALT_FOCUSABLE_IM} to modify this behavior. */
        public static final int FLAG_NOT_FOCUSABLE      = 0x00000008;
```

FLAG_NOT_FOCUSABLE 标记表示在 Window 范围以外的事件会发送给下方的 Window。

```java
        /** Window flag: even when this window is focusable (its
         * {@link #FLAG_NOT_FOCUSABLE} is not set), allow any pointer events
         * outside of the window to be sent to the windows behind it.  Otherwise
         * it will consume all pointer events itself, regardless of whether they
         * are inside of the window. */
        public static final int FLAG_NOT_TOUCH_MODAL    = 0x00000020;
```

FLAG_SHOW_WHEN_LOCKED 表示 Window 可以在锁屏的时候也显示。这个标记必须用作在顶层全屏的 Window，而且已经被标记为过时的（@deprecated）。

```java
        /** Window flag: special flag to let windows be shown when the screen
         * is locked. This will let application windows take precedence over
         * key guard or any other lock screens. Can be used with
         * {@link #FLAG_KEEP_SCREEN_ON} to turn screen on and display windows
         * directly before showing the key guard window.  Can be used with
         * {@link #FLAG_DISMISS_KEYGUARD} to automatically fully dismisss
         * non-secure keyguards.  This flag only applies to the top-most
         * full-screen window.
         * @deprecated Use {@link android.R.attr#showWhenLocked} or
         * {@link android.app.Activity#setShowWhenLocked(boolean)} instead to prevent an
         * unintentional double life-cycle event.
         */
        @Deprecated
        public static final int FLAG_SHOW_WHEN_LOCKED = 0x00080000;
```

type 表示 Window 的类型。TYPE_SYSTEM_OVERLAY 表示系统级别的 Window。它会显示在其他内容的上面。需要注意的是，使用这个类型必须声明权限，否则会抛出异常。它已经被标记为过时的（@deprecated）。

```java
        /**
         * Window type: system overlay windows, which need to be displayed
         * on top of everything else.  These windows must not take input
         * focus, or they will interfere with the keyguard.
         * In multiuser systems shows only on the owning user's window.
         * @deprecated for non-system apps. Use {@link #TYPE_APPLICATION_OVERLAY} instead.
         */
        @Deprecated
        public static final int TYPE_SYSTEM_OVERLAY     = FIRST_SYSTEM_WINDOW+6;
```

```java
Caused by: android.view.WindowManager$BadTokenException: Unable to add window android.view.ViewRootImpl$W@4d1c51d -- permission denied for window type 2006
```

FIRST_SYSTEM_WINDOW 的值是 2000，抛出的 2006 异常就是 TYPE_SYSTEM_OVERLAY 类型的权限被拒绝导致。

从 Android O（26） 版本开始，需要使用 TYPE_APPLICATION_OVERLAY 类型的 Window，并且在 AndroidManifest 加上 android.permission.SYSTEM_ALERT_WINDOW 权限。这个权限属于特殊权限，必须引导用户手动去设置的应用权限里面开启悬浮窗权限。

```java
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
```
