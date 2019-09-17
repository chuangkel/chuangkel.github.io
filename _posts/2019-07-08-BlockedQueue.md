---
layout:     post
title:	BlockingQueue
subtitle: 	BlockingQueue
date:       2019-07-08
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# BlockingQueue

### 问题

1. 队列的数据结构？
2. 怎么实现多线程的添加元素和获取元素的？
3. 相关方法详解和使用实例？

### 接口方法

```java
public interface BlockingQueue<E> extends Queue<E> {
    //添加到队列尾，如果成功返回true,如果失败（队列满了）将抛出{@code IllegalStateException}
    boolean add(E e);
	//添加元素到队尾，如果成功返回true,如果添加失败则返回false,队列满不会抛出异常
    boolean offer(E e);
   	//添加元素到队尾，如果队列满了会等待有效的空间出现
    void put(E e) throws InterruptedException;
  	//设置获取元素时间
    boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;
 	//获取队头元素并移除队头元素，等待队头有效为止
    E take() throws InterruptedException;
  	//获取队列头元素并移除队头元素，允许等待timeout时间
    E poll(long timeout, TimeUnit unit)
        throws InterruptedException;
    int remainingCapacity();
    boolean remove(Object o);
    public boolean contains(Object o);
    int drainTo(Collection<? super E> c);
    int drainTo(Collection<? super E> c, int maxElements);
}
```

### ArrayBlockedQueue

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    /** The queued items */
    final Object[] items;
    /** items index for next take, poll, peek or remove */
    int takeIndex;
    /** items index for next put, offer, or add */
    int putIndex;
    /** Number of elements in the queue */
    int count;
    /** Main lock guarding all access 多线程同步使用 */
    final ReentrantLock lock;
    /** Condition for waiting takes 入队出队同步使用 */
    private final Condition notEmpty;
    /** Condition for waiting puts 入队出队同步使用（生产者消费者类似） */
    private final Condition notFull;
    transient Itrs itrs = null;
    //...
}
```

#### 入队操作

> add本质是调用了offer的实现，若offer返回false则抛异常

```java
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```

> take无限期等待直到有元素可以获取

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await(); //若队列为空，则阻塞线程知道队列有元素，被其他线程
        return dequeue(); //出队，并唤醒入队操作（因为出队有空闲空间出现）
    } finally {
        lock.unlock();
    }
}
```

> 不带等待时间的poll出队操作，若为空队列，直接返回

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
```

> 带等待时间的poll的出队

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

> 公用出队操作，唤醒入队操作

```java
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal(); //唤醒notFull
    return x;
}
```

#### 出队操作

> put无限期等待，知道有效空间出现进行插入到队尾

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length) 
            notFull.await(); //若队列已满，则等待有效空间出现
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```

> 等待timeout时间，时间到返回或有效空间出现进行插入到队尾

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    checkNotNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos); // 等待nonos时间
        }
        enqueue(e); //入队 并唤醒出队操作 ，因为有元素可消费
        return true;
    } finally {
        lock.unlock();
    }
}
```

> 公用入队方法，同时唤醒出队操作

```java
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    notEmpty.signal(); //唤醒notEmpty 即出队操作
}
```