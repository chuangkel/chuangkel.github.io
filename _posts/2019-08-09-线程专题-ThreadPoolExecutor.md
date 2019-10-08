---
layout:     post
title:	线程专题
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
//任务阻塞队列，用来存放任务，在任务大于核心线程数情况下, 入队到阻塞队列中
private final BlockingQueue<Runnable> workQueue;
//加入Worker到线程池workers并发控制
private final ReentrantLock mainLock = new ReentrantLock();
/**
 * Set containing all worker threads in pool. Accessed only when
 * holding mainLock. 线程池用来存放线程的集合
 */
private final HashSet<Worker> workers = new HashSet<Worker>();
/**
 * Wait condition to support awaitTermination
 */
private final Condition termination = mainLock.newCondition();
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

> execute执行任务，会加入到队列里面吗？

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
       //到了这里，线程池状态是RUNNING且核心线程数已满且插入到阻塞队列成功
        if (! isRunning(recheck) && remove(command))
            //线程池不在运行且移除任务成功 那么按照拒绝策略处理
            reject(command);
        else if (workerCountOf(recheck) == 0)
            //1. 线程池在运行且运行线程数为0 2. 线程池不在运行且移除任务失败(队列为空)且运行线程数为0
            addWorker(null, false);// 为什么任务是null和false（不是核心）
    }
    //任务队列满了，直接运行，if 直接运行失败，按照拒绝策略处理
    else if (!addWorker(command, false))
        reject(command);
}
```

> addWorker是干什么的？ 直接运行任务

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // Check if queue empty only if necessary.    
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
// 到了这里，说明线程池状态是RUNNING 或者 线程池状态是SHUTDOWN且新加的任务是nul且任务队列不为空
        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry; //若新增运行线程数+1成功，则跳出retry循环
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs) 
                continue retry; //多线程并发添加任务了，重来一遍
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
//到了这里，新增ctl的线程数成功
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
//线程池是RUNNING状态 或 线程池是SHUTDOWN且加入的任务为空
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w); //Worker加入到线程池
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start(); //启动线程
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

## Worker(线程+任务)分析

### worker数据结构

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

> getTask是干什么的？ 从阻塞任务队列获取任务，若队列为空，则阻塞当前线程。

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

### 调用的类外(Worker类之外)方法

> 怎么样让线程一直运行的？怎样复用线程的？即通过运行阻塞队列的任务，若阻塞队列为空，则阻塞当前线程。

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

## 用阿里规范创建线程池

### 问题

1. 核心线程数量怎么确定？

> 自定义线程工厂（参考jdk实现）

```java
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```



## Executors创建线程池（不推荐）

> newCachedThreadPool

```java
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(),
                                  threadFactory);
}
```

> newFixedThreadPool

```java
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

> newScheduledThreadPool

```java
public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
```

> newSingleThreadExecutor，为什么不用当个线程代替只有一个线程的线程池？

```java
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```