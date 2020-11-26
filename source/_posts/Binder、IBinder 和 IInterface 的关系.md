# Binder、IBinder 和 IInterface 的关系

## IBinder 接口

Binder 实现了 IBinder 接口。

```java
public class Binder implements IBinder {
    ...
}
```

IBinder 接口定义了一些常量和方法。比如第一个交易码（transaction code）和最后一个交易码。

最小的交易码是 1，最大的交易码是 16 ^ 6 - 1 = 16777215。其他的交易码处于这两个值之间。可以看出最多有 16777215 个 transaction。

```java
    /**
     * The first transaction code available for user commands.
     */
    int FIRST_CALL_TRANSACTION  = 0x00000001;
    /**
     * The last transaction code available for user commands.
     */
    int LAST_CALL_TRANSACTION   = 0x00ffffff;
```

IBinder 接口也定义了一些方法，比如 isBinderAlive、queryLocalInterface、transact、linkToDeath、unlinkToDeath 等。

```java
    /**
     * Check to see if the process that the binder is in is still alive.
     *
     * @return false if the process is not alive.  Note that if it returns
     * true, the process may have died while the call is returning.
     */
    public boolean isBinderAlive();
    
    /**
     * Attempt to retrieve a local implementation of an interface
     * for this Binder object.  If null is returned, you will need
     * to instantiate a proxy class to marshall calls through
     * the transact() method.
     */
    public @Nullable IInterface queryLocalInterface(@NonNull String descriptor);

    public boolean transact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags)
        throws RemoteException;

    public void linkToDeath(@NonNull DeathRecipient recipient, int flags)
            throws RemoteException;

    public boolean unlinkToDeath(@NonNull DeathRecipient recipient, int flags);
```

IBinder 接口也定义了一个子接口，DeathRecipient，表示服务进程消失的回调。

```java
    /**
     * Interface for receiving a callback when the process hosting an IBinder
     * has gone away.
     * 
     * @see #linkToDeath
     */
    public interface DeathRecipient {
        public void binderDied();
    }
```

## IInterface 接口

自定义的 AIDL 接口继承了 IInterface 接口。

```java
public interface ICustomAidl extends android.os.IInterface {
    ...
}
```

IInterface 接口只有一个方法，asBinder。

```java
public IBinder asBinder();
```

asBinder() 方法用来返回 AIDL 接口里面的 Stub 类的对象。如果是同进程的本地接口，就是 this。否则是 BinderProxy 代理对象。

```java
/** Local-side IPC implementation stub class. */
public static abstract class Stub extends android.os.Binder implements com.caoshen.androidsample.binder.ICustomAidl
{
...

@Override public android.os.IBinder asBinder()
{
return this;
}
```

## asInterface 方法

asInterface 是 Stub 类里面的静态方法，用来返回自定义的 AIDL 接口。

```java
/**
 * Cast an IBinder object into an com.caoshen.androidsample.binder.ICustomAidl interface,
 * generating a proxy if needed.
 */
public static com.caoshen.androidsample.binder.ICustomAidl asInterface(android.os.IBinder obj)
{
if ((obj==null)) {
return null;
}
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
if (((iin!=null)&&(iin instanceof com.caoshen.androidsample.binder.ICustomAidl))) {
return ((com.caoshen.androidsample.binder.ICustomAidl)iin);
}
return new com.caoshen.androidsample.binder.ICustomAidl.Stub.Proxy(obj);
}
```

可以看出，如果 queryLocalInterface 得到的是本地接口，就强转成自定义的 AIDL 接口。否则把 Binder 自身传入 BinderProxy 作为 mRemote，返回一个 BinderProxy 对象。