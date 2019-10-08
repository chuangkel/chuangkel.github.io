---
layout:     post
title:	JUC专题
subtitle: 	CountDownLatch
date:       2019-08-04
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# CountDownLatch

> CountDownLathch使用场景：比如主线程启动子线程之后，需等待子线程都执行完，那么就需在主线程调用await()，每个子线程执行完成任务之后执行countDown()，待锁state减到0的子线程会唤醒主线程。如此便实现了主线程与子线程的协作工作。当然不一定主线程等待，任何线程都可以作为等待线程。而countDown()的线程并没被阻塞，countDown()底层实现是sync.releaseShared(1)。
>
> **存在多个线程调用CountDownLatch.await()的场景**

## 示例

> 一个线程A等待多线程执行完成的同步，等待指定数量的线程运行了countDownLatch.countDown()把state减到0 ，则挂起的A线程才能进入就绪状态，继续执行。因为执行await()线程要等待锁state减到0，并不一定是等待执行countDown()的线程执行完，只需state减到0的线程唤醒所有线程，除非当前线程发生中断。

```java
public static void main(String[] args) {
    CountDownLatch countDownLatch = new CountDownLatch(6);
    ExecutorService pool = Executors.newCachedThreadPool();
    for (int i = 0; i < 5; i++) {
        pool.submit(new CountDownLatchTest(countDownLatch, String.valueOf(i)));
    }
    try {
        countDownLatch.await(); // 1.若latch count 为0 则 继续执行；2.若不为0 则挂起当前线程；3.某个减少latch count到0的线程唤醒它。
        System.out.println("await()线程,任务汇总之后...");
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        pool.shutdown();
    }
}

static class CountDownLatchTest implements Runnable {
    CountDownLatch countDownLatch;
    String i;
    CountDownLatchTest(CountDownLatch countDownLatch, String i) {
        this.countDownLatch = countDownLatch;
        this.i = i;
    }
    @Override
    public void run() {
        System.out.println("hello->" + i);
        countDownLatch.countDown();
        System.out.println("sleep 3000ms");
    }
}
```



## 源码分析

1. CountDownLatch有什么特性，使用的场景有哪些？

   > 可以在一个线程中等待其他线程，然后其他线程中把state减到0的线程唤醒等待的线程。即在各自任务中的某个点停住，等待所有线程都就位之后，由最后一个到位的线程唤醒其他线程。

2. CountDownLatch的方法？

   > * countDown()
   >
   > * await()

3. 怎么实现主控线程（调用await()的线程）等待其他线程（调用countDown()方法的线程）的？

   > new CountDownLatch(n)传入了n，当n减到0的线程唤醒被await()的线程。传入的n是AQS同步器的锁变量state，当state减到0时，唤醒主线程（调用await()方法挂起的线程）。由减到0的线程唤醒被await()阻塞的线程。


### await()方法

> CountDownLatch.await()调用的是共享模式的可中断获取锁，存在多个线程调用CountDownLatch.await()的场景

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

> 若锁state大于0，则进入if内部，执行doAcquireSharedInterruptibly(arg)方法

```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

> 若锁state大于0，则调用父类AQS同步器中的doAcquireSharedInterruptibly代码，该代码实现了什么呢？

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

> 释放共享模式的锁

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

> CountDownLatch自身实现的tryReleaseShared方法，如果锁state减为0了，直接返回false。如果锁state不为0，则减一，同时返回 state == 0 的判断。

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

> 调用了父类AQS同步器的方法，在锁state减为0时调用了如下的方法，唤醒被CountDownLatch.await()阻塞的线程。

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
                unparkSuccessor(h); //唤醒后继者
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



## CountDownLatch 和 thread.join()方法的区别

* CountDownLatch
  * 在子线程未完成之前，主线程会一直处于挂起状态。（LockSupport.park())
  * 子线程的控制方法不同。CountDownLatch中，多个子线程的控制，让各自子线程去定义任务结束的条件，即调用countDown()的时机。而thread.join()是必须等待子线程执行完成，在主线程中调用thread.join()来实现的。
* thread.join()
  * 每个子线程都需要调用一遍thread.join()。（join需要线程实例来调用)
  * 若开启了多个子线程，然后调用了在主线程里使用子线程实例调用了join方法，主线程是按照调用join方法的顺序一个线程一个线程地等待子线程执行完成。前面join的子线程没执行完，主线程会一直等待。(在没传入参数的情况下)
  * thread.join()方法的缺点：子线程未完成之前，主线程会一直运行就绪挂起，运行就绪挂起三种状态之间流转，比较耗费资源。

### thread.join()示例

```java
public static void main(String[] args) throws InterruptedException {
    Thread thread = new Thread(()->{
        try {
            Thread.sleep(2000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("join thread");
    });
    thread.start();
    thread.join();
    System.out.println("main thread");
}
```

> 结果

```
join thread
main thread
```



## CountDownLatch（闭锁） 和 CyclicBarrier（栅栏）的区别

* CountDownLatch（闭锁）是一个线程等待其他多个线程执行完指定代码之后才能被唤醒。被阻塞的的线程只有一个（即等待其他线程执行完的线程），其他执行任务的线程不会阻塞。

* CyclicBarrier（栅栏）是多个线程到达了各自线程内部的指定的点（调用await()的代码行），然后挂起，指定数量的线程中最后一个到达的线程会启动回调线程，然后唤醒之前挂起的所有线程。

