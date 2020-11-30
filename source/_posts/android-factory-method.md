---
title: Android 源码的工厂方法模式
date: 2019-10-27 23:48:26
tags: 设计模式
categories: Android
---

# Android 源码的工厂方法模式

## 工厂方法模式介绍

工厂方法模式（Factory Pattern），是创建型设计模式之一。工厂方法模式是一种结构简单的模式，比如 Android 中的 Activity 里的各个生命周期方法，以 onCreate 方法为例，它可以看作是一个工厂方法，我们在其中可以构造 View，通过 setContentView 返回给 framework 处理。

## 工厂方法模式的定义

工厂方法模式定义一个用于创建对象的接口，让子类决定实例化哪个类。

## Iterable

以 List 和 Set 为例，List 和 Set 都继承于 Collection 接口，而 Collection 接口继承于 Iterable 接口，Iterable 接口很简单，有一个 iterator 方法。Java 1.8 定义了 forEach 和 spliterator 方法。

```java
public interface Iterable<T> {
    Iterator<T> iterator();
    ...
}
```

### ArrayList

ArrayList 实现了 iterator 方法，返回一个 Itr 迭代器对象。

```java
    public Iterator<E> iterator() {
        return new Itr();
    }
```

```java
    private class Itr implements Iterator<E> {
        ...
        protected int limit = ArrayList.this.size;

        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor < limit;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            int i = cursor;
            if (i >= limit)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
                limit--;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;

            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```
Itr 迭代器对象提供了 hasNext、next、remove、forEachRemaining 的实现。

### HashSet

HashSet 的 iterator 方法返回了它的成员变量 map 的 keySet 的 迭代器对象。

```java
    private transient HashMap<E,Object> map;
```

```java
    public Iterator<E> iterator() {
        return map.keySet().iterator();
    }
```

```java
    final class KeySet extends AbstractSet<K> {
        public final int size()                 { return size; }
        public final void clear()               { HashMap.this.clear(); }
        public final Iterator<K> iterator()     { return new KeyIterator(); }
        ...
    }
```

KeySet 的 iterator 方法返回了一个 KeyIterator。

## onCreate

Android 的 Activity 的 onCreate 方法是一个工厂方法模式。

```java
public class AActivity extends Activity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_a);
    }
}
```

我们在 AActivity 的 onCreate 方法中得到一个 View，并且设置为当前界面的 ContentView 返回给 framework 处理，如果现在又有一个 BActvity，只需要在其 onCreate 通过 setContentView 设置另外的 View，这是一个工厂模式的结构。


