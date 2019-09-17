---
layout:     post
title:	ReentrantLock
subtitle: 	ReentrantLock
date:       2019-08-02
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# ReentrantLock

> lockInterruptly调用了该方法，在尝试获取锁之前如果线程发生了中断，则抛出InterruptedException

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```