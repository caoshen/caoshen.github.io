
# Android 中的 IPC 方式

Android 中跨进程通信的方式有 intent 传入 bundle，共享文件，Messenger，AIDL，ContentProvider，Socket。

## Bundle

四大组件中的三大组件（Activity\Service\BroadcastReceiver）都是支持在 intent 中传递 Bundle 数据的。由于 Bundle　实现了 Parcelable 接口，所以它可以方便地在不同的进程间传输。传输的数据必须能够被序列化，比如基本类型，实现了 Parcelable 接口的对象，实现了 Serializable 接口的对象以及一些 Android 支持的特殊类型。

## 文件共享

共享文件也是一种不错的进程间通信方式，两个进程通过读写同一个文件来交换数据。比如 A 进程把数据写入文件，B 进程通过读取这个文件来获取数据。由于 Android 基于 Linux，使得并发读写文件可以没有限制地进行，甚至两个线程同时对同一个文件进行写操作都是允许的。

通过文件共享的方式是有局限性的。如果并发读写，读出来的内容可能不是最新的。如果并发写，可能内容就错乱了。因此需要尽量避免并发写的情况发生，或者使用线程同步来限制多线程的写操作。

SharedPreferences 是 Android 轻量级存储方案，它以键值对的形式存放在 xml 中。从本质上来说，SharedPreferences 也属于文件的一种，但是系统对它的读写有缓存策略，在内存中也有一份 SharedPreferences 的缓存。所以在多进程模式下，高并发读写会丢失数据，不建议在进程间通信中使用。

## Messenger

Messenger 的底层实现是 AIDL。

```java
public Messenger(Handler target) {
    mTarget = target.getIMessenger;
}

public Messenger(IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```

Messenger 对 AIDL 做了封装，使得可以更简便地进行进程间通信。由于它一次处理一个请求，所以在服务端不用考虑线程同步的问题。

1. 服务端进程：
创建一个 Handler，并通过它来创建 Messenger。在 Service 的 onBind 方法中返回 Messenger 的 IBinder。
2. 客户端进程：
用绑定成功后服务端返回的 IBinder 创建一个 Messenger，然后通过 Messenger 向服务端发消息（Message）。

如果需要服务端向客户端发消息，那么就和服务端一样，在客户端创建一个 Handler，并通过它来创建 Messenger。然后把 Messenger 通过 Message 的 replyTo 属性传递到服务端。服务端拿到客户端的 Messenger 后，就可以向客户端发消息了。

## AIDL

如果有大量请求同时从客户端发到服务端处理，就使用 AIDL。AIDL 的使用方式很简单，先创建 AIDL 文件，转换成 Binder 类后，服务端 onBind 返回 IBinder 对象，客户端调用 xxx.Stub.asInterface(binder) 方法获取接口对象，最后调用接口方法。

## ContentProvider

ContentProvider 通过 Binder 实现跨进程通信。继承 ContentProvider 类，然后实现 onCreate\getType\query\insert\update\delete 方法。

onCreate 负责 ContentProvider 的创建;

getType　返回 Uri 请求对应的 MIME 类型，如图片、视频等。如果不关注这个类型可以直接返回 null;

query 查询数据;

insert 插入数据;

update 更新数据；

delete 删除数据。

ContentProvider 对底层的数据存储方式没有任何要求，既可以使用 SQLite 数据库，也可以使用普通的文件，甚至可以采用内存中的对象来存储数据。

## Socket

也可以使用 Socket 来实现跨进程通信。在服务端新建 Socket 并监听请求。在客户端连接服务端的 Socket ，然后发送请求。因为需要联网，所以 AndroidManifest 要加入网络权限。






