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
    Node nextWaiter; //标识当前节点的模式
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

## 代码分析


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

> 根据给的模式(独占模式和共享模式) 创建节点和入队节点

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

> 挂起线程

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//阻塞当前线程
    return Thread.interrupted(); //返回中断标识，并清除中断标识为false
}
```

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



## Condition

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

