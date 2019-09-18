---
layout:     post
title:	CountDownLatch
subtitle: 	CountDownLatch
date:       2019-08-04
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# CountDownLatch

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }	//设置node前节点为需要唤醒后节点，即Node.SIGNAL
            if (shouldParkAfterFailedAcquire(p, node) && 
                parkAndCheckInterrupt()) //阻塞线程
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```



```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);//当前线程新建节点，加入等待队列
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;//pre指向最后一个节点
    if (pred != null) {//队列不为空
        node.prev = pred;//设置node在链表中的pre
        if (compareAndSetTail(pred, node)) {//cas设置tail,pred是预期值，node是更新值
            pred.next = node;
            return node;
        }
    }
    enq(node);//如果队列为null，或者compareAndSetTail(pred, node)失败
    return node;
}
```



```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)//-1 需要唤醒后面节点
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {//1,取消了等待
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {// 0,-2，一般都是0，将前节点设置成需唤醒后节点
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```



```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//阻塞当前线程
    return Thread.interrupted(); //返回中断标识，并清除中断标识为false
}
```



1. CountDownLatch有什么特性，使用的场景有哪些？

   > 可以在一个线程中等待其他线程，然后同时唤醒其他线程。即在各自任务中的某个点停住，等待所有线程都就位之后，由最后一个到位的线程唤醒其他线程。

2. CountDownLatch的方法？

   > * countDown()
   >
   > * await()

3. 怎么实现多线程等待的？

   > new CountDownLatch(n)传入了n，当n减到0的线程唤醒其他线程。穿入的n是AQS同步器的锁变量state，当state减到0时，唤醒其他线程。

   


### await()方法

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

> 调用AQS同步器中的代码

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0) //子类实现了该方法
        // 若锁state大于0
        doAcquireSharedInterruptibly(arg);
}
```

> 若锁state为0，则进入if内部，执行doAcquireSharedInterruptibly(arg)方法

```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

> 如果锁state变成了0，则调用父类AQS同步器中的doAcquireSharedInterruptibly代码，该代码实现了什么呢？

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    //到了这里 state锁 大于0
    final Node node = addWaiter(Node.SHARED); //新建一个当前线程的Node，添加到同步器队列的尾部
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor(); //取上一个节点
            if (p == head) { //上一个节点是头结点，自身节点是队列的第一个元素，头结点不属于队列
                int r = tryAcquireShared(arg); //获取锁state状态
                if (r >= 0) { //r == 1 锁state变成了0，可以获取锁了
            //设置头结点，并将头节点的waitStatus设置成PROPAGATE(-3)
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 到了这里，则不是头节点，将上一个节点设置成SIGNAL 同时挂起线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### countDown()方法

```java
public void countDown() {
    sync.releaseShared(1);
}
```

> 调用父类AQS同步器中的代码

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        //若锁state刚好减到0，这进入到了这里
        doReleaseShared();
        return true;
    }
    return false;
}
```

> CountDownLatch自身实现的tryReleaseShared方法，如果锁state减为0了，直接返回false。如果锁state不为0，则减一，同时返回 state == 0。

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc)) //多线程并发这里失败了会产生问题吗？
            return nextc == 0;
    }
}
```

> 调用了父类AQS同步器的方法

```java
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```