# Android 源码的桥接模式

## 桥接模式介绍

桥接模式（Bridge Pattern）也称为桥梁模式，是结构型设计模式之一。桥接模式承担着连接两边的作用。

## 桥接模式的定义

将抽象部分与实现部分分离，使它们都可以独立地进行变化。

## Android 源码中的桥接模式实现

Framework 内部的源码实现中，比较典型的桥接模式应用是 Window 与 WindowManager 之间的关系。

在 fwk 中 Window 和 PhoneWindow 构成窗口的抽象部分，其中 Window 类为该抽象部分的抽象接口，PhoneWindow 为抽象部分具体的实现及扩展。而 WindowManager 则为实现部分的基类，WindowManagerImpl 为实现部分具体的逻辑实现，其使用 WindowManagerGlobal 通过 IWindowManager 接口与 WindowManagerService （也就是 WMS）进行交互，并由 WMS 完成具体的窗口管理工作。


```java
public abstract class Window {
    ...
    /**
     * Set the window manager for use by this Window to, for example,
     * display panels.  This is <em>not</em> used for displaying the
     * Window itself -- that must be done by the client.
     *
     * @param wm The window manager for adding new windows.
     */
    public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
            boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        // 默认关闭硬件加速
        mHardwareAccelerated = hardwareAccelerated
                || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        // 如果传入的 WindowManager 为空，则通过 getSystemService 获取 WindowManager。
        if (wm == null) {
            wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        // 将 WindowManager 和 Window 绑定，创建一个 WindowManagerImpl
        mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    }
}
```

## 关于 WindowManagerService

Android 的 framework 层主要就是由 WindowManagerService 与另外一个系统服务 ActivityManagerService（简称 AMS）还有 View 所构成，这3个模块穿插交互在整个 framework 中，掌握了它们之间的关系以及每一个步骤的逻辑，你对 framework 就至少了解百分之五十了。

和很多其他系统服务一样，WMS 也是由 SystemServer 启动。

SystemServer 启动 WMS

```java
    private void startOtherServices() {
        ...
            wm = WindowManagerService.main(context, inputManager,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                    !mFirstBoot, mOnlyCore, new PhoneWindowManager());
            ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);
```

此后其他进程就可以通过用 ServiceManager 查询 window 来获取 WMS。WMS 的 main 方法如下：

```java
    public static WindowManagerService main(final Context context, final InputManagerService im,
            final boolean haveInputMethods, final boolean showBootMsgs, final boolean onlyCore,
            WindowManagerPolicy policy) {
        DisplayThread.getHandler().runWithScissors(() ->
                sInstance = new WindowManagerService(context, im, haveInputMethods, showBootMsgs,
                        onlyCore, policy), 0);
        return sInstance;
    }
```

调用 runWithScissors 构造了 WMS 实例。

WMS 的主要功能可以分为两方面，一是对窗口的管理；二是对事件的管理和分发。其接口方法以 AIDL 的方式定义在 IWindowManager.aidl 中，编译后会生成一个 IWindowManager.java 接口文件。


```java
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {
...
    /**
     * List of window tokens that have finished starting their application,
     * and now need to have the policy remove their windows.
     */
     // 已经完成启动的窗口
    final ArrayList<AppWindowToken> mFinishedStarting = new ArrayList<>();

    /**
     * List of app window tokens that are waiting for replacing windows. If the
     * replacement doesn't come in time the stale windows needs to be disposed of.
     */
     // 等待替换的窗口
    final ArrayList<AppWindowToken> mWindowReplacementTimeouts = new ArrayList<>();

    /**
     * Windows that are being resized.  Used so we can tell the client about
     * the resize after closing the transaction in which we resized the
     * underlying surface.
     */
     // 尺寸正在改变的窗口
    final ArrayList<WindowState> mResizingWindows = new ArrayList<>();
...
}
```

WMS 维护上述的各个成员变量值，可以看到大量线性表的应用，不同的窗口或同一个窗口在不同的状态阶段有可能位于不同的表中。

Android 的窗口类型主要只有 2 种：

1. 应用窗口。Activity 所处的窗口、应用对话框窗口、应用弹出窗口等都属于该类。与应用窗口相关的 Window 类主要是 PhoneWindow，其主要应用于手机，PhoneWindow 继承于 Window，其核心是 DecorView。应用窗口的添加主要就是通过 WindowManager 的 addView 方法将一个 DecorView 添加到 WindowManager 中。
2. 系统窗口。，常见的屏幕顶部的状态栏、底部的导航栏、桌面窗口等都是系统窗口，系统窗口没有针对性的封装类，只需直接通过WindowManager 的 addView 方法将一个 View 添加至 WindowManager 中即可。

以屏幕顶部的状态栏为例，其添加逻辑在 StatusBar.java 的 addStatusBarWindow 中。


```java
    private void addStatusBarWindow() {
        // 构造具体的状态栏 View，也就是下面会用到的 mStatusBarWindow，其本质上是一个 FrameLayout
        makeStatusBarView();
        mStatusBarWindowManager = Dependency.get(StatusBarWindowManager.class);
        mRemoteInputManager.setUpWithPresenter(this, mEntryManager, this,
                new RemoteInputController.Delegate() {
                    public void setRemoteInputActive(NotificationData.Entry entry,
                            boolean remoteInputActive) {
                        mHeadsUpManager.setRemoteInputActive(entry, remoteInputActive);
                        entry.row.notifyHeightChanged(true /* needsAnimation */);
                        updateFooter();
                    }
                    public void lockScrollTo(NotificationData.Entry entry) {
                        mStackScroller.lockScrollTo(entry.row);
                    }
                    public void requestDisallowLongPressAndDismiss() {
                        mStackScroller.requestDisallowLongPress();
                        mStackScroller.requestDisallowDismiss();
                    }
                });
        mRemoteInputManager.getController().addCallback(mStatusBarWindowManager);
        mStatusBarWindowManager.add(mStatusBarWindow, getStatusBarHeight());
    }
```

addView 实质上是由 WindowManagerGlobal 的 addView 方法实现具体逻辑，辗转多次后最终会调用到 ViewRootImpl 的 setView 方法。在该方法中通过 addToDisplay 方法向 WMS 发起一个 Session 请求，这里要注意的是，IWindowSession 也是一个 AIDL 接口文件，需要将其编译后才生成 IWindowSession.java 接口，这里 addToDisplay 方法最终会调用到 Session 中的对应方法：

addToDisplay

```java
    @Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
            Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
                outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel);
    }
```

addToDisplay 会调用 WMS 的 addWindow 添加窗口。


```java
    public int addWindow(Session session, IWindow client, int seq,
            LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
        int[] appOp = new int[1];

        // 检查窗口权限，能否添加窗口
        int res = mPolicy.checkAddPermission(attrs, appOp);
        if (res != WindowManagerGlobal.ADD_OKAY) {
            return res;
        }

       ...
       // 根据不同的窗口类型，检查窗口的有效性
            if (token == null) {
                if (rootType >= FIRST_APPLICATION_WINDOW && rootType <= LAST_APPLICATION_WINDOW) {
                    Slog.w(TAG_WM, "Attempted to add application window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_INPUT_METHOD) {
                    Slog.w(TAG_WM, "Attempted to add input method window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_VOICE_INTERACTION) {
                    Slog.w(TAG_WM, "Attempted to add voice interaction window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_WALLPAPER) {
                    Slog.w(TAG_WM, "Attempted to add wallpaper window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_DREAM) {
                    Slog.w(TAG_WM, "Attempted to add Dream window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_QS_DIALOG) {
                    Slog.w(TAG_WM, "Attempted to add QS dialog window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (rootType == TYPE_ACCESSIBILITY_OVERLAY) {
                    Slog.w(TAG_WM, "Attempted to add Accessibility overlay window with unknown token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type == TYPE_TOAST) {
                    // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                    if (doesAddToastWindowRequireToken(attrs.packageName, callingUid,
                            parentWindow)) {
                        Slog.w(TAG_WM, "Attempted to add a toast window with unknown token "
                                + attrs.token + ".  Aborting.");
                        return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                    }
                }
            ...
            // 构造 WindowState 对象
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid,
                    session.mCanAddInternalSystemWindow);
            ...
            // 将窗口添加至 Session
            win.attach();
            // 将窗口添加至 WMS
            mWindowMap.put(client.asBinder(), win);
            ...
            // 打开输入通道，以便当前窗口可以接收到事件
            final boolean openInputChannels = (outInputChannel != null
                    && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
            if  (openInputChannels) {
                win.openInputChannel(outInputChannel);
            }
            ...
            // 如果当前窗口可以接收按键事件，那么更新焦点，将窗口信息存入 InputDispatcher
            boolean focusChanged = false;
            if (win.canReceiveKeys()) {
                focusChanged = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                        false /*updateInputWindows*/);
                if (focusChanged) {
                    imMayMove = false;
                }
            }

            if (imMayMove) {
                displayContent.computeImeTarget(true /* updateImeTarget */);
            }

            // Don't do layout here, the window must call
            // relayout to be displayed, so we'll do it there.
            win.getParent().assignChildLayers();

            if (focusChanged) {
                mInputMonitor.setInputFocusLw(mCurrentFocus, false /*updateInputWindows*/);
            }
            mInputMonitor.updateInputWindowsLw(false /*force*/);

            if (localLOGV || DEBUG_ADD_REMOVE) Slog.v(TAG_WM, "addWindow: New client "
                    + client.asBinder() + ": window=" + win + " Callers=" + Debug.getCallers(5));

            if (win.isVisibleOrAdding() && updateOrientationFromAppTokensLocked(displayId)) {
                reportNewConfig = true;
            }
        }

        if (reportNewConfig) {
            sendNewConfiguration(displayId);
        }

        Binder.restoreCallingIdentity(origId);

        return res;
    }

```

addWindow 方法中会通过 win.attach 创建一个管理的 SurfaceSession 对象用来和 SurfaceFlinger 通信，而 SurfaceSession 实际对应 native 层的 SurfaceComposerClient 对象，SurfaceComposerClient 主要作用就是在应用进程与 SurfaceFlinger 服务之间建立连接。

## 参考资料

《Android 源码设计模式解析与实战 · 第24章 连接两地的交通枢钮——桥接模式》