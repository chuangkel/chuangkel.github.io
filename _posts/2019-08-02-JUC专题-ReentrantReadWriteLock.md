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

问题

1. 读锁怎么实现共享的？即不互斥的？
2. 写锁到读锁是怎么降价的 
3. 



## 代码

### Sync 内部类

```java
/** 同步器的读写锁实现，子类是公平和非公平锁 */
abstract static class Sync extends AbstractQueuedSynchronizer {
    /*锁状态state分成两个无符号短整型，低位代表独占模式写锁的持有数量，高位代表共享锁读锁的持有数量 */
    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

    /** 返回持有共享锁线程的数量  */
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    /** Returns the number of exclusive holds represented in count  */
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

    /**
     * A counter for per-thread read hold counts.
     * Maintained as a ThreadLocal; cached in cachedHoldCounter
     */
    static final class HoldCounter {
        int count = 0;
        // Use id, not reference, to avoid garbage retention
        final long tid = getThreadId(Thread.currentThread());
    }

    /**
     * ThreadLocal subclass. Easiest to explicitly define for sake
     * of deserialization mechanics.
     */
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }

    /**
     * The number of reentrant read locks held by current thread.
     * Initialized only in constructor and readObject.
     * Removed whenever a thread's read hold count drops to 0.
     */
    private transient ThreadLocalHoldCounter readHolds;

    /**
     * The hold count of the last thread to successfully acquire
     * readLock. This saves ThreadLocal lookup in the common case
     * where the next thread to release is the last one to
     * acquire. This is non-volatile since it is just used
     * as a heuristic, and would be great for threads to cache.
     *
     * <p>Can outlive the Thread for which it is caching the read
     * hold count, but avoids garbage retention by not retaining a
     * reference to the Thread.
     *
     * <p>Accessed via a benign data race; relies on the memory
     * model's final field and out-of-thin-air guarantees.
     */
    private transient HoldCounter cachedHoldCounter;

    /*
     * firstReader是第一个获取读锁的线程，firstReaderHoldCount是firstReader的持有count。更精确地说，firstReader是唯一的成功并发修改改变共享count从0到1的线程*/
    private transient Thread firstReader = null;
    //第一个线程的重入数量
    private transient int firstReaderHoldCount;

    Sync() {
        readHolds = new ThreadLocalHoldCounter();
        setState(getState()); // ensures visibility of readHolds
    }

    /*
     * Acquires and releases use the same code for fair and
     * nonfair locks, but differ in whether/how they allow barging
     * when queues are non-empty.
     */

    /**
     * Returns true if the current thread, when trying to acquire
     * the read lock, and otherwise eligible to do so, should block
     * because of policy for overtaking other waiting threads.
     */
    abstract boolean readerShouldBlock();

    /**
     * Returns true if the current thread, when trying to acquire
     * the write lock, and otherwise eligible to do so, should block
     * because of policy for overtaking other waiting threads.
     */
    abstract boolean writerShouldBlock();

    /*
     * Note that tryRelease and tryAcquire can be called by
     * Conditions. So it is possible that their arguments contain
     * both read and write holds that are all released during a
     * condition wait and re-established in tryAcquire.
     */
}

```

#### tryRelease

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

#### tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

#### tryAcquireShared

```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
private IllegalMonitorStateException unmatchedUnlockException() {
    return new IllegalMonitorStateException(
        "attempt to unlock read lock, not locked by current thread");
}
```

#### tryAcquireShared

```java
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    /*当前锁状态state没有写锁，readerShouldBlock方法如有同步队列有前节点，返回true,没有同步节点或队列为空，返回false*/
    if (!readerShouldBlock() && //头结点了
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    //不是头结点
    return fullTryAcquireShared(current);
}
```

#### fullTryAcquireShared

```java
/** 获取读锁的完整版，处理CAS失败，上面的tryAcquireShared没有处理可重入读 */
final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

#### tryWriteLock

```java
/**
 * Performs tryLock for write, enabling barging in both modes.
 * This is identical in effect to tryAcquire except for lack
 * of calls to writerShouldBlock.
 */
final boolean tryWriteLock() {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c != 0) {
        int w = exclusiveCount(c);
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
    }
    if (!compareAndSetState(c, c + 1))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

#### tryReadLock

```java
/**
 * Performs tryLock for read, enabling barging in both modes.
 * This is identical in effect to tryAcquireShared except for
 * lack of calls to readerShouldBlock.
 */
final boolean tryReadLock() {
    Thread current = Thread.currentThread();
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return false;
        int r = sharedCount(c);
        if (r == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return true;
        }
    }
}

protected final boolean isHeldExclusively() {
    // While we must in general read state before owner,
    // we don't need to do so to check if current thread is owner
    return getExclusiveOwnerThread() == Thread.currentThread();
}

// Methods relayed to outer class

final ConditionObject newCondition() {
    return new ConditionObject();
}

final Thread getOwner() {
    // Must read state before owner to ensure memory consistency
    return ((exclusiveCount(getState()) == 0) ?
            null :
            getExclusiveOwnerThread());
}

final int getReadLockCount() {
    return sharedCount(getState());
}

final boolean isWriteLocked() {
    return exclusiveCount(getState()) != 0;
}

final int getWriteHoldCount() {
    return isHeldExclusively() ? exclusiveCount(getState()) : 0;
}

final int getReadHoldCount() {
    if (getReadLockCount() == 0)
        return 0;

    Thread current = Thread.currentThread();
    if (firstReader == current)
        return firstReaderHoldCount;

    HoldCounter rh = cachedHoldCounter;
    if (rh != null && rh.tid == getThreadId(current))
        return rh.count;

    int count = readHolds.get().count;
    if (count == 0) readHolds.remove();
    return count;
}

/**
 * Reconstitutes the instance from a stream (that is, deserializes it).
 */
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    s.defaultReadObject();
    readHolds = new ThreadLocalHoldCounter();
    setState(0); // reset to unlocked state
}

final int getCount() { return getState(); }
```


### 方法



## 关系



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

## 读写锁的实践

> 读阻塞写，不阻塞其他读
>
> 写阻塞读，又阻塞其他写