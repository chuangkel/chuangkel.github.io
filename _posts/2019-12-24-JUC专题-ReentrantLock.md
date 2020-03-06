---
layout:     post
title:	JUC专题
subtitle: 	ReentrantLock
date:       2019-12-24
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# ReentrantLock

### 问题

#### ReentrantLock是什么？

ReentrantLock是可重入、非公平和公平锁的显式实现，可以实现多条件的等待（即多个锁）。

#### ReentrantLock有什么用？使用场景有哪些？

#### ReentrantLock有哪些特性？怎么实现的这些特性？




### 源码：ReentrantLock

> ReentrantLock重入锁，内部类继承AbstractQueueSynchronizer，实现Lock接口。具有重入性，同一线程能够重复获取进入锁。
>
> 在java关键字synchronized隐式支持重入性（synchronized通过获取自增，释放自减的方式实现重入）。
>
> ReentrantLock支持公平锁和非公平锁两种方式。

#### ReentrantLock重入性

以非公平锁为例，重入锁加锁

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

以非公平锁为例，重入锁释放

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

最大重入次数：Integer.MAX_VALUE，超过了会抛出异常

Exception in thread "main" java.lang.Error: Maximum lock count exceeded
	at java.util.concurrent.locks.ReentrantLock\$Sync.nonfairTryAcquire(ReentrantLock.java:141)
	at java.util.concurrent.locks.ReentrantLock\$NonfairSync.tryAcquire(ReentrantLock.java:213)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1198)
	at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)
	at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)
	at com.github.chuangkel.java_learn.base.thread.threadMutual.ReentrantLockTest.main(ReentrantLockTest.java:18)

#### ReentrantLock公平锁和非公平锁

公平和非公平锁的实现区别只有加锁过程不同： 

1. 公平锁的加锁过程，根据实现同步器的子类公平锁的tryAquire方法判断，**先判断state是为0并且同步队列没有后继者**，若满足进行state的CAS修改，CAS修改成功则获取到锁，设置当前线程为持有锁的线程。若该线程是重入，则state加一，成功获取到锁。若不满足或CAS修改state失败则新建一个节点放入同步队列中。 获取同步队列中新加入节点的上一个节点，如果上一个节点是head节点，同时尝试获取修改state状态，CAS修改state成功获取到锁。 如果不是头结点或者CAS修改state失败，往前遍历同步队列，淘汰已经取消等待的节点，保证上一个节点的waitStatus状态是SIGNAL，再park当前线程。保证了上一个节点释放锁时会唤醒当前节点线程。
2.  非公平锁加锁过程，根据同步器子类非公平锁实现的方法tryAquire，**不会判断state是否为0，不会判断同步队列是否有等待节点**，直接CAS修改state，修改成功或者当前线程是可重入的，那么成功获取到锁，把当前线程设置到头节点变量中。失败则加入同步队列，加入到同步队列都是同步器框架实现的，所以非公平锁和公平锁加入同步队列的过程相同。

**区别：** 公平锁可以插队，非公平锁不可以插队

ReentrantLock支持两种锁：公平锁和非公平锁。

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

tryAcquire(arg)方法在同步器中预留给子类实现。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

##### 公平锁

公平锁每次获取到锁为同步队列中的第一个节点，保证请求资源时间上的绝对顺序，满足FIFO，会产生频繁的上下文切换。

公平锁为了保证时间上的绝对顺序，需要频繁的上下文切换

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

非公平锁有可能刚释放锁的线程下次继续获取该锁，则有可能导致其他线程永远无法获取到锁，造成“饥饿”现象。而非公平锁会降低一定的上下文切换，降低性能开销。因此，**ReentrantLock默认选择的是非公平锁**，则是为了减少一部分上下文切换，保证了系统更大的吞吐量。

ReentrantLock默认非公平锁  java.util.concurrent.locks.ReentrantLock#ReentrantLock()

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
```

NonfairSync 一上来就干，直接CAS

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

上面的失败了再按公平锁的套路来，注意还是没有判断同步队列

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) { //没有鸟同步队列
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



lockInterruptly调用了该方法，在尝试获取锁之前如果线程发生了中断，则抛出InterruptedException

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```



### ReentrantLock和Synchronized的区别？

| 区别                     | ReentrantLock                                                | Synchronized                                                 |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 实现                     | 基于Unsafe类的实现来挂起和唤醒线程，采用双向链表的方式存储等待的线程节点 | jvm层面的实现,通过对象头来记录偏向锁、轻量级锁、重量级锁的标识，每个锁有对应的监视器对象，用来控制线程的切换 |
| 公平性和非公平锁         | 可以实现公平锁和非公平锁                                     | 只能使用公平锁                                               |
| 锁升级、锁粗化、锁消除、 | 无                                                           | jvm实现会对锁进行锁升级，通过对象头锁标识，来进行锁的升级，自动进行锁的粗化比如在for循环类加锁和锁的消除，对不可能存在锁竞争的地方进行锁的消除，比如字符串相加。 |
|                          |                                                              |                                                              |

ReentrantLock可中断锁是怎么实现的？Synchronized可以中断吗?

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

需要在acquireInterruptibly方法调用之前中断该线程。