# Android 主线程与Looper.loop() 循环

Android中为什么主线程不会因为Looper.loop()里的死循环卡死？这是一个知乎上的问题，链接如下：

https://www.zhihu.com/question/34652589

要回答这个问题，需要涉及一些知识，比如 Looper、Handler、AcitivityThread、Binder 等。

## 主线程消息循环

众所周知，在 ActivityThread 类的 main 方法中，通过 `Looper.prepareMainLooper();`准备主线程的 looper，通过 `Looper.loop();` 开启消息循环。假设主线程中的消息循环不是无限循环，而是过一段时间退出了，那么会导致程序退出，功能异常，就像当前正在前台交互的界面突然消失，这显然不是想看到的场景。因此开启一个无限循环，保证程序一直运行，如果有收到消息就处理，如果没有收到消息就休眠，直到被下一个消息唤醒。

真正导致主线程卡住的是处理某一个消息耗时太长，导致阻塞了其他消息，其他消息得不到处理，出现 ANR 问题（应用不响应 Application Not Response）。比如在 Activity 的生命周期 onCreate、onStart、onResume 处理非常耗时的任务，然后界面卡住，响应不了用户的交互事件（点击、滑动等动作）。

## 是谁发消息给主线程的 looper

ActivityManagerService(AMS) 通过 ActivityThread 里面的 ApplicationThread，发送消息到 ActivityThread，从而实现一些组件生命周期的回调，比如 Activity 的 onCreate、onResume 等。

ActivityThread 内部有一个 Handler，Handler 的名称就叫 H（一个大写的 H 来命名 ActivityThread 的主线程 Handler）。ApplicationThread 就是通过 H 发送消息到主线程的 looper，实现生命周期回调方法。所以说 onCreate、onResume 等方法是在主线程调用的。
