---
layout: post
title: "Java中的锁"
date: 2021-05-20 10:07:22 +0800 
categories: 计算机
tags: Java
typora-root-url: ..\images
typora-copy-images-to: upload
---

##  简要

**锁（Lock）**对于大家来说应该不陌生吧，对于需要同步互斥资源的使用，来**防止并发情况下的资源竞争**，锁是一个很好的手段。

例如有一个共享变量**A = 10**，**线程1**先**读取共享变量A的值10**，线程1将读取到的值**相加10**，但是还没来得及将相加后的值**写入共享变量A中**，线程2**已经读取了原来未修改的共享变量A的值10**，并也进行相加10的操作，这样会导致最后的结果为20。而并非正确结果30。

<center>    <img style="border-radius: 0.3125em;    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"     src="https://i.loli.net/2021/05/20/DQwmizoV6Cr9GYX.png">    <br>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;    display: inline-block;    color: #999;    padding: 2px;">线程1、线程2读取共享变量A（1）（2）</div> </center>

<center>    <img style="border-radius: 0.3125em;    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"     src="https://i.loli.net/2021/05/20/Wc2agOCsSlAUhdP.png">    <br>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;    display: inline-block;    color: #999;    padding: 2px;">线程1、线程2写值进共享变量A（3）（4）</div> </center>

那么我们如何解决这个问题呢？我们可以给变量A**加一把锁**，当其他线程访问变量A的时候，首先需要**获取锁**，如果**获取锁成功了**，就可以**操作变量A**。如果**没有获取到锁（包括获取锁失败和其他线程已经获取了锁）**，就需要**重试或者放弃（取决于策略）**。

Java中源码无所不有，其中JUC中**（Java.util.concurrent）**大部分代码都是一个鼎鼎大名的叫 [Doug Lea](https://en.wikipedia.org/wiki/Doug_Lea) 的人写的。

接下来我们介绍一下Java中的锁知识。

<center>    <img style="border-radius: 0.3125em;    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"     src="https://i.loli.net/2021/05/21/qzrlkeywFA9mhCa.png">    <br>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;    display: inline-block;    color: #999;    padding: 2px;">Java的锁分类</div> </center>

- 根据**线程要不要锁住同步资源**可以分为**悲观锁**和**乐观锁**。
- 根据**锁锁住同步资源失败，线程要不要阻塞**。可以分为**阻塞、非阻塞**。其中**非阻塞可以分为自旋锁和自适应自旋锁**。
- 根据**多个线程竞争同步资源的流程细节有没有区别**可以分为**无锁、偏向锁、轻量级锁和重量级锁**。
- 根据**多个线程竞争时要不要排队**可以分为**公平锁和非公平锁。**
- 根据**一个线程中的多个流程能不能获得同一把锁**可以分为**可重入锁**和**非可重入锁**。
- 根据**多个线程是否可以获得同一把锁**可以分为**排他锁**和**共享锁**。

## 悲观锁、乐观锁

**悲观锁：**对于访问资源的时候**处于悲观状态**，即每次获取资源的时候为了**防止资源竞争**，**每次获取资源的时候都需进行加锁**。适用于**写多读少**的情况。

例如Java中的**synchronized关键字**和**RetreenLock类**是悲观锁的实现。

**乐观锁：**对于访问资源的时候**处于乐观状态**，每次获取资源的时候都**不需进行加锁**。适用于**读多写少**的情况。

**CAS就是乐观锁**的实现。

悲观锁的使用：

```java
    public synchronized void test(){
        //write your code
    }
    
    public void testReenterLock(){
        Lock lock = new ReentrantLock();
        lock.lock();
        //write your code
        lock.unlock();
    }
```

### CAS

CAS全称Compare And Swap，比较并交换。其中CAS有三个值，**旧值oldvalue、期望值expected以及更新值V**。根据比较**旧值是否等于期望值**，如果**相等**，将**旧值更新为V**。如果不相等，说明有其他线程修改了这个变量。**重试或者放弃修改。**

Java中JUC里面的原子类底层就是依赖的Unsafe方法中的**CAS+自旋**的方法实现的。CAS在操作系统底层是由cpu指令**cmpxchg** 所实现的。

**AtomicInteger.java源码**

```java

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe(); //获取并操作内存的数据。
    private static final long valueOffset; //存储value在AtomicInteger中的偏移量。

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));//获取存储value在AtomicInteger中的偏移量。
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;//存储AtomicInteger的int值，该属性需要借助volatile关键字保证其在线程间是可见的。
```

其中有比较重要的三个值。

- unsafe：获取并操作内存的数据。
- valueOffset：存储value在AtomicInteger中的偏移量。
- value：存储AtomicInteger的int值，该属性需要借助volatile关键字保证其在线程间是可见的

我们来查看一下**AtomicInteger**里面的**incrementAndGet（）**方法。

```java
    /**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
```

点进去**getAndAddInt（）**这个方法看一下

```java
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));//CAS + 自旋

        return var5;
    }
```

其中var5是期望值，查看**对应内存偏移量（var2）的那个值是否等于var5**，如果等于，就将**值更新为var5 + var4**。在**incrementAndGet（）这个方法**中，**var4等于1**。也就是进行自增的操作。由可以查看到底层是**CAS+自旋**实现的。

这样看可能不是很清晰，我们查看openjdk里面的源代码。

```java
// ------------------------- OpenJDK 8 -------------------------
// Unsafe.java
public final int getAndAddInt(Object o, long offset, int delta) {
   int v;
   do {
       v = getIntVolatile(o, offset);
   } while (!compareAndSwapInt(o, offset, v, v + delta));
   return v;
}
```

CAS虽然效率很高，但是有以下几个问题。

1. **ABA问题：**CAS需要在操作值的时候检查内存值是否变化了，但是如果内存原来的值是A，然后修改为了B，然后又修改为了A。那么进行CAS进行检查的时候是感知不到这个变化的。ABA的问题解决思路是在变量前面添加版本号，每次进行修改的时候也进行版本号的修改。1A--->2B--->3A。
   - 于jdk1.5的时候，Doug Lea写了一个AtomicStampedReference来添加一个stamp来解决ABA问题。
2. **循环时间开销长：**如果CAS一直不成功，会导致其一直在自旋，给CPU带来大量消耗
3. **只能保证一个共享变量的原子操作：**对一个变量执行操作的时候。CAS能够保证原子性。
   - Java从1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作。

## 自旋锁VS适应性自旋锁

在描述自旋锁前，我们先补充一些前置知识。

阻塞和唤醒一个Java线程需要操作系统进行切换CPU状态来进行，如果同步资源块执行的代码比较小，那么状态转换后所消耗的资源可能要比用户执行代码时间还要长。

而为了让当前线程“稍等一下”来获取锁，我们需要让当前线程自旋，如果在自旋完成后前面锁定同步资源的线程已经释放了锁，那么当前线程就可以不必阻塞而是直接获取锁了，从而避免线程切换的消耗。

![spinlock](https://i.loli.net/2021/05/21/iu1y7CFUpckq2hG.png)

但是自旋锁也有自己的缺点，如果前面线程一直不释放的话，自旋锁就会一直消耗CPU资源。所以，适应性自旋锁就诞生了。

如果锁被占用的时间很短，自旋等待的效果就会非常好。反之，如果锁被占用的时间很长，那么自旋的线程只会白浪费处理器资源。所以，自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是10次，可以使用-XX:PreBlockSpin来更改）没有成功获得锁，就应当挂起线程。

自旋锁在JDK1.4.2中引入，使用-XX:+UseSpinning来开启。JDK 6中变为默认开启，并且引入了自适应的自旋锁（适应性自旋锁）。

自适应意味着自旋的时间（次数）不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费处理器资源。

## 无锁、偏向锁、轻量级锁、重量级锁

这四种锁是指锁的状态，专门针对synchronized的。在介绍和四种锁前我们还需要介绍一下其他知识。

### synchronized

synchronized 关键字经过javac编译后，会在同步块前后生成**moniterenter和moniterexit**这两个字节码指令。这两个字节码指令都需要一个reference类型来指明要锁定和解锁的对象。如果Java源码中的synchronized明确指明了对象参数，那么就以这个对象的引用作为reference。如果没有明确指明，那么根据synchronized修饰的方法类型（如实例方法或者类方法）来决定是取代码所在的对象实例还是取类型对应的Class对象来作为线程要持有的锁。

在**执行moniterenter指令**的时候，首先要去尝试获取对象的锁，如果这个对象没有被锁定或者当前线程已经持有了这个对象的锁，就把锁的计数器加一。而在执行moniterexit指令的时候将计数器的值减一。一旦计数器的值为0，这个锁就被释放了。如果获取当前锁对象失败，那么当前线程就应该阻塞等待，直到请求锁定的对象被持有它的线程释放为止。

在JDK1.5之前，synchronized的性能是很差的。（因为是阻塞）

通过Java团队对于锁的优化后，synchronized和ReenterLock的性能是差不多的了。

要了解偏向锁和轻量级锁，首先问偶们先要介绍一下HotSport虚拟机对象头（Object Header）的组成部分。对象头分为两个部分，一个是存储对象自身的运行时数据，如哈希码（HashCode）、GC年龄分代（Generational GC Age）等。这部分在32位和64位Java虚拟机中占用32个比特和64个比特，官方称之为Mark Word。这是实现偏向锁和自旋锁的关键。

![image-20210522172657300](https://i.loli.net/2021/05/22/yUDXnxGFtrcCv2Y.png)









## 参考资料

- [美团技术团队：不可不说的Java"锁"事](https://tech.meituan.com/2018/11/15/java-lock.html)
- 周志明：《深入理解JVM第三版》

