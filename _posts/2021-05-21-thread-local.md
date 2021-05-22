---

layout: post
title: "ThreadLocal原理"
date: 2021-05-21 19:49:55 +0800 
categories: 计算机
tags: Java

---

# ThreadLocal原理

每一个线程的Thread对象中都有一个ThreadLocal.ThreadLocalMap对象。

**Thread.java**中ThreadLocal.ThreadLocalMap对象

```java
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```

ThreadLocalMap是ThreadLocal类中的一个静态内部类。

查看ThreadLocal类中的get方法

```java

    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }


    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

```

查看这个getMap（Thread t）方法其实就是返还当前线程的threadLocals变量。我们这里可以猜想一下是否ThreadLocal的原理就是将每个线程都拥有自己的threadLocal变量，从而实现变量变量线程隔离的。

来分析一下get（）方法

- 首先获取当前的线程和当前线程的ThreadLocalMap对象
- 如果这个ThreadLocal对象存在（不为null）
- 获取ThreadLocalMap中的Entry
- 如果这个Entry不为空，返回Entry中存的Value
- 否则初始化并创造一个Value

我们首先看一下ThreadLocalMap.Entry这个类

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

可以看出Entry是存了一个弱引用的ThreadLocal为K，相应的存在ThreadLocal为Value的K-V键值对。

我们再查看一下ThreadLocalMap.getEntry()这个方法

```java
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;
```

其实ThreadLocal是将k-v存在一个Entry[]数组中。其中table.length 必须为2的幂，这里解释一下为什么。

- 一般通过长度为2的幂，可以将模运算简化为位运算。

  即 **key.threadLocalHashCode & (table.length - 1) = key.threadLocalHashCode % table.length**。

- 提高Hash函数的散列值，可以使散列的时候分散得更均匀。

我们再来查看一下当map获取失败的时候的setInitialValue（）方法。

```java
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

- initialValue（）方法是用户重写的
- 获取当前的线程，获取ThreadLocalMap
- 如果map不为null，那么就更新value
- 如果map为null，那么就创建map

再查看一下set（）方法

```java
        private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

- 首先通过threadLocalHashCode进行取模运算计算位置。
- 判断取到的ThreadLocal是否是当前的ThreadLocal，如果是更新值。

那么Thread的ThreadLocals是如何更新的呢？是通过createMap（）来进行的

```java
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```



让我们来捋一下思路。

每一个线程都有自己的ThreadLocalMap。

ThreadLocalMap里面有一个Entry数组，存储着线程自身对应的ThreadLocal对象和相应存储的Value。

通过ThreadLocal的set（）方法将value哈希进ThreadLocalMap中。

每一个ThreadLocal对象（一个线程可以包含多个ThreadLocal对象）都包含了一个独一无二的threadLocalHashCode，使用这个值就可以在线程K-V值对中找回对应的本地线程变量。



### ThreadLocal如何防止内存泄漏？

为什么ThreadLocal会内存泄漏呢？每个Entry都是以ThreadLoca的弱引用为K，一个值为V。那么K会被JVM自己回收，可是值可能就是强引用了。

如果K被JVM回收了，那么我们的值就永远不会被回收，这样就会导致内存泄漏。我们来看下ThreadLocal是如何解决这个问题的。

```java
        private void set(ThreadLocal<?> key, Object value) {
			//**略**//
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

针对该问题，ThreadLocalMap 的 set 方法中，通过 replaceStaleEntry 方法将所有键为 null 的 Entry 的值设置为 null，从而使得该值可被回收。另外，会在 rehash 方法中通过 expungeStaleEntry 方法将键和值为 null 的 Entry 设置为 null 从而使得该 Entry 可被回收。通过这种方式，ThreadLocal 可防止内存泄漏。



## 参考资料

- [Java进阶（七）正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/)
- JDK源码

