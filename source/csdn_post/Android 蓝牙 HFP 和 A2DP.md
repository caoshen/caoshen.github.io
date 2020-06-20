# Android 蓝牙 HFP 和 A2DP

HFP（Hands Free Profile）和 A2DP (Advanced Audio Distribution Profile) 是经典蓝牙常用的两个协议。

HFP 协议一般用来支持耳机打电话、接电话、挂断电话、拒接电话等操作。

A2DP 协议一般用来听歌，传输音频立体声。

## ProfileProxy

Android SDK 使用 BluetoothAdapter 的 getProfileProxy 获取每个具体的 Profile。

获取 HFP：

```java
    public static boolean getHfpProfileProxy(Context context, BluetoothProfile.ServiceListener listener) {
        BluetoothAdapter defaultAdapter = BluetoothAdapter.getDefaultAdapter();
        boolean hfp = defaultAdapter.getProfileProxy(context, listener, BluetoothProfile.HEADSET);
        return hfp;
    }
```

获取 A2DP：

```java
    public static boolean getA2dpProfileProxy(Context context, BluetoothProfile.ServiceListener listener) {
        BluetoothAdapter defaultAdapter = BluetoothAdapter.getDefaultAdapter();
        boolean profileProxy = defaultAdapter.getProfileProxy(context, listener, BluetoothProfile.A2DP);
        return profileProxy;
    }
```

## HFP 连接

通过反射调用 BluetoothHeadset 的 connect 方法连接 HFP。

```java
    public static boolean connectHfp(BluetoothHeadset hfp, BluetoothDevice device) {
        boolean ret = false;
        Method connect = null;
        try {
            connect = BluetoothHeadset.class.getDeclaredMethod("connect", BluetoothDevice.class);
            connect.setAccessible(true);
            ret = (boolean) connect.invoke(hfp, device);
        } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
            e.printStackTrace();
        }
        return ret;
    }
```

## A2DP 连接

通过反射调用 BluetoothA2dp 的 connect 方法连接 A2DP。

```java
    public static boolean connectA2dp(BluetoothA2dp a2dp, BluetoothDevice device) {
        boolean ret = false;
        Method connect = null;
        try {
            connect = BluetoothA2dp.class.getDeclaredMethod("connect", BluetoothDevice.class);
            connect.setAccessible(true);
            ret = (boolean) connect.invoke(a2dp, device);
        } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
            e.printStackTrace();
        }
        return ret;
    }
```

## HFP 状态广播

通过 BluetoothHeadset 的 ACTION_CONNECTION_STATE_CHANGED 监听 HFP 连接状态变化。

```java
    /**
     * Intent used to broadcast the change in connection state of the Headset
     * profile.
     *
     * <p>This intent will have 3 extras:
     * <ul>
     * <li> {@link #EXTRA_STATE} - The current state of the profile. </li>
     * <li> {@link #EXTRA_PREVIOUS_STATE}- The previous state of the profile. </li>
     * <li> {@link BluetoothDevice#EXTRA_DEVICE} - The remote device. </li>
     * </ul>
     * <p>{@link #EXTRA_STATE} or {@link #EXTRA_PREVIOUS_STATE} can be any of
     * {@link #STATE_DISCONNECTED}, {@link #STATE_CONNECTING},
     * {@link #STATE_CONNECTED}, {@link #STATE_DISCONNECTING}.
     *
     * <p>Requires {@link android.Manifest.permission#BLUETOOTH} permission to
     * receive.
     */
    @SdkConstant(SdkConstantType.BROADCAST_INTENT_ACTION)
    public static final String ACTION_CONNECTION_STATE_CHANGED =
            "android.bluetooth.headset.profile.action.CONNECTION_STATE_CHANGED";
```

## A2DP 状态广播

通过 BluetoothA2dp 的 ACTION_CONNECTION_STATE_CHANGED 监听 A2DP 连接状态变化。

```java
    /**
     * Intent used to broadcast the change in connection state of the A2DP
     * profile.
     *
     * <p>This intent will have 3 extras:
     * <ul>
     * <li> {@link #EXTRA_STATE} - The current state of the profile. </li>
     * <li> {@link #EXTRA_PREVIOUS_STATE}- The previous state of the profile.</li>
     * <li> {@link BluetoothDevice#EXTRA_DEVICE} - The remote device. </li>
     * </ul>
     *
     * <p>{@link #EXTRA_STATE} or {@link #EXTRA_PREVIOUS_STATE} can be any of
     * {@link #STATE_DISCONNECTED}, {@link #STATE_CONNECTING},
     * {@link #STATE_CONNECTED}, {@link #STATE_DISCONNECTING}.
     *
     * <p>Requires {@link android.Manifest.permission#BLUETOOTH} permission to
     * receive.
     */
    @SdkConstant(SdkConstantType.BROADCAST_INTENT_ACTION)
    public static final String ACTION_CONNECTION_STATE_CHANGED =
            "android.bluetooth.a2dp.profile.action.CONNECTION_STATE_CHANGED";
```
