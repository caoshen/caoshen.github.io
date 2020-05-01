# Android 源码的原型模式

## 原型模式介绍

原型模式是一个创建型的模式。原型二字表明了该模式应该有一个样板实例，用户从这个样板对象中复制出一个内部属性一致的对象，这个过程也就是我们俗称的“克隆”。被复制的实例就是我们所称的原型，这个原型是可定制的。原型模式多用于创建复杂的或者构造耗时的实例，因为这种情况下，复制一个已经存在的实例可使程序运行更高效。


## 原型模式的定义

用原型实例指定创建对象的种类，并通过拷贝这些原型创建新的对象。

## 浅拷贝和深拷贝

浅拷贝在拷贝引用型的字段时，直接复制引用，导致和原型指向同一个对象。深拷贝会将引用型的字段也拷贝一份，不影响原型对象。

## ArrayList

JDK 的 ArrayList 实现了 Cloneable 接口，可以使用原型模式拷贝对象。

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    ...
    transient Object[] elementData; // non-private to simplify nested class access
    private int size;
    ...
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
...
}    
```

clone 方法先调用 super.clone 得到一个 ArrayList v，然后将 v 的引用型成员变量 elementData 使用 Arrays.copyOf 深拷贝。

## Intent

Intent 实现了 Cloneable 接口，可以使用原型模式构造 Intent。

```java
public class Intent implements Parcelable, Cloneable {
...
    /**
     * Copy constructor.
     */
    public Intent(Intent o) {
        this(o, COPY_MODE_ALL);
    }

    private Intent(Intent o, @CopyMode int copyMode) {
        this.mAction = o.mAction;
        this.mData = o.mData;
        this.mType = o.mType;
        this.mPackage = o.mPackage;
        this.mComponent = o.mComponent;

        if (o.mCategories != null) {
            this.mCategories = new ArraySet<>(o.mCategories);
        }

        if (copyMode != COPY_MODE_FILTER) {
            this.mFlags = o.mFlags;
            this.mContentUserHint = o.mContentUserHint;
            this.mLaunchToken = o.mLaunchToken;
            if (o.mSourceBounds != null) {
                this.mSourceBounds = new Rect(o.mSourceBounds);
            }
            if (o.mSelector != null) {
                this.mSelector = new Intent(o.mSelector);
            }

            if (copyMode != COPY_MODE_HISTORY) {
                if (o.mExtras != null) {
                    this.mExtras = new Bundle(o.mExtras);
                }
                if (o.mClipData != null) {
                    this.mClipData = new ClipData(o.mClipData);
                }
            } else {
                if (o.mExtras != null && !o.mExtras.maybeIsEmpty()) {
                    this.mExtras = Bundle.STRIPPED;
                }

                // Also set "stripped" clip data when we ever log mClipData in the (broadcast)
                // history.
            }
        }
    }

    @Override
    public Object clone() {
        return new Intent(this);
    }
}
```

Intent 的 clone 方法没有调用 super.clone 方法来实现对象拷贝，而是调用了 new Intent(this) 的拷贝构造方法。拷贝构造方法将原始对象的每个数据逐个拷贝一遍。到此，整个克隆过程就完成了。