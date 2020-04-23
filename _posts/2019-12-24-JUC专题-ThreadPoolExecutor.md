---
layout:     post
title:	线程专题
subtitle: 	ThreadPoolExecutor
date:       2019-12-24
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
8. 线程一开始就是创建了核心线程数数量的线程吗？
9. 线程池是命令模式的应用？
10. 为什么Worker要实现同步器？

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
//追踪到达的最大池大小， 记录峰值
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
            //1. 线程池在运行 且运行线程数为0
            //2. 线程池不在运行且移除任务失败 且运行线程数为0
            //在上面两种情况下，应该开一个线程来执行阻塞队列里的任务
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
        //! (rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()) 等效于
        //   rs != SHUTDOWN || firstTask != null ||  workQueue.isEmpty() 线程关闭从这里退出
        //firstTask为null的情况是为了执行任务队列里的任务。
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
		// 到了这里，说明线程池状态是RUNNING或者线程池状态是SHUTDOWN且新加的任务是nul且任务队列不为空
        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry; //若新增运行线程数+1成功，则跳出retry循环
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs) 
                continue retry; //线程池状态改变了，重来一遍
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
}
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
        //ShutDown状态并且队列是空 或者 是STOP状态
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;//返回null结束当前线程
        }
        int wc = workerCountOf(c);
        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null; //返回null结束当前线程
            continue;
        }
        try {
            //poll等待keepAalived若队列中仍没有元素，返回
            //take一直等待任务队列中元素有效 
            //若在等待期间，线程池关闭（会修改中断标识），这抛出中断异常。
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

翻译：Worker类主要保留了对正在执行任务中的线程们的控制，以及一些统计和监控。这个类合理地继承了AbstractQueuedSynchronizer同步器，为了简化获取和释放锁在每个任务执行前后，这可以防止一些中断，这些中断的目的是唤醒正在等待任务的工作线程，而不是中断正在运行的任务。它实现了一个简单的**非重入互斥独占锁**而不是用ReentrantLock重入锁，因为我们不想工作线程在执行任务期间被重入锁当他们调用线程池方法的时候，比如setCorePoolSize。此外，为了防止线程在开始运行任务之前被中断，我们设置了state为-1，并且在执行任务之前清除他为0在runWorker。

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //循环 让线程一直运行，除非设置了存活时间过后回收。任务队列中获取任务时，任务为空会阻塞线程
        //若存活线程为0的情况，会新建一个Worker(null,false)来执行队列中的任务。
        while (task != null || (task = getTask()) != null) {
            //这里不会抛出中断异常，lock实现忽略了中断了的。目的是调用shutdown时阻塞tryLock()方法，保证任务执行期间不被中断，同时阻塞线程池的关闭等待线程执行完成。
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            //如果不进入if判断内部的话清除了中断信号
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
    //线程池是Running和shutdown,原来是中断的中断信号被清除了,重新判断线程池不是Running和shutdown
                // 线程池不是Running和shutdown 且 当前线程wt没有被中断
                wt.interrupt(); //上面的情况需要中断当前线程
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

## 线程池的关闭方法

shutdown 不在接收新的任务进入阻塞队列，继续处理正在运行的任务和阻塞队列的任务。

shutdownNow 不在接收新的任务进入阻塞队列，尝试中断正在运行的线程（修改中断标识）和不再处理阻塞队列的任务并将任务列表返回。

1. **shutdown** 。线程池shutdown 停止向阻塞队列里面提交任务。

   问题：shutdown有什么功能？

   将线程池状态改为shutdown，拒绝向阻塞队列提交任务。修改空闲线程的中断标识(怎么知道是空闲线程的呢？看代码)。 这个时候会继续执行阻塞队列里的任务。

   问题：怎么做到停止向阻塞队列里面提交任务的，看下面的

   ```java
   private boolean addWorker(Runnable firstTask, boolean core) {
       retry:
       for (;;) {
           int c = ctl.get();
           int rs = runStateOf(c);
    	//! (rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()) 等效于
       //   rs != SHUTDOWN || firstTask != null ||  workQueue.isEmpty() 线程shutdown关闭从          这里退出。firstTask为null的情况是为了执行任务队列里的任务。
           if (rs >= SHUTDOWN &&
               ! (rs == SHUTDOWN &&
                  firstTask == null &&
                  ! workQueue.isEmpty()))
               return false;
           //......
       }
   }
   ```

   ```java
   public void shutdown() {
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           checkShutdownAccess();
           advanceRunState(SHUTDOWN);
           interruptIdleWorkers();
           onShutdown(); // hook for ScheduledThreadPoolExecutor
       } finally {
           mainLock.unlock();
       }
       tryTerminate();
   }
   ```

   修改空闲的线程中断标识

   ```java
   private void interruptIdleWorkers(boolean onlyOne) {
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           for (Worker w : workers) {
               Thread t = w.thread;
               //线程没有被中断 并且 不在执行(尝试获取锁成功，因为是不可重入锁)
               if (!t.isInterrupted() && w.tryLock()) { 
                   //从这里可以判断是空闲线程，空闲线程会阻塞住在获取任务的时候。这里会发生中断异常。
                   try {
                       t.interrupt();
                   } catch (SecurityException ignore) {
                   } finally {
                       w.unlock();
                   }
               }
               if (onlyOne)
                   break;
           }
       } finally {
           mainLock.unlock();
       }
   }
   ```

   这个方法是为了在调用shutdown之后占用线程的锁，使线程池中的线程阻塞住。

   ```java
   protected boolean tryAcquire(int unused) {
       if (compareAndSetState(0, 1)) {
           setExclusiveOwnerThread(Thread.currentThread());
           return true;
       }
       return false;
   }
   ```

   它是阻塞在任务队列中了，这个时候关闭线程池，这个阻塞的线程会state置成1，当获得任务的lock的时候会被阻塞呀 

2. **shutdownNow** 试图停止激活的正在执行的任务，停止对阻塞队列中任务的处理，并返回等待的任务的List，这些任务将被从阻塞队列移除。该方法不等待任务到终止。任务如果无法响应中断或许永远不会停止。

   问题：shutdownNow做了那些事：1. 将线程池状态改成STOP  2. 中断workers中的所有线程  3. 不会执行阻塞队列里的任务了（shutdown会），而是返回阻塞队列中的等待的任务。

   ```java
   public List<Runnable> shutdownNow() {
       List<Runnable> tasks;
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           checkShutdownAccess();
           advanceRunState(STOP);
           interruptWorkers();
           tasks = drainQueue();
       } finally {
           mainLock.unlock();
       }
       tryTerminate();
       return tasks;
   }
   ```

   ```java
   private void advanceRunState(int targetState) {
       for (;;) {
           int c = ctl.get();
           if (runStateAtLeast(c, targetState) ||
               ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
               //ctlOf(targetState, workerCountOf(c))将高位和低位拼在一起
               break;
       }
   }
   ```

   中断所有线程

   ```java
   private void interruptWorkers() {
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           for (Worker w : workers)
               w.interruptIfStarted();
       } finally {
           mainLock.unlock();
       }
   }
   ```

   ```java
   void interruptIfStarted() {
       Thread t; // state为-1禁止中断
       if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
           try {
               t.interrupt();
           } catch (SecurityException ignore) {
           }
       }
   }
   ```

3. **awaitTermination** 该方法会等待任务到终止。

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



ForkJoinPool

java.util.concurrent.CompletableFuture