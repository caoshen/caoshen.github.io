---
title: Android 源码的命令模式
date: 2019-11-08 23:45:07
tags: 设计模式
categories: Android
---

# Android 源码的命令模式

## 命令模式介绍

命令模式是行为型设计模式之一。命令模式相对于其他的设计模式来说并没有那么多的条条框框，其实它不是一个很“规矩”的模式，不过，就是基于这一点，命令模式相对于其他的设计模式更为灵活多变。我们接触比较多的命令模式个例无非是程序菜单命令，如在操作系统中，我们点击“关机”命令，系统就会执行一系列的操作，如先是暂停处理事件，保存系统的一些配置，然后结束程序进程，最后调用内核命令关闭计算机等，对于这一系列的命令，用户不用去管，用户只需点击系统的关机按钮即可完成如上一系列的命令。而我们的命令模式其实也与之相同，将一系列的方法调用封装，用户只需调用一个方法执行，那么所有这些被封装的方法就会被挨个执行调用。

## 命令模式的定义

将一个请求封装成一个对象，从而让用户使用不同的请求把客户端参数化；对请求排队或者记录请求日志，以及支持可撤销的操作。

## Android 源码中的命令模式实现

Android 在对应用程序包管理的部分也有对命令模式应用的体现。

PackageManagerService 也是 Android 系统的 Service 之一，其主要功能在于实现对应用包的解析、管理、卸载等操作。

在 PackageManagerService 中，其对包的相关消息处理由其内部类 PackageHandler 承担，其将需要处理的请求作为对象通过消息传递给相关的方法，而对于包的安装是 HandlerParams 的具体子类 InstallParams。HandlerParams 的逻辑不多，主要是对 3 个抽象方法的声明。

HandlerParams 的代码如下：

```java
    private abstract class HandlerParams {
        ...
        final boolean startCopy() {
            boolean res;
            try {
                if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);

                if (++mRetries > MAX_RETRIES) {
                    Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
                    mHandler.sendEmptyMessage(MCS_GIVE_UP);
                    handleServiceError();
                    return false;
                } else {
                    handleStartCopy();
                    res = true;
                }
            } catch (RemoteException e) {
                if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
                mHandler.sendEmptyMessage(MCS_RECONNECT);
                res = false;
            }
            handleReturnCode();
            return res;
        }

        final void serviceError() {
            if (DEBUG_INSTALL) Slog.i(TAG, "serviceError");
            handleServiceError();
            handleReturnCode();
        }

        abstract void handleStartCopy() throws RemoteException;
        abstract void handleServiceError();
        abstract void handleReturnCode();
    }
```

InstallParams 实现了 HandlerParams 类。

```java
class InstallParams extends HandlerParams {
        ...
        private int installLocationPolicy(PackageInfoLite pkgLite) {
            ...
        }

        /*
         * Invoke remote method to get package information and install
         * location values. Override install location based on default
         * policy if needed and then create install arguments based
         * on the install location.
         */
        public void handleStartCopy() throws RemoteException {
            ...
        }

        @Override
        void handleReturnCode() {
            ...
        }

        @Override
        void handleServiceError() {
            ...
        }
    }
```

这里结合命令模式来看，HandlerParams 也是一个抽象命令者（Command），而对于它的子类 InstallParams 则对应具体命令角色（ConcreteCommand），请求者（Invoker）依然可以看作是由 PackageManagerService 承担的。