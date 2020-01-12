---
clayout:     post
title:		JUC专题
subtitle: 	AbstractQueuedSynchronizer同步器
date:       2019-08-01
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# AbstractQueuedSynchronizer同步器

> Head是Node.thread == null的节点，并不属于同步器队列

### 问题

1. 同步器是什么，使用场景有哪些？

   ```
   Provides a framework for implementing blocking locks and related
   * synchronizers (semaphores, events, etc) that rely on
   * first-in-first-out (FIFO) wait queues.  This class is designed to
   * be a useful basis for most kinds of synchronizers that rely on a
   * single atomic {@code int} value to represent state. Subclasses
   * must define the protected methods that change this state, and which
   * define what that state means in terms of this object being acquired
   * or released.  Given these, the other methods in this class carry
   * out all queuing and blocking mechanics. Subclasses can maintain
   * other state fields, but only the atomically updated {@code int}
   * value manipulated using methods {@link #getState}, {@link
   * #setState} and {@link #compareAndSetState} is tracked with respect
   * to synchronization.
   ```

   > AQS同步器是一个框架，为了实现阻塞锁和与阻塞锁关联的同步器（synchronizer），比如共享信号量（semaphores）等，它们依赖于一个先进先出的等待队列。AQS同步器作为大多数同步器的基类。

2. 同步器的数据结构？

3. 同步器中的ConditionObject类解析。

4. 什么是共享模式和独占模式？ 有什么区别？

5. 使用aqs实现自己的锁？

6. 什么时候唤醒后继者呢？

7. 重入是怎么实现的？

  ​    


### 同步队列

> 同步队列以链表方式，具有共享模式和独占模式
>
> sync队列和condition队列

```java
static final class Node {
    //用同一个对象 来标识线程获取的锁是共享的，在已占用的情况下，其他线程的共享模式可以进入锁
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null; //标识该节点的线程获取的锁是独占的
    static final int CANCELLED =  1; //标识当前节点的线程已取消
    static final int SIGNAL    = -1;//唤醒下一个节点，什么时候唤醒呢？
    static final int CONDITION = -2;//正在等待什么条件？
    static final int PROPAGATE = -3;//标识在共享模式下面需要传播，怎么传播？
    volatile int waitStatus;//取值 CANCELLED、SIGNAL、CONDITION、PROPAGATE
    volatile Node prev;//同步队列中元素的上一个节点
    volatile Node next;//同步队列中元素的下一个节点
    volatile Thread thread; //入队列时的当前线程
    Node nextWaiter; //标识下一个节点的模式
    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode; //标识当前节点的模式
        this.thread = thread;
    }
}
```

#### waitStatus属性表

| waitStatus    | 含义                                                         |
| ------------- | ------------------------------------------------------------ |
| SIGNAL(-1)    | 当前节点线程的继任者处于被阻塞状态，当前线程在释放或取消时，必须unpark（启动）它的继任者。 |
| CANCELLED(1)  | 当前节点线程因为中断或者超时，被标识为CANCELLED。            |
| CONDITION(-2) | 当前节点线程处于condition队列中。nextWaiter存储condition队列中的后继节点 |
| PROPAGATE(-3) | 当前节点线程把共享消息传递给其他节点。                       |
| 0             | 当前节点线程处于sync队列中。                                 |



AQS共享模式和独占模式有什么区别

共享模式实现了多线程并发读，阻塞写，读线程释放锁之后才能读

独占模式实现了写线程阻塞读写线程

## 方法汇总

### 供Lock接口调用（代理）的方法

| 方法                                           | 说明                         | 调用者/使用者                                     |
| ---------------------------------------------- | ---------------------------- | ------------------------------------------------- |
| final void acquire(int arg)                    | 独占模式下获取锁，忽略中断   |                                                   |
| final boolean release(int arg)                 | 独占模式下释放锁             |                                                   |
| final void acquireInterruptibly(int arg)       | 独占模式下获取锁，可响应中断 |                                                   |
| final void acquireSharedInterruptibly(int arg) | 共享模式下获取锁，可响应中断 |                                                   |
| final void acquireShared(int arg)              | 共享模式下获取锁，忽略中断   | TwinsLock、Semaphore、ReentrantReadWrite.ReadLock |
| final boolean releaseShared(int arg)           | 共享模式下释放锁             |                                                   |

### 需子类实现的方法

> 子类实现AQS同步器过程中需调用getState()、setState(int newState)和compareAndSetState(int expect, int update)来操作锁状态

| 方法                                        | 说明                                               |
| ------------------------------------------- | -------------------------------------------------- |
| protected boolean tryAcquire(int arg)       | 在独占模式下尝试获取锁，会操纵锁状态来实现获取逻辑 |
| protected boolean tryRelease(int arg)       | 在独占模式下释放锁，会操纵锁状态                   |
| protected int tryAcquireShared(int arg)     | 在共享模式下尝试获取锁                             |
| protected boolean tryReleaseShared(int arg) | 在共享模式下释放锁                                 |
| protected final boolean isHeldExclusively() | 在独占模式下，查询当前线程是否持有锁               |

## 独占模式和共享模式的区别

* 获取锁
  * 共享模式
  * 独占模式

* 释放锁
	* 共享模式：doReleaseShared()
	* 独占模式：release()
* 都调用了如下的代码


## 独占模式

> 释放独占模式的操作

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```


## 共享模式



## 子类需实现的方法

* int tryAcquireShared(int arg) 用共享模式获取锁，



## Node

```java
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    static final Node EXCLUSIVE = null;

    /** waitStatus value to indicate thread has cancelled */
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    static final int PROPAGATE = -3;

    /**
     * Status field, taking on only the values:
     *   SIGNAL:     The successor of this node is (or will soon be)
     *               blocked (via park), so the current node must
     *               unpark its successor when it releases or
     *               cancels. To avoid races, acquire methods must
     *               first indicate they need a signal,
     *               then retry the atomic acquire, and then,
     *               on failure, block.
     *   CANCELLED:  This node is cancelled due to timeout or interrupt.
     *               Nodes never leave this state. In particular,
     *               a thread with cancelled node never again blocks.
     *   CONDITION:  This node is currently on a condition queue.
     *               It will not be used as a sync queue node
     *               until transferred, at which time the status
     *               will be set to 0. (Use of this value here has
     *               nothing to do with the other uses of the
     *               field, but simplifies mechanics.)
     *   PROPAGATE:  A releaseShared should be propagated to other
     *               nodes. This is set (for head node only) in
     *               doReleaseShared to ensure propagation
     *               continues, even if other operations have
     *               since intervened.
     *   0:          None of the above
     *
     * The values are arranged numerically to simplify use.
     * Non-negative values mean that a node doesn't need to
     * signal. So, most code doesn't need to check for particular
     * values, just for sign.
     *
     * The field is initialized to 0 for normal sync nodes, and
     * CONDITION for condition nodes.  It is modified using CAS
     * (or when possible, unconditional volatile writes).
     */
    volatile int waitStatus;

    /**
     * Link to predecessor node that current node/thread relies on
     * for checking waitStatus. Assigned during enqueuing, and nulled
     * out (for sake of GC) only upon dequeuing.  Also, upon
     * cancellation of a predecessor, we short-circuit while
     * finding a non-cancelled one, which will always exist
     * because the head node is never cancelled: A node becomes
     * head only as a result of successful acquire. A
     * cancelled thread never succeeds in acquiring, and a thread only
     * cancels itself, not any other node.
     */
    volatile Node prev;

    /**
     * Link to the successor node that the current node/thread
     * unparks upon release. Assigned during enqueuing, adjusted
     * when bypassing cancelled predecessors, and nulled out (for
     * sake of GC) when dequeued.  The enq operation does not
     * assign next field of a predecessor until after attachment,
     * so seeing a null next field does not necessarily mean that
     * node is at end of queue. However, if a next field appears
     * to be null, we can scan prev's from the tail to
     * double-check.  The next field of cancelled nodes is set to
     * point to the node itself instead of null, to make life
     * easier for isOnSyncQueue.
     */
    volatile Node next;

    /**
     * The thread that enqueued this node.  Initialized on
     * construction and nulled out after use.
     */
    volatile Thread thread;

    /**
     * Link to next node waiting on condition, or the special
     * value SHARED.  Because condition queues are accessed only
     * when holding in exclusive mode, we just need a simple
     * linked queue to hold nodes while they are waiting on
     * conditions. They are then transferred to the queue to
     * re-acquire. And because conditions can only be exclusive,
     * we save a field by using special value to indicate shared
     * mode.
     */
    Node nextWaiter;

    /**
     * Returns true if node is waiting in shared mode.
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /** 返回当前节点的上一个节点，当前节点为该方法的调用对象。 */
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
    
    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```



## AQS方法

### acquireQueued

用于处于独占模式且已经处于同步队列中的线程，主要是为了维护节点唤醒功能。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### doAcquireShared


> 在共享模式获取锁。获取锁的操作都是需要入同步队列操作的，之后若能获取锁则获取，不能则挂起线程。

```java
private void doAcquireShared(int arg) {
    // 共享模式都是同一个对象Node.SHARED（静态类里面定义的），加入到同步队列。
    final Node node = addWaiter(Node.SHARED); 
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor(); //刚入同步队列节点的前一个节点
            if (p == head) {
                //如果前一个节点是头结点
                int r = tryAcquireShared(arg); //该类由子类实现，判断锁state是否==0
                 //参考： 若 r >= 0 则state == 0, 若 r < 0 则锁state>0。每个子类实现不同
                if (r >= 0) {
                    //到了这里，锁空闲，可以获取
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted) 
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### addWaiter

> 根据给的模式(独占模式和共享模式) 创建节点并且插入到同步队列尾部(控制了并发）

```java
/** 
 * Creates and enqueues node for current thread and given mode.
 * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
 * @return the new node
 */
private Node addWaiter(Node mode) {
    //mode: 共享模式下是Node.SHARED 独占模式下是Node.EXCLUSIVE即null
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) { //cas设置tail,pred是预期值，node是更新值
            pred.next = node;
            return node;
        }
    }
    //到了这里 队列为空 或者 加入到同步队列并发失败
    enq(node);
    return node;
}
```
### setHeadAndPropagate

> 设置当前node为head，即获取到了锁了。 这个地方有并发吗？有并发但并不影响，相当于设置头结点，

```java
private void setHeadAndPropagate(Node node, int propagate) {
    //node 是获取锁新建的节点，propagate是否可以获取锁 progagate >= 0 可以获取锁
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    //propagate什么时候大于0呢 h.waitStatus < 0线程不是取消状态。
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        //到了这里 当前节点是共享模式吗？
        if (s == null || s.isShared()) // node的下一个节点不为空且是共享模式
            doReleaseShared(); //释放共享
    }
}
```
###doReleaseShared

> 释放共享模式的操作

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h); //唤醒一个后继者
            }
            // loop on failed CAS 为什么把head的waitStatus置为了-3 PROPAGATE
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                
        }
        // loop if head changed 什么情况下head会改变呢？
        if (h == head)                   
            break;
    }
}
```
### unparkSuccessor

> 唤醒node的后继者，不是唤醒node。

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    // 到了这里，当前node的waitStatus有两种 1. 0 2. 1当前节点线程已取消
    if (s == null || s.waitStatus > 0) {
        //当前节点node是最后一个节点 或 其线程已取消等待
        s = null;
        //遍历 去掉已取消等待的节点 
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //以上：1. 找到node的后继节点中第一个不是取消的节点 2. 把当前node的waitStatus置为0
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

### shouldParkAfterFailedAcquire

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

### parkAndCheckInterrupt

> 挂起线程

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//阻塞当前线程
    return Thread.interrupted(); //返回中断标识，并清除中断标识为false
}
```

### hasQueuedPredecessors

判断同步队列中当前线程是否有前辈节点，若有前辈节点，返回true。若当前线程是头结点（head.next）或者同步队列为空，返回false。

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

### transferForSignal

转换节点从条件队列到同步队列

```java
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

## Condition实现类

```java
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;
```


#### unlinkCancelledWaiters

```java
    private void unlinkCancelledWaiters() {
        Node t = firstWaiter;
        Node trail = null;
        while (t != null) {
            Node next = t.nextWaiter;
            if (t.waitStatus != Node.CONDITION) {
                t.nextWaiter = null;
                if (trail == null)
                    firstWaiter = next;
                else
                    trail.nextWaiter = next;
                if (next == null)
                    lastWaiter = trail;
            }
            else
                trail = t;
            t = next;
        }
    }
```
#### signal

```java
    public final void signal() {//唤醒一个Condition节点
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);
    }
```
```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null); //只唤醒头结点
}
```

#### signalAll

```java
    public final void signalAll() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignalAll(first);
    }
```
```java
//将Condition队列的所有Condition节点放入到同步节点中并唤醒它们
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);//同步器方法，转换节点从条件队列到同步队列
        first = next;
    } while (first != null);
}	
```

```java
    /**
     * Implements uninterruptible condition wait.
     * <ol>
     * <li> Save lock state returned by {@link #getState}.
     * <li> Invoke {@link #release} with saved state as argument,
     *      throwing IllegalMonitorStateException if it fails.
     * <li> Block until signalled.
     * <li> Reacquire by invoking specialized version of
     *      {@link #acquire} with saved state as argument.
     * </ol>
     */
    public final void awaitUninterruptibly() {
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        boolean interrupted = false;
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            if (Thread.interrupted())
                interrupted = true;
        }
        if (acquireQueued(node, savedState) || interrupted)
            selfInterrupt();
    }

    /*
     * For interruptible waits, we need to track whether to throw
     * InterruptedException, if interrupted while blocked on
     * condition, versus reinterrupt current thread, if
     * interrupted while blocked waiting to re-acquire.
     */

    /** Mode meaning to reinterrupt on exit from wait */
    private static final int REINTERRUPT =  1;
    /** Mode meaning to throw InterruptedException on exit from wait */
    private static final int THROW_IE    = -1;

    /**
     * Checks for interrupt, returning THROW_IE if interrupted
     * before signalled, REINTERRUPT if after signalled, or
     * 0 if not interrupted.
     */
    private int checkInterruptWhileWaiting(Node node) {
        return Thread.interrupted() ?
            (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
            0;
    }

    /**
     * Throws InterruptedException, reinterrupts current thread, or
     * does nothing, depending on mode.
     */
    private void reportInterruptAfterWait(int interruptMode)
        throws InterruptedException {
        if (interruptMode == THROW_IE)
            throw new InterruptedException();
        else if (interruptMode == REINTERRUPT)
            selfInterrupt();
    }
```
#### await

```java
    /**
     * Implements interruptible condition wait.
     * <ol>
     * <li> If current thread is interrupted, throw InterruptedException.
     * <li> Save lock state returned by {@link #getState}.
     * <li> Invoke {@link #release} with saved state as argument,
     *      throwing IllegalMonitorStateException if it fails.
     * <li> Block until signalled or interrupted.
     * <li> Reacquire by invoking specialized version of
     *      {@link #acquire} with saved state as argument.
     * <li> If interrupted while blocked in step 4, throw InterruptedException.
     * </ol>
     */
    public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }
```
#### awaitNanos

```java
    /**
     * Implements timed condition wait.
     * <ol>
     * <li> If current thread is interrupted, throw InterruptedException.
     * <li> Save lock state returned by {@link #getState}.
     * <li> Invoke {@link #release} with saved state as argument,
     *      throwing IllegalMonitorStateException if it fails.
     * <li> Block until signalled, interrupted, or timed out.
     * <li> Reacquire by invoking specialized version of
     *      {@link #acquire} with saved state as argument.
     * <li> If interrupted while blocked in step 4, throw InterruptedException.
     * </ol>
     */
    public final long awaitNanos(long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        final long deadline = System.nanoTime() + nanosTimeout;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (nanosTimeout <= 0L) {
                transferAfterCancelledWait(node);
                break;
            }
            if (nanosTimeout >= spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
            nanosTimeout = deadline - System.nanoTime();
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return deadline - System.nanoTime();
    }
```
#### awaitUntil

```
    /**
     * Implements absolute timed condition wait.
     * <ol>
     * <li> If current thread is interrupted, throw InterruptedException.
     * <li> Save lock state returned by {@link #getState}.
     * <li> Invoke {@link #release} with saved state as argument,
     *      throwing IllegalMonitorStateException if it fails.
     * <li> Block until signalled, interrupted, or timed out.
     * <li> Reacquire by invoking specialized version of
     *      {@link #acquire} with saved state as argument.
     * <li> If interrupted while blocked in step 4, throw InterruptedException.
     * <li> If timed out while blocked in step 4, return false, else true.
     * </ol>
     */
    public final boolean awaitUntil(Date deadline)
            throws InterruptedException {
        long abstime = deadline.getTime();
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        boolean timedout = false;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (System.currentTimeMillis() > abstime) {
                timedout = transferAfterCancelledWait(node);
                break;
            }
            LockSupport.parkUntil(this, abstime);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return !timedout;
    }
```
#### await

```
    /**
     * Implements timed condition wait.
     * <ol>
     * <li> If current thread is interrupted, throw InterruptedException.
     * <li> Save lock state returned by {@link #getState}.
     * <li> Invoke {@link #release} with saved state as argument,
     *      throwing IllegalMonitorStateException if it fails.
     * <li> Block until signalled, interrupted, or timed out.
     * <li> Reacquire by invoking specialized version of
     *      {@link #acquire} with saved state as argument.
     * <li> If interrupted while blocked in step 4, throw InterruptedException.
     * <li> If timed out while blocked in step 4, return false, else true.
     * </ol>
     */
    public final boolean await(long time, TimeUnit unit)
            throws InterruptedException {
        long nanosTimeout = unit.toNanos(time);
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        final long deadline = System.nanoTime() + nanosTimeout;
        boolean timedout = false;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (nanosTimeout <= 0L) {
                timedout = transferAfterCancelledWait(node);
                break;
            }
            if (nanosTimeout >= spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
            nanosTimeout = deadline - System.nanoTime();
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return !timedout;
    }

    //  support for instrumentation

    /**
     * Returns true if this condition was created by the given
     * synchronization object.
     *
     * @return {@code true} if owned
     */
    final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
        return sync == AbstractQueuedSynchronizer.this;
    }

    /**
     * Queries whether any threads are waiting on this condition.
     * Implements {@link AbstractQueuedSynchronizer#hasWaiters(ConditionObject)}.
     *
     * @return {@code true} if there are any waiting threads
     * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
     *         returns {@code false}
     */
    protected final boolean hasWaiters() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION)
                return true;
        }
        return false;
    }

    /**
     * Returns an estimate of the number of threads waiting on
     * this condition.
     * Implements {@link AbstractQueuedSynchronizer#getWaitQueueLength(ConditionObject)}.
     *
     * @return the estimated number of waiting threads
     * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
     *         returns {@code false}
     */
    protected final int getWaitQueueLength() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int n = 0;
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION)
                ++n;
        }
        return n;
    }

    /**
     * Returns a collection containing those threads that may be
     * waiting on this Condition.
     * Implements {@link AbstractQueuedSynchronizer#getWaitingThreads(ConditionObject)}.
     *
     * @return the collection of threads
     * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
     *         returns {@code false}
     */
    protected final Collection<Thread> getWaitingThreads() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION) {
                Thread t = w.thread;
                if (t != null)
                    list.add(t);
            }
        }
        return list;
    }
}
```



### condition应用

> 生产者消费者设计模式实现：两个线程，A线程打印A,B线程打印B,要求打印ABABABABABABABABABAB

```java
public class PrintThread implements Runnable {
    private int COUNT = 10;
    private ReentrantLock reentrantLock;
    private Condition conditionA;
    private Condition conditionB;
    private String c;
    PrintThread(ReentrantLock reentrantLock,Condition conditionA,Condition conditionB,String c){
        this.reentrantLock = reentrantLock;
        this.conditionA = conditionA;
        this.conditionB = conditionB;
        this.c = c;
    }
    @Override
    public void run() {
        try{
            reentrantLock.lock();
            for(int i = 0;i < COUNT; i++){
                System.out.print(c);
                try {
                    conditionB.signal();
                    if(i < COUNT - 1){
                        conditionA.await();
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }finally {
            reentrantLock.unlock();
        }
    }
}

public class ReentranLockTest {
    static ReentrantLock reentrantLock = new ReentrantLock();
    static Condition conditionA = reentrantLock.newCondition();
    static Condition conditionB = reentrantLock.newCondition();
    public static void main(String[] args) {
       PrintThread printThreadA = new PrintThread(reentrantLock,conditionA,conditionB,"A");
       PrintThread printThreadB = new PrintThread(reentrantLock,conditionB,conditionA,"B");
        Thread tA = new Thread(printThreadA);
        Thread tB = new Thread(printThreadB);
        tA.start();
        tB.start();
        //输出BABABABABABABABABABA或者ABABABABABABABABABAB
    }
}
```

#### 例子：ArrayBlockingQueue中使用条件变量Condition

```java
/** Condition for waiting takes */
private final Condition notEmpty;
/** Condition for waiting puts */
private final Condition notFull;
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

### Condition是什么

> Condition是对象监视器的替代品，拓展了监视器的语义

* 相同点
  * 都有一组类似的方法
    * 对象监视器:Object.wait()、Object.wait(long timeout)、Object.notify()、Object.notifyAll()
    * Condition对象: Condition.await()、Condition.awaitNanos(long nanosTimeout)、Condition.signal()、Condition.signalAll()
  * 都需要和锁进行关联
    * 对象监视器: 需要进入synchronized语句块（进入对象监视器）才能调用对象监视器的方法
    * Condition对象:需要和一个Lock绑定

- 不同点
  - Condition拓展的语义方法
    - awaitUninterruptibly()：等待时忽略中断
    - awaitUntil(Date deadline) throws InterruptedException：等待到特定日期
  - 使用方法
    - 对象监视器: 进入synchronized语句块（进入对象监视器）后调用Object.wait()
    - Condition对象: 需要和一个Lock绑定，并显示的调用lock()获取锁，然后调用 Condition.await()
  - 等待队列数量
    - 对象监视器: 1个
    - Condition对象: 多个。通过多次调用lock.newCondition()返回多个等待队列

### await()

Condition内部定义类的方法，该方法把当前线程作为参数新建一个节点放入条件队列中(不是同步队列)，释放可重入锁状态，同时挂起当前线程。

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node); //释放锁状态
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

### 扩展：

多个Condition

state 0 

### 源码：LockSupport

> LockSupport中主要是park和unpark方法以及设置和读取parkBlocker方法。
>
> parkBlockerOffset：parkBlocker用于记录线程被谁阻塞的，用于线程监控和分析工具来定位原因的，可以通过LockSupport的getBlocker获取到阻塞的对象。parkBlocke在线程处于阻塞的情况下才会被赋值。线程都已经阻塞了，如果不通过这种内存的方法，而是直接调用线程内的方法，线程是不会回应调用的。

```java
// Hotspot implementation via intrinsics API
private static final sun.misc.Unsafe UNSAFE;
private static final long parkBlockerOffset;
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> tk = Thread.class;
        parkBlockerOffset = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("parkBlocker"));
    } catch (Exception ex) { throw new Error(ex); }
}
```



> JUC(Java Util Concurrency)仅用简单的park, unpark和CAS指令就实现了各种高级同步数据结构

#### Unsafe.park和Unsafe.unpark

> 其实park/unpark的设计原理核心是“许可”：park是等待一个许可，unpark是为某线程提供一个许可。
>
> 在Linux系统下是用的Posix线程库pthread中的mutex（互斥量），condition（条件变量）来实现的。mutex和condition保护了一个_counter的变量，当park时，这个变量被设置为0，当unpark时，这个变量被设置为1。

- **park 过程**
  * 当调用park时，先尝试能否直接拿到“许可”，即_counter>0时
    * 如果成功，则把\_counter设置为0，并返回
    * 如果不成功，则构造一个ThreadBlockInVM，然后检查\_counter是不是>0，如果是，则把\_counter设置为0，unlock mutex并返回；否则，再判断等待的时间，然后再调用pthread_cond_wait函数等待，如果等待返回，则把 \_counter设置为0，unlock mutex并返回

* **unpark 过程**
  * 当unpark时，直接设置\_counter为1，再unlock mutex返回
    * 如果\_counter之前的值是0，则还要调用pthread_cond_signal唤醒在park中等待的线程：



### 如何终止一个Java线程

对于runnable的线程，利用一个变量做标记位，定期检查

对于非runnable的线程，应该采取中断的方式退出阻塞，并处理捕获的中断异常

- 对于大部分阻塞线程的方法，使用Thread.interrupt()，可以立刻退出等待，抛出InterruptedException
    这些方法包括Object.wait(), Thread.join()，Thread.sleep()，以及各种AQS衍生类：
  - Lock.lockInterruptibly()**等任何显示声明throws InterruptedException的方法**。
  - 被阻塞的nio Channel也会响应interrupt()，抛出ClosedByInterruptException，相应nio通道需要实现java.nio.channels.InterruptibleChannel接口
  - 还有一些阻塞方法不会响应interrupt，如等待进入synchronized段、Lock.lock()。他们不能被动的退出阻塞状态。

## 趣题实践

1. 循环打印ABABAB...十次

```java
public class ReentranLockTest {
    static ReentrantLock reentrantLock = new ReentrantLock();
    static Condition conditionA = reentrantLock.newCondition();
    static Condition conditionB = reentrantLock.newCondition();
    static CountDownLatch latch = new CountDownLatch(1);

    public static void main(String[] args) {
        PrintThread printThreadA = new PrintThread(reentrantLock, conditionA, conditionB, "A", latch);
        PrintThread printThreadB = new PrintThread(reentrantLock, conditionB, conditionA, "B", latch);
        Thread tA = new Thread(printThreadA);
        Thread tB = new Thread(printThreadB);

        tB.start();
        try {
            Thread.sleep(100L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        tA.start();
    }
}
```



```java
public class PrintThread implements Runnable {
    private int COUNT = 10;
    private ReentrantLock reentrantLock;
    private Condition conditionA;
    private Condition conditionB;
    private CountDownLatch latch;
    private String c;

    PrintThread(ReentrantLock reentrantLock, Condition conditionA, Condition conditionB, String c, CountDownLatch latch) {
        this.reentrantLock = reentrantLock;
        this.conditionA = conditionA;
        this.conditionB = conditionB;
        this.c = c;
        this.latch = latch;
    }

    @Override
    public void run() {
        try {
            if (c.equals("A")) {
                //保证A先加锁
                reentrantLock.lock();
                latch.countDown();
            } else {
                latch.await();
                reentrantLock.lock();//这里会等待conditionA.await()执行才能继续
            }

            for (int i = 0; i < COUNT; i++) {
                System.out.print(c);
                try {
                    conditionB.signal(); //将一个Condition节点放入等待队列，等待执行
                    if (i < COUNT - 1) {
                        conditionA.await(); //此处是关键，会将锁state清空
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            reentrantLock.unlock();
        }
    }
}
```

#### 三线程循环打印ABC十次？

