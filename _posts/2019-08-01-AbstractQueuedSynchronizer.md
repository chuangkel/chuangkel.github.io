---
clayout:     post
title:		AbstractQueuedSynchronizer同步器
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

2. 同步器的数据结构？

3. 同步器中的ConditionObject类解析。

4. 什么是共享模式和独占模式？ 有什么区别？

5. 使用aqs实现自己的锁？

   

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


### 同步队列

> 同步队列以链表方式，具有共享模式和独占模式
>
> sync队列和condition队列

```java
static final class Node {
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;//正在等待什么条件？
    static final int PROPAGATE = -3;//标识在共享模式下面需要传播，怎么传播？
    volatile int waitStatus;//取值 CANCELLED、SIGNAL、CONDITION、PROPAGATE
    volatile Node prev;
    volatile Node next;
    volatile Thread thread; //入队列时的当前线程
    Node nextWaiter;
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

