---
title: Android 源码的中介者模式
date: 2019-11-17 23:58:39
tags: 设计模式
categories: Android
---

# Android 源码的中介者模式

## 中介者模式介绍

中介者模式（Mediator Pattern）也成为调解者模式或者调停者模式，Mediator 本身就有调停者和调解者的意思。

## 中介者模式的定义

中介者模式包装了一系列对象相互作用的方式，使得这些对象不必相互明显作用。从而使它们可以松散耦合。当某些对象之间的作用发生改变时，不会立即影响其他的一些对象之间的作用。保证这些作用可以彼此独立的变化。

中介者模式将多对多的相互作用转化为一对多的相互作用。中介者模式将对象的行为和协作抽象化，把对象在小尺度的行为上与其他对象的相互作用分开处理。

##  Android 源码的中介者模式实现

中介者模式在 Android 源码中的一个比较好的例子是 Keyguard 锁屏的功能实现。

### KeyguardViewMediator

KeyguardViewMediator 代码如下：

```java
public class KeyguardViewMediator extends SystemUI {
    ...
    private AlarmManager mAlarmManager;
    private AudioManager mAudioManager;
    private StatusBarManager mStatusBarManager;
    ...
    private StatusBarKeyguardViewManager mStatusBarKeyguardViewManager;
    ...
    private void playSounds(boolean locked) {
        playSound(locked ? mLockSoundId : mUnlockSoundId);
    }

    private void playSound(int soundId) {
        if (soundId == 0) return;
        final ContentResolver cr = mContext.getContentResolver();
        if (Settings.System.getInt(cr, Settings.System.LOCKSCREEN_SOUNDS_ENABLED, 1) == 1) {

            mLockSounds.stop(mLockSoundStreamId);
            // Init mAudioManager
            if (mAudioManager == null) {
                mAudioManager = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
                if (mAudioManager == null) return;
                mUiSoundsStreamType = mAudioManager.getUiSoundsStreamType();
            }

            mUiOffloadThread.submit(() -> {
                // If the stream is muted, don't play the sound
                if (mAudioManager.isStreamMute(mUiSoundsStreamType)) return;

                int id = mLockSounds.play(soundId,
                        mLockSoundVolume, mLockSoundVolume, 1/*priortiy*/, 0/*loop*/, 1.0f/*rate*/);
                synchronized (this) {
                    mLockSoundStreamId = id;
                }
            });

        }
    }
    ...
}
```

KeyguardViewMediator 中看到很多 XXXManager 管理器的成员变量，这些各种各样的管理器其实就是各个具体的同事类。

Android 使用 KeyguardViewMediator 来充当中介者协调这些管理器的状态改变。同样地，KeyguardViewMediator 中也定义了很多方法来处理这些管理器地状态，以解锁或锁屏时声音的播放为例，在 KeyguardViewMediator 中对应 playSounds 方法来协调音频的这一状态。

而其他管理器的协调也可以在 KeyguardViewMediator 中找到类似的协调方法，这里不再多说。

### Binder 机制

Binder 机制是 Android 中非常重要的一个机制，其用来绑定不同的系统级服务并进行跨进程通信。

在 Binder 机制中，有 3 个非常重要的组件 ServiceManager、Binder Driver 和 Bp Binder，其中 Bp Binder 是 Binder 的一个代理角色，其提供 IBinder 接口给各个客户端服务使用，这三者就扮演了一个中介者的角色。

当手机启动后，ServiceManager 会先向 Binder Driver 进行注册，ServiceManager 是一个特殊的服务，它在 Binder Driver 中是最先被注册的，其注册 ID 为 0。

当其他的服务想要注册到 Binder Driver 时，会先通过这个 0 号 ID 获取到 ServiceManager 所对应的 IBinder 接口，该接口实质上的实现逻辑是由 Bp Binder 实现的，获取到对应的接口后就会回调其中的 transact 方法，此后就会在 Binder Driver 中新注册一个 ID 1 来对应这个服务。如果有客户端想要使用这个服务，它会像 Binder Driver 一样获取到 ID 为 0 的接口，也就是 ServiceManager 对应的接口，并调用其 transact 方法要求连接到刚才的服务，这时候 Binder Driver 就会将 ID 为 1 的服务回传给客户端并将相关消息反馈给 ServiceManager 完成连接。

这里 ServiceManager 和 Binder Driver 就相当于一个中介者，协调各个服务端与客户端。