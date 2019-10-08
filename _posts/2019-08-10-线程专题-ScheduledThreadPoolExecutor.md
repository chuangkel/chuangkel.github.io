---
layout:     post
title:	线程专题
subtitle: 	ScheduledThreadPoolExecutor
date:       2019-08-10
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

## ScheduledThreadPoolExecutor延时队列、间隔时间执行

#### 问题

1. ScheduledThreadPoolExecutor使用到的数据结构？

   > 使用数据结构堆来作为队列的，同时又是使用数组来作为堆的数据结构，为什么要使用堆呢？

2. 使用堆的数据结构，怎么样实现排序的呢？有什么方便之处吗？

3. 堆的数据结构是什么样的？

4. 怎么实现定时执行任务的呢？

### DelayedWordQueue(线程池ThreadPoolExecutor定义)

   #### 堆数据结构

   - 大顶堆 父节点大于左右节点，左节点大于右节点。
   - 小顶堆 父节点是小于左右节点，左节点要小于右节点。

   > 延时任务队列，队列的元素先后顺序在元素加入的时候就进行了排序，排序依据是任务的time值。入队和出队操作需同步控制，因为线程池空闲线程可能不止一个，存在多线程取队列任务的情况。

   ```java
static class DelayedWorkQueue extends AbstractQueue<Runnable>
       implements BlockingQueue<Runnable> {
       private static final int INITIAL_CAPACITY = 16;//队列的初始容量16
       private RunnableScheduledFuture<?>[] queue =
           new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
       private final ReentrantLock lock = new ReentrantLock();
       private int size = 0;
       private Thread leader = null;
       private final Condition available = lock.newCondition();
       //...
    }
   ```

   > 队列(数组)扩容，队列容量上限为Integer.MAX_VALUE，每次增长原队列容量的一半。

   ```java
private void grow() {
    int oldCapacity = queue.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%
    if (newCapacity < 0) // overflow
        newCapacity = Integer.MAX_VALUE;
    queue = Arrays.copyOf(queue, newCapacity);
}
   ```
   >添加元素到队列，需并发控制，比较任务time值。堆数据结构的上浮。

   ```java
public boolean offer(Runnable x) {
    if (x == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = size;
        if (i >= queue.length)
            grow();
        size = i + 1;
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0);
        } else {
            siftUp(i, e);
        }
        if (queue[0] == e) {
            leader = null;
            available.signal();
        }
    } finally {
        lock.unlock();
    }
    return true;
}
   ```

   > 设置任务在堆中的下标idx。

   ```java
   private void setIndex(RunnableScheduledFuture<?> f, int idx) {
       if (f instanceof ScheduledFutureTask)
           ((ScheduledFutureTask)f).heapIndex = idx;
   }
   ```


> 堆下沉。

   ```java
   private void siftDown(int k, RunnableScheduledFuture<?> key) {
       int half = size >>> 1;
       while (k < half) {
           int child = (k << 1) + 1;
           RunnableScheduledFuture<?> c = queue[child];
           int right = child + 1;
           if (right < size && c.compareTo(queue[right]) > 0)
               c = queue[child = right];
           if (key.compareTo(c) <= 0)
               break;
           queue[k] = c;
           setIndex(c, k);
           k = child;
       }
       queue[k] = key;
       setIndex(key, k);
   }
   ```

> 堆上浮

   ```java
   private void siftUp(int k, RunnableScheduledFuture<?> key) {
       while (k > 0) {
           int parent = (k - 1) >>> 1;
           RunnableScheduledFuture<?> e = queue[parent];
           if (key.compareTo(e) >= 0)
               break;
           queue[k] = e;
           setIndex(e, k);
           k = parent;
       }
       queue[k] = key;
       setIndex(key, k);
   }
   ```

> 任务比较函数，根据ScheduledFutureTask.time来比较先后执行顺序

```java
public int compareTo(Delayed other) {
    if (other == this) // compare zero if same object
        return 0;
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        long diff = time - x.time;
        if (diff < 0)
            return -1;
        else if (diff > 0)
            return 1;
        else if (sequenceNumber < x.sequenceNumber)
            return -1;
        else
            return 1;
    }
    long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
    return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
}
```


### ScheduleThreadPoolExecutor线程池方法分析

> 

```java
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay,
                                   TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> t = decorateTask(command,
        new ScheduledFutureTask<Void>(command, null,
                                      triggerTime(delay, unit)));
    delayedExecute(t);
    return t;
}
```




> scheduleAtFixedRate延时initialDelay执行，每隔period周期执行任务，若隔period时间周期前一次任务还没执行完成，则等待前一次任务执行完成之后再开始重新执行任务。若上个任务提前执行完成，则需等待时间到两次任务开始执行间隔时间周期为period。

```java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                              long initialDelay,
                                              long period,
                                              TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(period));//传入周期
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}
```

> scheduleWithFixedDelay 任务执行的间隔时间为delay，等待任务执行之后再等待delay时长，再重新执行任务，不管任务执行的时间长短，只管两次执行任务的间隔时间，即前一次完成时间和后一次的开始时间的间隔。

```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (delay <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(-delay));//why？
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}
```

### 怎么实现重复执行任务的

> ScheduledFutureTask延时（间隔执行）任务定义类

   ```java
private class ScheduledFutureTask<V>
           extends FutureTask<V> implements RunnableScheduledFuture<V> {
       /** Sequence number to break ties FIFO */
       private final long sequenceNumber;
       /** The time the task is enabled to execute in nanoTime units */
       private long time;
       /**
        * Period in nanoseconds for repeating tasks.  A positive
        * value indicates fixed-rate execution.  A negative value
        * indicates fixed-delay execution.  A value of 0 indicates a
        * non-repeating task. 正值:固定速率循环 负值：固定延时循环 0:不循环
        */
       private final long period;
       /** The actual task to be re-enqueued by reExecutePeriodic */
       RunnableScheduledFuture<V> outerTask = this;
   	//记录当前任务在堆中的index下标
       int heapIndex;
       //...
    }
   ```

