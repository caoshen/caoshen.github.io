# Android蓝牙系统框架和代码结构

## 概述

在 Android 4.2版本中，谷歌公司和博通合作，引入了博通的 BTE/BTA 协议栈，重构了蓝牙子系统。新的蓝牙协议栈被命名为 BlueDroid。它包含了两层：BTE（完成蓝牙核心功能）和 BTA（与 Android 蓝牙服务层进行通信）。蓝牙服务层与 Bluedroid （封装了 BTIF 层）通过 JNI 进行通信，与上层应用通过 Binder IPC 进行通信。BlueZ 及配套框架在 Android 系统上被移除。

## Application Framework

该层代码主要是利用 android.bluetooth API 和蓝牙进程（Bluetooth Process）进行交互，也就是通过 Binder IPC 机制调用了蓝牙进程的各个服务（Service）封装的接口。代码位于 frameworks/base/core/java/android/bluetooth 下。

## Bluetooth Process

该层代码主要是在 Bluetooth Process 里实现各种 Bluetooth Service 和各种配置文件（Profile），Service 通过 JNI 调用到硬件抽象（HAL）层。代码最后编译形成一个 Android Application 包（Bluetooth.apk）。代码位于 packages/apps/Bluetooth 下。

## Bluetooth JNI

该层代码位于 packages/apps/bluetooth/jni 下，定义了蓝牙适配层和协议层对应的 JNI 服务，直接调用 HAL 层并给 HAL 层提供相应的回调。

## Bluetooth HAL

该层代码定义了 android.bluetooth API 和 Bluetooth Process 调用的标准接口，通过调用这些接口使得硬件（Hardware）运行正常。代码位于 hardware/libhardware/include/hardware 下。

在 HAL 层并没有实现定义的蓝牙协议与属性，默认实现在 Bluedroid 中，位于 external/bluetooth/bluedroid 下（6.0版本之后在system/bt），用户可以根据自己的需求增加属性。

## Bluetooth Stack

该层代码实现了 HAL 层中的定义，可以通过扩展和改变配置来自定义。代码位于 external/bluetooth/bluedroid 下（6.0版本之后在system/bt）。

## 参考资料

《低功耗蓝牙智能硬件开发实战》第2章《Android蓝牙系统框架和代码结构》
