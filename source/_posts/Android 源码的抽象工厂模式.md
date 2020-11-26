# Android 源码的抽象工厂模式

## 抽象工厂模式介绍

抽象工厂模式也是创建型模式之一。抽象工厂模式起源于以前对不同操作系统的图形化解决方案。如不同操作系统中的按钮和文本框控件实现不同，展示效果也不一样，对于每个操作系统，其本身就构成一个产品类，而按钮与文本框控件也构成一个产品类，两种产品类两种变化，各自有自己的特性。

## 抽象工厂模式的定义

为创建一组相关或者是相互依赖的对象提供一个接口，而不需要指定它们的具体类。

## MediaPlayerFactory

MediaPlayerFactory 是一个用来创建 MediaPlayer 的工厂类。

```cpp
sp<MediaPlayerBase> MediaPlayerFactory::createPlayer(
        player_type playerType,
        const sp<MediaPlayerBase::Listener> &listener,
        pid_t pid) {
    sp<MediaPlayerBase> p;
...
    p = factory->createPlayer(pid);
...
    return p;
}
```

createPlayer 方法用来创建不同的 MediaPlayer。

MediaPlayerFactory 本质只是用来管理 Android 内置的 MediaPlayer，而每一种具体的 MediaPlayer 则由一个具体的 Factory 类来创建。

### NuPlayerFactory

```cpp
class NuPlayerFactory : public MediaPlayerFactory::IFactory {
...
    virtual sp<MediaPlayerBase> createPlayer(pid_t pid) {
        ALOGV(" create NuPlayer");
        return new NuPlayerDriver(pid);
    }
};
```

NuPlayerFactory 创建 NuPlayerDriver。


### TestPlayerFactory

```cpp
class TestPlayerFactory : public MediaPlayerFactory::IFactory {
  ...
    virtual sp<MediaPlayerBase> createPlayer(pid_t /* pid */) {
        ALOGV("Create Test Player stub");
        return new TestPlayerStub();
    }
};
```

TestPlayerFactory 创建 TestPlayerStub。