---

layout: post
title: "ThreadLocal原理"
date: 2021-05-21 19:49:55 +0800 
categories: 计算机
tags: Java

---

# ThreadLocal原理

每一个线程的Thread对象中都有一个ThreadLocalMap对象，这个对象存储了一组以ThreadLocal.threadLocalHashCode为key，以本地现场变量为value的K-V键值对，ThreadLocal对象就是以当前的ThreadLocalMap的访问入口，每一个ThreadLocal对象都包含了一个独一无二的threadLocalHashCode，使用这个值就可以在线程K-V值对中找回对应的本地线程变量。

