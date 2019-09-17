---
layout:     post
title:	ThreadPoolExecutor
subtitle: 	ThreadPoolExecutor
date:       2019-08-09
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# ThreadPoolExecutor

### 问题

1. 为什么有线程池的存在？
2. ThreadPoolExecutor是什么？
3. 线程池的数据结构？
4. 线程池原理是什么？怎么管理多线程的？
5. 线程池的状态有哪些？
6. 线程池六大参数分析? 用阿里规范创建线程池
7. 线程池是怎么实现线程复用的 ？

### 数据结构

```java
private final BlockingQueue<Runnable> workQueue;
private final ReentrantLock mainLock = new ReentrantLock();
/**
 * Set containing all worker threads in pool. Accessed only when
 * holding mainLock.
 */
private final HashSet<Worker> workers = new HashSet<Worker>();
/**
 * Wait condition to support awaitTermination
 */
private final Condition termination = mainLock.newCondition();
/**
 * Tracks largest attained pool size. Accessed only under
 * mainLock.
 */
private int largestPoolSize;
/**
 * Counter for completed tasks. Updated only on termination of
 * worker threads. Accessed only under mainLock.
 */
private long completedTaskCount;
private volatile ThreadFactory threadFactory;
/**
 * Handler called when saturated or shutdown in execute.
 */
private volatile RejectedExecutionHandler handler;
/**
 * Timeout in nanoseconds for idle threads waiting for work.
 * Threads use this timeout when there are more than corePoolSize
 * present or if allowCoreThreadTimeOut. Otherwise they wait
 * forever for new work.
 */
private volatile long keepAliveTime;
/**
 * If false (default), core threads stay alive even when idle.
 * If true, core threads use keepAliveTime to time out waiting
 * for work.
 */
private volatile boolean allowCoreThreadTimeOut;
/**
 * Core pool size is the minimum number of workers to keep alive
 * (and not allow to time out etc) unless allowCoreThreadTimeOut
 * is set, in which case the minimum is zero.
 */
private volatile int corePoolSize;
/**
 * Maximum pool size. Note that the actual maximum is internally
 * bounded by CAPACITY.
 */
private volatile int maximumPoolSize;
/**
 * The default rejected execution handler 默认拒绝策略AbortPolicy()
 */
private static final RejectedExecutionHandler defaultHandler =
    new AbortPolicy();
```

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
//线程池容量存储在ctl低29位
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits 线程运行状态存储在ctl高3位
//11100000000000000000000000000000 高位111
private static final int RUNNING    = -1 << COUNT_BITS;
//00000000000000000000000000000000 高位000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//00100000000000000000000000000000 高位001
private static final int STOP       =  1 << COUNT_BITS;
//01000000000000000000000000000000 高位010
private static final int TIDYING    =  2 << COUNT_BITS;
//01100000000000000000000000000000 高位011
private static final int TERMINATED =  3 << COUNT_BITS;
```



```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
  	//线程池状态和线程池线程数量都存储在ctl中
    int c = ctl.get();
    //线程池线程数量小于核心线程数
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //线程池线程数量大于核心线程数，isRunning(c)判断线程池状态，if true，则添加到阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
       
        if (! isRunning(recheck) && remove(command))
            //线程池不在运行 && 移除任务成功 按照拒绝策略处理
            reject(command);
        else if (workerCountOf(recheck) == 0)
            //1. 线程池在运行 运行线程数为0 2. 线程池不在运行，移除任务失败 运行线程数为0
            addWorker(null, false);
    }
    //任务队列满了，直接运行，if 直接运行失败，按照拒绝策略处理
    else if (!addWorker(command, false))
        reject(command);
}
```



```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 线程池不是Running 
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```



```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
		
        // Check if queue empty only if necessary.
        //rs >= SHUTDOWN不是运行状态 rs>=STOP不是RUNNING和SHUTDOWN状态，队列是空
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //poll等待keepAalived若队列中仍没有元素，返回
            //take一直等待任务队列中元素有效
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```



### Worker分析

worker数据结构

```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{   //继承了Runnable和同步器
    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread; 
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);//工厂方法创建线程
    }
    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }
    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }
```

#### 调用的类外方法

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //循环 让线程一直运行，除非设置了存活时间过后回收。任务队列中获取任务时，任务为空会阻塞线程
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //在运行的线程中调用run方法执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```