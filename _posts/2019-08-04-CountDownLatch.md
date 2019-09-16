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