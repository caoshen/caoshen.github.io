# Service 难点

## 如何停止 Service

如果先 startService，然后 bindService，如何停止这个 Service？

如果只是调用了 stopService，但是还有其他的地方 bindService，service 是停止不了的。必须保证所有 bindService 的地方都调用了 unBindService，然后再调用 stopService，才能停止 service。

## onStartCommand 返回值

onStartCommand 的返回值有 4 种。它的返回值代表了 Service 被系统 Kill 后，该如何继续 Service。

1. START_STICKY_COMPATIBILITY。START_STICKY_COMPATIBILITY 是 START_STICKY 的兼容版本。但是它无法保证 Service 被系统 Kill 后，onStartCommand 方法重新被调用。

2. START_STICKY。当 Service 被系统 Kill 后，onStartCommand 方法会被重新调用。但是 onStartCommand 的 intent 不会重新传递，会为空。

3. START_NOT_STICKY。当 Service 被系统 Kill 后，onStartCommand 方法不会被重新调用。需要再 startService 才能启动 Service。

4. START_REDELIVER_INTENT。当 Service 被系统 Kill 后，onStartCommand 方法会被重新调用。上一次的 intent 也会重新传递。

## ServiceConnection 的方法运行在哪个线程

ServiceConnection 的回调方法运行在主线程。

## ServiceConnection 的方法调用时机

1. void onServiceConnected(ComponentName name, IBinder service)。onServiceConnected 在与Service 建立连接时执行。这个方法也有可能不执行，比如当 Service 创建的过程中 crash 了。

2. void onServiceDisconnected(ComponentName name)。onServiceDisconnected 在 Service 连接丢失的情况下执行，比如 Service 所在进程 crash 或者被系统 kill 。

## onCreate 运行在哪个线程

onCreate 运行在主线程。

