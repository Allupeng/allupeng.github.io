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

<center>    <img style="border-radius: 0.3125em;    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"     src="https://i.loli.net/2021/05/20/WMuxakG8bCi9NqK.png">    <br>    <div style="color:orange; border-bottom: 1px solid #d9d9d9;    display: inline-block;    color: #999;    padding: 2px;">Java的锁分类</div> </center>

- 根据**Java中是否需要在每次获取资源时候加锁**分为悲观锁、乐观锁
- 根据**获取了锁之后是否还能继续加锁**分为可重入锁和非可重入锁。
- 根据**获取锁的粒度来区分**可以分为无锁、偏向锁、轻量级锁、重量级锁。
- 根据**获取锁之后是否能被其他线程继续获取锁**分为共享锁和排他锁。
- 根据**获取锁时是否能插队**分为公平锁和非公平锁。

## 悲观锁、乐观锁

**悲观锁：**对于访问资源的时候**处于悲观状态**，即每次获取资源的时候为了**防止资源竞争**，**每次获取资源的时候都需进行加锁**。适用于**写多读少**的情况。

例如Java中的**synchronized关键字**和**RetreenLock类**是悲观锁的实现。

**乐观锁：**对于访问资源的时候**处于乐观状态**，每次获取资源的时候都**不需进行加锁**。适用于**读多写少**的情况。

**CAS就是乐观锁**的实现。

### Synchronized

