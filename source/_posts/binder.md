---
title: Binder 笔记
date: 2019-02-03 15:57:57
tags:
---

# Binder 笔记

Binder 是 Android RPC 机制的核心基础类，它实现了 IBinder 接口。Android 中跨进程通信一般都涉及到了 Binder 调用。


## Binder 简介

[Binder](http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/android/os/Binder.java) 类位于 /frameworks/base/core/java/android/os/Binder.java 文件中。

大多数使用Binder的时候，不是直接实现Binder类，而是通过 AIDL(Android 接口描述语言) 文件描述接口方法，然后用 Android Studio 自动编译产生 AIDL 对应的 Binder 子类。当然，也可以不用 AIDL， 直接写一个继承 Binder 的类来使用。

因为 Binder 类是一个基础 IPC 原语，所以它对应用的生命周期没有影响，只要创建它的进程在运行，Binder 就是有效的。一般我们会在应用的顶级组件，比如 Servce、Activity、ContentProvider，以让系统知道 Binder 所在进程正在运行中。

如果 Binder 所在的进程停止了，当进程重启时，需要重新创建 Binder 并且绑定到进程。比如在 Activity 中使用 Binder 时，如果 Activity 没有被使用，Activity的进程可能被回收，那么当 Activity 重新启动时需要再创建 Binder。

## Binder 的使用场景

Binder 在 Android 广泛被使用。从 Framework 层来说，Binder 是　ServiceManager 连接各种 Manger(ActivityManager、WindowManager，等等)和相应的 ManagerService(ActivityManagerService、WindowMangerService，等等)的桥梁。从应用层的角度来说，Binder 是连接客户端（调用方）和服务端　（被调用方）的桥梁。当客户端 bindService 时，服务端会返回一个包含了服务端业务调用的 Binder 对象，通过这个 Binder 对象，客户端就可以获取服务端提供的服务或者数据。比如应用 A 想获取应用 B 的数据或者应用 B 需要开接口给应用 A，双方可以约定好 AIDL 接口，然后在应用 B 中完成服务和接口的实现，应用 A 通过 bindService 方法绑定 B 的服务获取 Binder，通过 Binder 来获取应用 B 的数据，这样就完成了一次应用间跨进程调用。

## Binder 的使用方式

这里以 AIDL 接口方式为例。

AIDL 方式只需要声明接口，大量重复的样板代码由工具自动在编译构建时为我们生产了对应的 Java 类。

假设应用 A 需要从应用 B 获取数据，那么此时相对而言 A 是客户端，B 是服务端。

在 B 中创建 AIDL 文件

```AIDL
// IMyAidlInterfaceB.aidl
package com.example.b.myapplicationb;

// Declare any non-default types here with import statements

interface IMyAidlInterfaceB {
    String getSomeData();
}
```

构建项目，生产对应的 IMyAidlInterfaceB.java。位于 app/build/generated/source/aidl 目录

创建需要暴露给应用 A 使用的服务 BMainService

```java
public class BMainService extends Service {
    private static final String DATA_FROM_B = "this is data from app b";
    private IBinder mBinder = new IMyAidlInterfaceB.Stub() {
        @Override
        public String getSomeData() throws RemoteException {
            return DATA_FROM_B;
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```
在 AndroidManifest 声明 Service

```
        <service
            android:name=".BMainService"
            android:exported="true">
            <intent-filter>
                <action android:name="intent.action.BMainService" />
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
        </service>
```

接下来把应用 B 的 AIDL 文件连同目录复制到应用 A 的 AIDL 目录中，然后在应用 A 调用 BMainService。

```java
public class MainActivity extends AppCompatActivity {

    public static final String INTENT_ACTION_BMAIN_SERVICE = "intent.action.BMainService";
    public static final String PACKAGE_NAME = "com.example.b.myapplicationb";
    private static final String TAG = MainActivity.class.getSimpleName();
    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d(TAG, "on service connected");
            try {
                IMyAidlInterfaceB iMyAidlInterfaceB = IMyAidlInterfaceB.Stub.asInterface(service);
                String someData = iMyAidlInterfaceB.getSomeData();
                mText.setText("data from B: " + someData);
            } catch (RemoteException e) {
                Log.e("MainActivity", "Remote err:" + e);
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };
    private Intent mIntent;
    private TextView mText;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mText = findViewById(R.id.text);

        mIntent = new Intent(INTENT_ACTION_BMAIN_SERVICE);
        mIntent.setPackage(PACKAGE_NAME);
        bindService(mIntent, mServiceConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        stopService(mIntent);
    }
}
```

通过绑定服务拿到 Binder，把 Binder 转化为相应接口调用服务端方法。这一步是通过 IMyAidlInterfaceB.Stub.asInterface 完成的。

```java
IMyAidlInterfaceB iMyAidlInterfaceB = IMyAidlInterfaceB.Stub.asInterface(service);
```

打印 logcat 日志，可以发现应用 A 通过调用应用 B 的服务启动了 B 的进程：
```shell
ActivityManager: Start proc 10658:com.example.b.myapplicationb/u0a139 for service com.example.b.myapplicationb/.BMainService caller=com.example.a.myapplicationa
```

如果是应用内通过 android:process 属性声明了应用内部的多进程，也可以用 AIDL 的方式调用不同进程的服务。

## Binder 代码分析

打开系统生成的 IMyAidlInterfaceB 文件，可以看出它继承了 IInterface 接口，同时它自身也是一个接口类。

```java
public interface IMyAidlInterfaceB extends android.os.IInterface
```

IInterface 接口只有一个方法，asBinder()，返回接口关联的 Binder 类。当需要获取 Service 的 Binder 时，不要直接做类型转换，而是通过 asBinder 方法获取 Binder。
　
```java
package android.os;
/**
 * Base class for Binder interfaces.  When defining a new interface,
 * you must derive it from IInterface.
 */
public interface IInterface
{
    /**
     * Retrieve the Binder object associated with this interface.
     * You must use this instead of a plain cast, so that proxy objects
     * can return the correct result.
     */
    public IBinder asBinder();
}
```

在 IMyAidlInterfaceB 中声明了自定义的接口方法，同时还有一个 Transaction ID，对应自定义的接口方法，用来指明在 transact 过程中客户端请求的是哪一个方法 ID。

接着，在 IMyAidlInterfaceB 类中声明了一个内部类 Stub，Stub 继承了 Binder 类。当客户端和服务端都位于同一个进程时，方法调用不走跨进程的 transact 过程，而是通过 queryLocalInterface() 方法并做类型转换然后返回一个接口类；当客户端和服务端位于不同的进程时，方法返回的是 Stub 类的内部类代理 Stub.Proxy(ojb)，方法调用会通过这个代理类完成 transact 过程。

下面开始分析 Stub 类组成：

```java
private static final java.lang.String DESCRIPTOR = "com.example.b.myapplicationb.IMyAidlInterfaceB";

```

DESCRIPTOR 常量 是 Binder 的唯一标识，一般用包名加类名表示，如 "com.example.b.myapplicationb.IMyAidlInterfaceB"。

```java
public static com.example.b.myapplicationb.IMyAidlInterfaceB asInterface(android.os.IBinder obj)
```

asInterface() 是 Stub 类的静态公有方法，用来把 Binder 转换成需要的自定义接口类型的对象，以便调用自定义接口方法。如果客户端和服务端位于同一个进程，那么此方法返回的就是服务端的 Stub 对象本身；如果不在同一个进程，那么返回的是系统封装后的 Stub.Proxy 对象。

```java
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
```

onTransact 方法运行在服务端的 Binder 线程池中。当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。服务端可以通过 code(方法标识) 来判断客户端请求的目标方法是什么，接着从 data(方法入参)中取出目标方法所需的参数，然后执行目标方法。当目标方法执行完毕后，把结果写入 reply(方法返回值)。

需要注意的是，如果此方法返回 false，那么客户端请求会失败，可以利用这个特性做权限验证。

```java
public java.lang.String getSomeData() throws android.os.RemoteException
```

getSomeData() 是接口自定义的方法，在 Proxy 代理类中。这个方法运行在客户端，当客户端远程调用此方法时，会把方法参数写入 _data(如果有参数)，然后调用 mRemote 的 transact 方法来发起远程过程调用，同时当前线程挂起。然后服务端的 onTransact 方法会被调用，直到远程过程调用完成并返回后，当前线程继续执行，从 _reply 中取出返回值并返回结果。

有两点需要注意：

1. 当客户端发起远程请求时，由于当前线程会被挂起直到服务端返回结果，所以如果一个远程方法很耗时，那么不能在 UI 线程中发起此远程请求；

2. 由于服务端的 Binder 方法运行在 Binder 的线程池中，所以 Binder 方法不管是不是耗时都应该采用同步的方式去实现，因为它已经运行在一个线程中了。

