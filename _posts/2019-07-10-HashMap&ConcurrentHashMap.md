---
layout:     post
title:	当HashMap遇ConcurrentHashMap
subtitle: 	当HashMap遇ConcurrentHashMap
date:       2019-08-10
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# 当HashMap遇ConcurrentHashMap

> 将hashCode无符号右移，即低位等于高位和低位异或运算，减少高位相同低位不同的hashCode导致hash碰撞（因为使用的是低位，减少hash碰撞）

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```



## HashMap

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



### HashMap不加锁，高并发会发生什么？

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



## ConcurrentHashMap

### ConcurrentHashMap怎么做到的同步？



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
                if (tabAt(tab, i) == f) {//获得锁之后判断，防止扩容的变化
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
>   * 乘子：31 
>
>     * 选31作为乘子的原因：
>       - 31是质数，是作为hashCode乘子的优选质数之一，根据多个质数统计得出。
>       - 31可以被jvm优化，**31*i = (5 << i)  - i**,移位运算更快
>
>   * 对象的属性：
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