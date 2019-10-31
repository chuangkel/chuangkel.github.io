---
layout:     post
title:	JUC专题
subtitle: 	ReentrantLock
date:       2019-08-02
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# ReentrantLock

### 问题

1. ReentrantLock是什么？
2. ReentrantLock有什么用？使用场景有哪些？
3. ReentrantLock有哪些特性？怎么实现的这些特性？



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



### 源码：ReentrantLock

> ReentrantLock重入锁，是实现Lock接口的一个类，能够对共享资源能够重复加锁，即当前线程获取该锁再次获取不会被阻塞。在java关键字synchronized隐式支持重入性（synchronized通过获取自增，释放自减的方式实现重入）。ReentrantLock支持公平锁和非公平锁两种方式。
>
> * 重入性的实现原理 
> * 公平锁和非公平锁

#### 重入性的实现原理 

> 以非公平锁为例，重入锁加锁

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {//重入锁实现点。同一线程再次进入
        int nextc = c + acquires;//state + 1
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);//更新state，The synchronization state.
        return true;
    }
    return false;
}
```

> 以非公平锁为例，重入锁释放

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;//state - 1
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

#### 公平锁和非公平锁

> ReentrantLock支持两种锁：公平锁和非公平锁。

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

> tryAcquire(arg)方法在同步器中预留给子类实现。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

##### 公平锁

> 公平锁每次获取到锁为同步队列中的第一个节点，保证请求资源时间上的绝对顺序，满足FIFO，会产生频繁的上下文切换。
>
> 公平锁为了保证时间上的绝对顺序，需要频繁的上下文切换，

```java
final void lock() {
    acquire(1);
}
```

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {//公平锁多了hasQueuedPredecessors(),判断当前线程是否有前驱节点
        if (!hasQueuedPredecessors() && //根据公平性，有前驱节点将获取锁失败
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

##### 非公平锁

> 非公平锁有可能刚释放锁的线程下次继续获取该锁，则有可能导致其他线程永远无法获取到锁，造成“饥饿”现象。而非公平锁会降低一定的上下文切换，降低性能开销。因此，ReentrantLock默认选择的是非公平锁，则是为了减少一部分上下文切换，保证了系统更大的吞吐量。

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



## 读写锁的实践

> 读阻塞写，不阻塞其他读
>
> 写阻塞读，又阻塞其他写