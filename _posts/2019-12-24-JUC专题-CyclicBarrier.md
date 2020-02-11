---
layout:     post
title:	JUC专题
subtitle: 	CyclicBarrier
date:       2019-12-24
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# CyclicBarrier

## 问题

1. CyclicBarrier有哪些方法？
2. CyclicBarrier怎么实现多线停住？ 在停住所有线程到位之后并发执行的？
3. 栅栏怎么实现多次重复使用的？ 有使用场景吗？

## 示例

```java
public static void main(String[] args) {
    int NUM = 3;
    ExecutorService executorService = Executors.newCachedThreadPool();
    
    CyclicBarrier cyclicBarrier = new CyclicBarrier(NUM, ()->{
        System.out.println("各个线程都执行完成，本线程汇总一下...");
    });
    for (int i = 0; i < NUM; i++) {
        executorService.submit(new CylicBarrierTest(String.valueOf(i), cyclicBarrier));
    }
    executorService.shutdown();
}
static class CylicBarrierTest implements Runnable {
    private String i;
    private CyclicBarrier cyclicBarrier;
    CylicBarrierTest(String i, CyclicBarrier cyclicBarrier) {
        this.i = i;
        this.cyclicBarrier = cyclicBarrier;
    }
    @Override
    public void run() {
        System.out.println("hello" + i);
        try {
            cyclicBarrier.await(); //多线程在这里停止
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println(i + ": await之后");
    }
}
```

> 输出结果

```
hello2
hello1
hello0
各个线程都执行完成，本线程汇总一下... //回调线程执行的结果
0: await之后 
1: await之后
2: await之后
```



## 源码分析

### CyclicBarrier

```java
public class CyclicBarrier {
    /**
     * Each use of the barrier is represented as a generation instance.
     * The generation changes whenever the barrier is tripped, or
     * is reset. There can be many generations associated with threads
     * using the barrier - due to the non-deterministic way the lock
     * may be allocated to waiting threads - but only one of these
     * can be active at a time (the one to which {@code count} applies)
     * and all the rest are either broken or tripped.
     * There need not be an active generation if there has been a break
     * but no subsequent reset.
     */
    private static class Generation {
        boolean broken = false;
    }

    /** The lock for guarding barrier entry */
    private final ReentrantLock lock = new ReentrantLock();
    /** Condition to wait on until tripped */
    private final Condition trip = lock.newCondition();
    /** The number of parties */
    private final int parties;
    /* The command to run when tripped */
    private final Runnable barrierCommand;
    /** The current generation */
    private Generation generation = new Generation();

    /**
     * Number of parties still waiting. Counts down from parties to 0
     * on each generation.  It is reset to parties on each new
     * generation or when broken.
     */
    private int count;

    /**
     * Updates state on barrier trip and wakes up everyone.
     * Called only while holding lock.
     */
    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }

    /**
     * Sets current barrier generation as broken and wakes up everyone.
     * Called only while holding lock.
     */
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }
}
```

### 内部方法

#### await

> 多线程在栅栏停住，指定数量线程到位之后同时被唤醒

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

#### dowait

> 调用了dowait()

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count;//此处需同步
        if (index == 0) {  // tripped，栅栏倾倒
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();//执行所有线程到达之后的任务
                ranAction = true;
                nextGeneration();//status值减到0时，进入下一代，status又变为初始值parties，把条件队列的节点加入到同步队列（是线程安全的，外面加了重入锁），该线程unlock后会唤醒前面加到同步队列的线程。
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                    trip.await(); //关键：加入到条件队列，等到栅栏倾倒时把条件队列的节点加入到同步队列
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {//在定时等待情况下
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```



### 内部类

```java
/**
 * Each use of the barrier is represented as a generation instance.
 * The generation changes whenever the barrier is tripped, or
 * is reset. There can be many generations associated with threads
 * using the barrier - due to the non-deterministic way the lock
 * may be allocated to waiting threads - but only one of these
 * can be active at a time (the one to which {@code count} applies)
 * and all the rest are either broken or tripped.
 * There need not be an active generation if there has been a break
 * but no subsequent reset.
 */
private static class Generation {
    boolean broken = false;
}
```