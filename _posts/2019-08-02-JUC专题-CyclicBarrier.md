---
layout:     post
title:	JUC专题
subtitle: 	CyclicBarrier
date:       2019-08-02
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
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();//执行所有线程到达之后的任务
                ranAction = true;
                nextGeneration();
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
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);//返回nanos会小于0吗
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