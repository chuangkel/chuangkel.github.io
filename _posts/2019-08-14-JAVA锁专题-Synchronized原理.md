---
layout:     post
title:	JAVA锁专题
subtitle: 	Synchronized原理解析
date:       2019-08-14
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# Synchronized原理解析

> 临界区（Critical Section）、互斥量（Mutex）和信号量（Semaphore）都是主要的互斥实现方式。
>
> **Java的线程是映射到操作系统的原生线程之上的，如果要阻塞或唤醒一个线程，都需要操作系统来帮忙完成，这就需要从用户态转换到核心态中，因此状态转换需要耗费很多的处理器时间。** 对于代码简单的同步块（如被synchronized修饰的getter（）或setter（）方法），状态转换消耗的时间有可能比用户代码执行的时间还要长。



### Synchronized和Lock联系和区别

除了synchronized之外，我们还可以使用java.util.concurrent（下文称J.U.C）包中的重入锁 （ReentrantLock）来实现同步，在基本用法上，ReentrantLock与synchronized很相似，他们都 具备一样的线程重入特性，只是代码写法上有点区别，一个表现为API层面的互斥锁 （lock（）和unlock（）方法配合try/finally语句块来完成），另一个表现为原生语法层面的互斥锁。不过，相比synchronized,ReentrantLock增加了一些高级功能，主要有以下3项：等待可 中断、可实现公平锁，以及锁可以绑定多个条件。

* 等待可中断是指当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等 待，改为处理其他事情，可中断特性对处理执行时间非常长的同步块很有帮助。

* 公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁；而 非公平锁则不保证这一点，在锁被释放时，任何一个等待锁的线程都有机会获得锁。 synchronized中的锁是非公平的，ReentrantLock默认情况下也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁。 

* 锁绑定多个条件是指一个ReentrantLock对象可以同时绑定多个Condition对象，而在 synchronized中，锁对象的wait（）和notify（）或notifyAll（）方法可以实现一个隐含的条 件，如果要和多于一个的条件关联的时候，就不得不额外地添加一个锁，而ReentrantLock则 无须这样做，只需要多次调用newCondition（）方法即可。 



```shell
# 查看class执行的指令码
javac x.java
javap -c    x.class
javap -l x.class
javap -v x.class 
```

### 管程

> Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（Monitor）来支持的。方法级的同步是隐式的，即无须通过字节码指令来控制，它实现在方法调用和返回操作之中。虚拟机可以从方法常量池的方法表结构中的ACC_SYNCHRONIZED访问标志得知一个方法是否声明为同步方法。当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程，然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时释放管程。在方法执行期间，执行线程持有了管程，其他任何线程都无法再获取到同一个管程。如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那么这个同步方法所持有的管程将在异常抛到同步方法之外时自动释放。同步一段指令集序列通常是由Java语言中的synchronized语句块来表示的，Java虚拟机的指令集中有monitorenter和monitorexit两条指令来支持synchronized关键字的语义，正确实现synchronized关键字需要Javac编译器与Java虚拟机两者共同协作支持。

```
public class SynchronizedTest {
    public static void main(String[] args) {
        synchronized (SynchronizedTest.class){

        }
    }
}
```

> javap -c    x.class 结果如下： 为什么有两个monitorexit 管程退出？

```
public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // class com/github/chuangkel/java_learn/base/lock/SynchronizedTest
       2: dup
       3: astore_1
       4: monitorenter
       5: aload_1
       6: monitorexit
       7: goto          15
      10: astore_2
      11: aload_1
      12: monitorexit
      13: aload_2
      14: athrow
      15: return
```

> 编译器必须确保无论方法通过何种方式完成，方法中调用过的每条monitorenter指令都必须执行其对应的monitorexit指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时monitorenter和monitorexit指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行monitorexit指令。



### 管程和信号量

> 无论是同步方法还是同步代码块，无论是`ACC_SYNCHRONIZED`还是`monitorenter`、`monitorexit`都是基于`Monitor`实现的

#### 操作系统中的管程

> 管程 (Monitors，也称为监视器) 是一种程序结构，结构内的多个子程序（对象或模块）形成的多个工作线程互斥访问共享资源。这些共享资源一般是硬件设备或一群变量。管程实现了在一个时间点，最多只有一个线程在执行管程的某个子程序。与那些通过修改数据结构实现互斥访问的并发程序设计相比(信号量实现互斥访问？)，管程实现很大程度上简化了程序设计。 管程提供了一种机制，线程可以临时放弃互斥访问，等待某些条件得到满足后，重新获得执行权恢复它的互斥访问。



### 对象和管程（Monitor 监视器）

> 每个对象（包括class对象）都有一个监视器。监视器有三部分，比较形象的比喻，special room只能有一个线程持有，wait root是挂起线程的队列，等待队列在hallway。

![1566178299653](/../img/1566178299653.png)

> 在JAVA虚拟机中，每个对象(Object和class)通过某种逻辑关联监视器，为了实现监视器的互斥功能，每个对象(Object和class)都关联着一个监视器，对象可以有它自己的临界区，并且能够监视线程序列为了使线程协作，JAVA提供了wait()和notifyAll以及notify()实现挂起线程、唤醒另外一个等待的线程。

![1566178354439](/../img/1566178354439.png)

和下面的图一样的：

![1566195759821](/../img/1566195759821.png)

### 对象头

| 长度     | 内容                   | 说明                     |
| -------- | ---------------------- | ------------------------ |
| 32/64bit | markword               | 存储hashCode或者锁信息   |
| 32/64bit | class metadata address | 存储到对象类型数据的指针 |
| 32/64bit |                        |                          |





> markword的结构（32位虚拟机，25位存储hashCode,4位存储对象的分代年龄，2位存储锁标志位，1位固定为0，其他位对象不同状态下存储的内容有所不同）

| 存储内容               | 标志位 | 状态       |
| ---------------------- | ------ | ---------- |
| 对象HashCode，分代年龄 | 01     | 未锁定     |
| 指向锁记录的指针       | 00     | 轻量级锁定 |
| 指向重量级锁指针       | 10     | 重量级锁定 |
|                        | 11     | GC         |
| 偏向线程id,偏向时间戳  | 01     | 可偏向     |




锁的升级

偏向锁 -> 轻量级锁->偏向锁->重量级锁

![img](/../img/偏向锁的撤销.png)





![img](/../img/轻量级锁.png)