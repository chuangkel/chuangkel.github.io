---
layout:     post
title:	JUC专题
subtitle: 	ReentrantReadWriteLock
date:       2019-08-04
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# ReentrantReadWriteLock
> 读写锁都是同一把锁，都是父类AQS同步器的锁state

## 问题

1. 读锁怎么实现共享的？即不互斥的？
2. 写锁到读锁是怎么降价的 
3. 



## 锁降级



## 代码体验
### 实例： 读锁和读锁之间的关系

```java
public static void main(String[] args) {
    ThreadPoolExecutor executor = new ThreadPoolExecutor(5,10,1000L,
            TimeUnit.MILLISECONDS,new LinkedBlockingDeque<>(),new PersonThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy());

    ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    CyclicBarrier cyclicBarrier = new CyclicBarrier(2);
    //读锁和读锁之间的关系
    for(int i = 0; i < 2; i++){
        executor.submit(()->{
            readAndRead(rwLock,cyclicBarrier);
        });
    }
}
public static void readAndRead(ReentrantReadWriteLock rwLock, CyclicBarrier cyclicBarrier){
    rwLock.readLock().lock();
    try {
        cyclicBarrier.await();
        for(int i = 0; i < 20; i++){
            Thread.sleep(200L);//挂起线程，让实验更直观
            System.out.println("threadGroupName: " + Thread.currentThread().getThreadGroup().getName()+
                    ", threadName: "+Thread.currentThread().getName());
        }
    }catch (Exception e){
    }finally {
        rwLock.readLock().unlock();
    }
}
static class PersonThreadFactory implements ThreadFactory{
    ThreadGroup group = new ThreadGroup("person");
    private final AtomicInteger atomic = new AtomicInteger(1);
    private final String prefix = "pthread-";
    @Override
    public Thread newThread(Runnable r) {
        return new Thread(group,r,prefix + atomic.getAndIncrement());
    }
}
```

> 读锁之间是共享的，由下可以看出是交替执行的，多个读线程之间共享了锁，执行了同一段代码块。

```
threadGroupName: person, threadName: pthread-1
threadGroupName: person, threadName: pthread-2
threadGroupName: person, threadName: pthread-2
threadGroupName: person, threadName: pthread-1
threadGroupName: person, threadName: pthread-1
threadGroupName: person, threadName: pthread-2
//...
```



### 实例：读锁和写锁的关系

```java
public static void main(String[] args) {
    ThreadPoolExecutor executor = new ThreadPoolExecutor(5,10,1000L,
            TimeUnit.MILLISECONDS,new LinkedBlockingDeque<>(),new PersonThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy());

    ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    CyclicBarrier cyclicBarrier = new CyclicBarrier(2);
    //读锁和写锁之间的关系
    executor.submit(()->{
        readAction(rwLock,cyclicBarrier);
    });
    executor.submit(()->{
        writeAction(rwLock,cyclicBarrier);
    });
}
public static void readAction(ReentrantReadWriteLock rwLock, CyclicBarrier cyclicBarrier){
    rwLock.readLock().lock();
    try {
        //cyclicBarrier.await();
        for(int i = 0; i < 20; i++){
            Thread.sleep(100L);
            System.out.println("threadGroupName: " + Thread.currentThread().getThreadGroup().getName()+
                    ", threadName: "+Thread.currentThread().getName()+", 正在读...");
        }
    }catch (Exception e){
    }finally {
        rwLock.readLock().unlock();
    }
}
public static void writeAction(ReentrantReadWriteLock rwLock, CyclicBarrier cyclicBarrier){
    rwLock.writeLock().lock();
    try {
        // 这个同步需去掉，若获得锁到了这里，又将当下线程挂起，挂起不会释放锁？，造成死锁。
        //cyclicBarrier.await(); 
        for(int i = 0; i < 20; i++){
            Thread.sleep(100L);
            System.out.println("threadGroupName: " + Thread.currentThread().getThreadGroup().getName()+
                    ", threadName: "+Thread.currentThread().getName()+", 正在写...");
        }
    }catch (Exception e){
    }finally {
        rwLock.writeLock().unlock();
    }
}
```

> 读锁和写锁互斥，是怎么实现的？

```
//...
threadGroupName: person, threadName: pthread-1, 正在读...
threadGroupName: person, threadName: pthread-1, 正在读...
threadGroupName: person, threadName: pthread-2, 正在写...
threadGroupName: person, threadName: pthread-2, 正在写...
//...
```



### 实例：写锁和写锁之间的关系

> 写锁和写锁之间排斥

```java
public static void main(String[] args) {
    ThreadPoolExecutor executor = new ThreadPoolExecutor(5,10,1000L,
            TimeUnit.MILLISECONDS,new LinkedBlockingDeque<>(),new PersonThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy());

    ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    CyclicBarrier cyclicBarrier = new CyclicBarrier(2);
    //写锁和写锁之间的关系
    for(int i = 0; i < 2; i++){
        executor.submit(()->{
            writeAndWrite(rwLock,cyclicBarrier);
        });
    }
}
public static void writeAndWrite(ReentrantReadWriteLock rwLock, CyclicBarrier cyclicBarrier){
    rwLock.writeLock().lock();
    try {
        //cyclicBarrier.await();
        for(int i = 0; i < 20; i++){
            Thread.sleep(200L);
            System.out.println("threadGroupName: " + Thread.currentThread().getThreadGroup().getName()+
                    ", threadName: "+Thread.currentThread().getName());
        }
    }catch (Exception e){
    }finally {
        rwLock.writeLock().unlock();
    }
}
```

> 输出结果如下：

```java
//...
threadGroupName: person, threadName: pthread-1
threadGroupName: person, threadName: pthread-1
threadGroupName: person, threadName: pthread-2
threadGroupName: person, threadName: pthread-2
//...
```

