---
layout:     post
title:	Collection专题
subtitle: 	 
date:       2019-12-24
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# ArrayList

```java
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}

private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```

# HashMap

hash值计算过程，把hashCode的高位和低位做异或运行，作为hash值的低位。因为(hash & n - 1) 作为数组的索引下标，大多数情况都是使用低位的，把高位也加入到hash值低位的计算，保证了低位hash值的散列均匀程度。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

红黑树的五个特性

1. 树节点只有红节点和黑节点构成。
2. 根节点是黑节点。
3. 叶子节点是黑节点。
4. 黑高相同，即从根节点到叶子结点的黑色节点数量相同。
5. 不存在两个直接相连的红色节点。

通过上面的五个特性保证了红黑树的自适应平衡，节点数指定情况下不会出现深度过深，对检索查找节点的时间复杂度的稳定性很好，时间复杂度为logn

```java
//红黑树左旋
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                      TreeNode<K,V> p) {
    TreeNode<K,V> r, pp, rl;
    if (p != null && (r = p.right) != null) {
        if ((rl = p.right = r.left) != null)
            rl.parent = p;
        if ((pp = r.parent = p.parent) == null)
            (root = r).red = false;
        else if (pp.left == p)
            pp.left = r;
        else
            pp.right = r;
        r.left = p;
        p.parent = r;
    }
    return root;
}
```

扩容

jdk1.7的扩容，链表的部分会造成节点顺序反转，因为扩容过程中采用的是前插法，数组下标位置的节点为链表头结点，扩容必须是从头结点开始转移节点，插入到新table里面是采用前插法导致了链表节点的顺序反转。多线程情况下会导致循环链表死循环CPU100%的情况。

jkd1.8的扩容，放弃了扩容过程中的前插法，对每个节点进行计算（hash & bit) 只能为0或1，bit为原数组的长度，这样为了将节点先分为lo和hi两个链表中，再对两个链表判断，将数量大于8转换成红黑树，将数量小于6转换成链表。同时维护了节点之间的顺序。扩容把原始容量扩大到了原来的两倍，扩容过程中对原节点的转移也分成了高低两个部分。



TreeSet

排序 去重 





PriorityQueue

堆 排序

# BlockingQueue

* ArrayBlockingQueue
* LinkedBlockingQueue
* PriorityBlockingQueue
* LinkedTransferQueue
* DelayQueue
* DelayedWordQueue(线程池ThreadPoolExecutor定义)



**问题**

1. 队列的数据结构？
2. 阻塞队列有哪些特性？
3. 怎么实现多线程的添加元素和获取元素的？怎么实现阻塞队列的特性的 ？即等待获取等待添加。

**接口方法**

```java
public interface BlockingQueue<E> extends Queue<E> {
    //添加到队列尾，如果成功返回true,如果失败（队列满了）将抛出{@code IllegalStateException}
    boolean add(E e);
	//添加元素到队尾，如果成功返回true,如果添加失败则返回false,队列满不会抛出异常，线程池用的是这个方法
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

# ArrayBlockedQueue

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

**入队操作**

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

**出队操作**

> put无限期等待，知道有效空间出现进行插入到队尾

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length) 
            notFull.await(); //若队列已满，则等待有效空间出现,释放锁，同时挂起线程
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
            nanos = notFull.awaitNanos(nanos); // 等待nonos时间，同时释放锁挂起线程
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

**总结**

1. 队列的数据结构？

   ArrayBlockedQueue和LinkedBlockedQueue的数据结构不同，前者使用数组后者使用单向链表。

2. 阻塞队列有哪些特性？

   若队列为空，可以阻塞获取元素的线程等到队列的元素有为止。

   若队列已满，可以阻塞添加元素的线程等到队列的空间有效为止。

3. 怎么实现多线程的添加元素和获取元素的？怎么实现阻塞队列的特性的 ？即等待获取等待添加。

   所分析的ArrayBlockedQueue中，使用了一把锁和锁的两个条件变量（Condition)，使用锁来进行多线程下的队列的出队入队的同步操作。使用两个条件变量来进行出队入队的同步。**可以使用一个条件变量行吗？思考**

   使用条件变量notFull和notEmpty来进行多线程间的通信协作。

   若队列为空，notEmpty.await()方法会使获取元素的线程挂起并释放队列的锁，等待添加元素的线程会调用notEmpty.signal()方法将其唤醒。

   若队列已满，notFull.await()方法会使添加元素的线程挂起并释放当前持有的锁，之后获取元素的线程会调用notFull.signal()方法将其唤醒。

   

**线程池中应用的几种阻塞队列** 

   LinkedBlockingQueue

   DelayedWorkQueue

   SynchronousQueue



# HashMap

> 如果容量sc小于0，表示有线程在初始化或者扩容。若sc大于0,则并发修改sizeCtl的值为-1(使用cas操作,高效),防止后面线程进来；扩容；还原sizeCtl。

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0) //sizeCtl即将扩容的大小
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);//阈值 0.75*n
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```



**HashMap不加锁，高并发会发生什么？**

> 假设数组下标0key值为null，a 、b两个线程同时插入到下标0。同时判断都为null，都进入if内部，都插入到下标0， 后一个线程b会覆盖前一个线程a。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)//这里高并发，可能会覆盖同一下标数组的值
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

> 扩容调用resize()的时候，可能会产生循环链表，造成死循环





# ConcurrentHashMap

> 将hashCode无符号右移，即低位等于高位和低位异或运算，减少高位相同低位不同的hashCode导致hash碰撞（因为使用的是低位，减少hash碰撞）

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```



**ConcurrentHashMap**怎么做到的同步？



```java
public V putIfAbsent(K key, V value) {
    return putVal(key, value, true);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,//unsafe类原子操作
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                //类似双重检查 ，获得锁之后判断，防止扩容的变化
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {//插入链表的情况
                        binCount = 1;//记录链表节点数量，若大于8，链表会转换成红黑树
                        for (Node<K,V> e = f;; ++binCount) {//遍历链表
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;//key存在（即对应的value存在），替换/不替换
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {//key不存在，新增
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {//插入红黑树情况
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;//putTreeVal查找key,存在->返回p/不存->在新增返回p
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)//若链表数量大于8，转换成红黑树
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

### ConcurrentHashMap应用场景？

> 资源隔离中的线程池隔离，存储线程资源池，为每种请求类型创建一个线程池



```JAVA
// 在线用户管理类
public class UserManager {
    private Map<String, User> userMap = new ConcurrentHashMap<>();
    
    // 当用户登入时调用
    public void onUserSignIn(String sessionId, User user) {
        this.userMap.put(sessionId, user);
    }
    
    // 当用户登出或超时时调用
    public void onUserSignOut(String sessionId) {
        this.userMap.remove(sessionId);
    }
    
    public getUser(String sessionId) {
        return this.userMap.get(sessionId);
    }
}
```



## equals()和hashCode()

### 1.为什么要重写hashCode()？

存在根据对象属性去判断两个对象是否相等，则需要重写equals()方法，根据原则，也需要重写hashCode()方法。

###  2.在什么场景下需要重写

> java源码中对hashCode特性的论述如下

* hashCode()是根据对象的属性来重写的，如果两次调用hashCode()时该对象的属性(参与计算hashCode的属性）都没变过，则要求返回的整型是相同的hashCode整数。如果属性已被修改，则返回的hashCode整数可以不同。

* equals()相等，则hashCode()一定相等。
* equals不相等，不要求hashCode不相等，但是尽量做到不相等，减少hash碰撞。

### 怎么重写hashCode和equals()?

#### 重写hashCode

> * 参与计算的项：
>
>   * 乘子：31 
>
>     * 选31作为乘子的原因：
>       - 31是质数，是作为hashCode乘子的优选质数之一，根据多个质数统计得出。
>       - 31可以被jvm优化，**31*i = (5 << i)  - i**,移位运算更快
>
>   * 对象的属性：
>
>     * byte,boolean,short,char,int,float,double,long转换成int之后参与计算
>     * 对象中引用的对象，调用改引用对象的hashCode()方法参与计算
>
>     JDK中String类重写hashCode()
>
> * 计算公式：
>
>   * result = c //c是对象属性转化成的整型值
>   * result = 31*result+c//循环计算
>
> * 下面是String类的hashCode()

```java
 public int hashCode() {
     int h = hash;
     if (h == 0 && value.length > 0) {
         char val[] = value;

         for (int i = 0; i < value.length; i++) {
             h = 31 * h + val[i];
         }
         hash = h;
     }
     return h;
 }
```

> * 业务场景中重写hashCode()

```java
/** * hours */
private int hour;
/** * minutes */
private int minute;
/** * seconds */
private final int second;
@Override
public int hashCode() {
    int result = hour;
    result = 31 * result + minute;
    result = 31 * result + second;
    return result;
}
```

#### 重写equals()



### 减少hash碰撞的方法







LinkedHashMap

[a,a,b,b,b,c,d,e,a]找出出现次数最多且排在最前的



# TreeMap

### 问题

1. TreeMap是什么？特性？
2. TreeMap使用的是何种数据结构来进行排序的，红黑树的原理？红黑树的时间复杂度？
3. TreeMap怎么实现的排序（升序和降序）？
4. 迭代key和value怎么保证了顺序迭代的？即怎么重写Iterator的 ？

# 迭代器



## Iterable和Iterator的区别

Iterable表明某个类具有使用Iterator的能力，Iterator提供真正的遍历操作

### Iterable

Iterable内部定义了获取Iterator的方法。参考ServiceLoader，其实现了Iterable

```java
/**
 * Implementing this interface allows an object to be the target of
 * the "for-each loop" statement. See
 * <strong>
 * <a href="{@docRoot}/../technotes/guides/language/foreach.html">For-each Loop</a>
 * </strong>
 *
 * @param <T> the type of elements returned by the iterator
 *
 * @since 1.5
 * @jls 14.14.2 The enhanced for statement
 */
public interface Iterable<T> {
    /**
     * Returns an iterator over elements of type {@code T}.
     *
     * @return an Iterator.
     */
    Iterator<T> iterator();
    //...
}
```

### Iterator

```java
/**
 * An iterator over a collection.  {@code Iterator} takes the place of
 * {@link Enumeration} in the Java Collections Framework.  Iterators
 * differ from enumerations in two ways:
 *
 * <ul>
 *      <li> Iterators allow the caller to remove elements from the
 *           underlying collection during the iteration with well-defined
 *           semantics.
 *      <li> Method names have been improved.
 * </ul>
 *
 * <p>This interface is a member of the
 * <a href="{@docRoot}/../technotes/guides/collections/index.html">
 * Java Collections Framework</a>.
 *
 * @param <E> the type of elements returned by this iterator
 *
 * @author  Josh Bloch
 * @see Collection
 * @see ListIterator
 * @see Iterable
 * @since 1.2
 */
public interface Iterator<E> {
    /**
     * Returns {@code true} if the iteration has more elements.
     * (In other words, returns {@code true} if {@link #next} would
     * return an element rather than throwing an exception.)
     *
     * @return {@code true} if the iteration has more elements
     */
    boolean hasNext();
    /**
     * Returns the next element in the iteration.
     *
     * @return the next element in the iteration
     * @throws NoSuchElementException if the iteration has no more elements
     */
    E next();
    /**
     * Removes from the underlying collection the last element returned
     * by this iterator (optional operation).  This method can be called
     * only once per call to {@link #next}.  The behavior of an iterator
     * is unspecified if the underlying collection is modified while the
     * iteration is in progress in any way other than by calling this
     * method.
     *
     * @implSpec
     * The default implementation throws an instance of
     * {@link UnsupportedOperationException} and performs no other action.
     *
     * @throws UnsupportedOperationException if the {@code remove}
     *         operation is not supported by this iterator
     *
     * @throws IllegalStateException if the {@code next} method has not
     *         yet been called, or the {@code remove} method has already
     *         been called after the last call to the {@code next}
     *         method
     */
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
}
```



## ConcurrentModificationException

```java
public static void main(String[] args) {
    ArrayList<String> aList = new ArrayList<String>();
    aList.add("Apple");
    aList.add("Mango");
    aList.add("Guava");
    aList.add("Orange");
    aList.add("Peach");
    Iterator i = aList.iterator();
    String str = "";
    while (i.hasNext()) {
        str = (String) i.next();
        if (str.equals("Guava")) {
            //i.remove(); //这是正确的迭代移除方法
            aList.remove(str);//很可能会引起 ConcurrentModificationException
            System.out.println("\nThe element Orange is removed");
        }
    }
}
```

Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList\$Itr.checkForComodification(ArrayList.java:909)
	at java.util.ArrayList\$Itr.next(ArrayList.java:859)
	at com.github.chuangkel.java_learn.base.collection.IteratorTest.main(IteratorTest.java:27)





# LinkedHashMap

1. LinkedHashMap是什么？特性？
2. 元素的put的顺序是怎么保证的？
3. LinkedHashMap的使用场景？





## 实用方法

java.util.HashMap#getOrDefault(Object key, V defaultValue)

java.util.Collections#reverse(List<?> list)