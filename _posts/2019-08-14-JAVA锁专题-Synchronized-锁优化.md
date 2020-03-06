---
layout:     post
title:	JAVA锁专题
subtitle: 	Synchronized-锁消除
date:       2019-08-14
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# Synchronized-锁优化

## 锁消除

**引言**

1. 什么是锁消除？
2. 为什么要锁消除？
3. 有哪些锁消除的案例

**分析**

1. 什么是锁消除？为什么要锁消除？

   因为加锁和释放锁都是具有成本的。锁消除顾名思义消除锁，对理论上不存在并发的加锁代码的锁进行锁消除，从而避免一定的系统开销。在堆上的数据如果不可能被其他线程访问到，那么则可以当做栈上的数据使用，认为它是线程私有的，自然就无需加锁。

2. 锁是怎么消除的
  

​       编译器基于逃逸分析技术进行了锁的消除。需开启系统参数：

  ```
  -server -XX:+DoEscapeAnalysis -XX:+EliminateLocks
  // +DoEscapeAnalysis 表示开启逃逸分析技术 +EliminateLocks表示锁消除
  ```

3. 逃逸分析技术是什么呢
  
  逃逸分析技术是目前比较前沿的java虚拟机的优化分析技术，它与类继承关系分析不一样，并不直接优化代码手段，而是为其他优化手段提供依据的分析技术。
  
  逃逸分析的行为就是分析对象动态作用域：当一个对象定义在方法内部，并且作为方法调用返回参数，那么该对象可能被其他方法使用，则称改对象方法逃逸了。当一个对象赋值给静态变量或者可以在其他线程中访问到的变量，那么该对象可能会被其他线程访问，则称为线程逃逸。
  
  如果能证明一个对象不会方法逃逸或者线程逃逸，那么可以对该变量做一些优化，比如：
  
  栈上分配：为什么要有栈上分配？ 栈的回收时会回收
  
  标量替换：标量指无法拆分成更小的数据类型，比如byte,boolean,short(2字节),char(2字节),int,float,long,double。聚合量指可以继续分解的数据，比如对象。如果一个对象不会被外部访问并且该对象可以分解的话，那程序执行的时候可能不创建这个对象，而是直接创建被这个方法用到的若干个成员变量来代替，优化之后直接在栈上分配和读写该对象的成员变量。
  
  同步消除：线程同步本身是一个相对耗时的过程，如果逃逸分析技术能确定一个变量不会逃逸出当前线程被其他线程访问到，即该变量的读写不存在竞争，则可以消除相应的同步措施。
  
  是否逃出变量的作用域，比如下面的方法中的sb对象，若将该对象返回，则编译器会将其作为全局变量使用，就有可能存在线程安全的问题，就可以锁sb对象发生逃逸了，因而append方法上的锁就不可以消除。
  
4. 有哪些锁消除的案例

   ```mysql
   public class StringContract {
       /**
        * 每一个sb.append都是一个同步方法。sb变量处于方法内部，每一个方法调用都会开辟一个新的对象，
        * 所以sb.append的锁可以消除
        * 注意：若sb作为返回值返回，那sb就是一个全局对象，从而存在并发问题。
        */
       private static String contract(String a,String b,String c){
           StringBuffer sb = new StringBuffer();
           sb.append(a);
           sb.append(b);
           sb.append(c);
           return sb.toString();
       }
   }
   ```

   

## 锁粗化

**引言**

1. 什么是锁粗化？为什么要锁粗化
2. 锁粗化实例

**分析**
1. 什么是锁粗化？为什么要锁粗化

   平时写代码推荐的写法是将同步块的作用范围要尽量小，这样做事为了执行同步块的时间尽可能少，若存在锁竞争，其他等待的线程也能尽快拿到锁。大部分情况下，以上原则没什么问题，但若出现一系列连续的操作都对同一个对象反复加锁和解锁，甚至加锁是出现在循环体中。比如上面的sp.append()方法（append方法实现加了关键字synchronized）,在第一个sb.append前面加锁，锁直到最后一个sb.append()方法结束。 循环体里面出现了加锁解锁，则把锁加到for循环上，这样就避免了反复的加锁和解锁带来的性能开销。 这些优化都是java虚拟机自动做的。

2. 锁粗化实例



## 偏向锁

对象头信息包含两部分，第一部分为Mark Word，其结构如下表。为了节省空间，java虚拟机根据锁标志位的不同存储不同的数据。

![1571039443563](/..\img\1571039443563.png)

偏向锁时锁标志位为01

## 轻量级锁

轻量级锁时，锁标志为00，拷贝Mark Word到线程的栈中为Displaced Mark Word，再通过CAS操作将对象头的Mark Word更新成指向Displaced Mark Word的指针。



## 自旋锁和适应性自旋

适应性自旋是jvm通过之前的轻量级锁的占用的时间来动态调节后来线程的自旋时间。



### 例子：

```java
public class VolatileB {
    public static void main(String[] args) {
        Object object = new Object();
        Thread threadA = new Thread(() -> {
            int i = 0;
            while (true) {
                synchronized (object) {
                    if (i == 5) {
                        object.notify();
                        try {
                            object.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    System.out.println(i);
                    i++;
                    i %= 9;
                }
            }
        });

        Thread threadB = new Thread(() -> {
            while (true) {
                synchronized (object) {
                    try {
                        object.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("这是B");
                    object.notify();
                }
            }
        });

        threadA.start();
        threadB.start();
    }
}
```

wait() notify()针对特定的对象，需要先获取锁监视器。