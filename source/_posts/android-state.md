---
title: Android 源码的状态模式
date: 2019-11-02 00:49:53
tags: 设计模式
categories: Android
---

# Android 源码的状态模式

## 状态模式介绍

状态模式中的行为是由状态来决定的，不同的状态下有不同的行为。状态模式和策略模式的结构几乎完全一样，但它们的目的、本质却完全不一样。状态模式的行为是平行的、不可替换的，策略模式的行为是彼此独立、可相互替换的。用一句话来表述，状态模式把对象的行为包装在不同的状态对象里，每一个状态对象都有一个共同的抽象状态基类。状态模式的意图是让一个对象在其内部状态改变的时候，其行为也随之改变。

## 状态模式的定义

当一个对象的内在状态改变时允许改变其行为，这个对象看起来像是改变了其类。

## StateMachine

Android 的 Wifi 状态切换会使用 WifiController 类。

```java
/**
 * WifiController is the class used to manage on/off state of WifiStateMachine for various operating
 * modes (normal, airplane, wifi hotspot, etc.).
 */
public class WifiController extends StateMachine {
```

WifiController 类继承了 StateMachine 类。StateMachine 是一个状态机，它的实现是一种状态模式。

StateMachine 的 sendMessage 方法如下：

```java
    public void sendMessage(int what) {
        // mSmHandler can be null if the state machine has quit.
        SmHandler smh = mSmHandler;
        if (smh == null) return;

        smh.sendMessage(obtainMessage(what));
    }
```

sendMessage 方法通过 SmHandler 发送消息。

### SmHandler

SmHandler 的代码如下：

```java
    private static class SmHandler extends Handler {
    ...
        private class StateInfo {
            /** The state */
            State state;

            /** The parent of this state, null if there is no parent */
            StateInfo parentStateInfo;

            /** True when the state has been entered and on the stack */
            boolean active;
    ...
        }
    ...
```

StateInfo 是 SmHandler 的内部类，用来保存状态信息。state 是当前状态，parentStateInfo 是上一个状态，active 表示状态是否激活。

SmHandler 的 handleMessage 代码如下：

```java
        @Override
        public final void handleMessage(Message msg) {
            if (!mHasQuit) {
                ...
                    msgProcessedState = processMsg(msg);
                ...
                performTransitions(msgProcessedState, msg);
                ...
            }
        }
```

handleMessage 首先通过 processMsg 处理消息，然后通过 performTransitions 执行状态转换。

processMsg 方法如下：

```java
        private final State processMsg(Message msg) {
            StateInfo curStateInfo = mStateStack[mStateStackTopIndex];
            if (mDbg) {
                mSm.log("processMsg: " + curStateInfo.state.getName());
            }

            if (isQuit(msg)) {
                transitionTo(mQuittingState);
            } else {
                while (!curStateInfo.state.processMessage(msg)) {
                    /**
                     * Not processed
                     */
                    curStateInfo = curStateInfo.parentStateInfo;
                    if (curStateInfo == null) {
                        /**
                         * No parents left so it's not handled
                         */
                        mSm.unhandledMessage(msg);
                        break;
                    }
                    if (mDbg) {
                        mSm.log("processMsg: " + curStateInfo.state.getName());
                    }
                }
            }
            return (curStateInfo != null) ? curStateInfo.state : null;
        }
```

如果当前状态处理了消息，就返回当前状态，否则循环寻找它的上一个状态来处理。如果没有状态处理，会调用 unhandledMessage 方法。

performTransitions 方法如下：

```java
private void performTransitions(State msgProcessedState, Message msg) {
            ...
            State orgState = mStateStack[mStateStackTopIndex].state;
            ...
            State destState = mDestState;
            if (destState != null) {
                ...
                while (true) {
                    ...
                    StateInfo commonStateInfo = setupTempStateStackWithStatesToEnter(destState);
                    // flag is cleared in invokeEnterMethods before entering the target state
                    mTransitionInProgress = true;
                    invokeExitMethods(commonStateInfo);
                    int stateStackEnteringIndex = moveTempStateStackToStateStack();
                    invokeEnterMethods(stateStackEnteringIndex);
                    ...
        }
```

orgState 是原始状态，destState 是目标状态。invokeExitMethods 先退出状态，然后用 invokeEnterMethods 进入新的状态。

### addState

WifiController 在构造是初始化了一些状态。

```java
    WifiController(Context context, WifiStateMachine wsm, Looper wifiStateMachineLooper,
                   WifiSettingsStore wss, Looper wifiServiceLooper, FrameworkFacade f,
                   WifiStateMachinePrime wsmp) {
        super(TAG, wifiServiceLooper);
        ...
        // CHECKSTYLE:OFF IndentationCheck
        addState(mDefaultState);
            addState(mStaDisabledState, mDefaultState);
            addState(mStaEnabledState, mDefaultState);
                addState(mDeviceActiveState, mStaEnabledState);
            addState(mStaDisabledWithScanState, mDefaultState);
            addState(mEcmState, mDefaultState);
        // CHECKSTYLE:ON IndentationCheck
        ...
    }
```

WifiController 使用 addState 添加状态。

这里的 6 个 addState 方法没有按照正常的空格缩进。为了表示层级关系，越底层的状态缩进的越多。

mDefaultState 是最顶层的 State，mStaDisabledState、mStaEnabledState、mStaDisabledWithScanState、mEcmState 是第二层的 State，mDeviceActiveState 是最底层的 State。

addState 方法如下：

```java
        private final StateInfo addState(State state, State parent) {
            ...
            StateInfo parentStateInfo = null;
            if (parent != null) {
                parentStateInfo = mStateInfo.get(parent);
                if (parentStateInfo == null) {
                    // Recursively add our parent as it's not been added yet.
                    parentStateInfo = addState(parent, null);
                }
            }
            ...
            stateInfo.state = state;
            stateInfo.parentStateInfo = parentStateInfo;
            stateInfo.active = false;
            ...
            return stateInfo;
        }
```

addState 方法通过递归的方式构造了 stateInfo。

状态之间不是跨越式转换的，当前状态只能转换到上一个状态或者下一个状态。
