---
layout:     post
title:	Synchronized原理解析
subtitle: 	Synchronized原理解析
date:       2019-08-14
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Synchronized原理解析
---

Synchronized原理解析





### 对象头

> markword的结构（32位虚拟机，25位存储hashCode,4位存储对象的分代年龄，2位存储锁标志位，1位固定为0，其他位对象不同状态下存储的内容有所不同）

| 存储内容               | 标志位 | 状态       |
| ---------------------- | ------ | ---------- |
| 对象HashCode，分代年龄 | 01     | 未锁定     |
| 指向锁记录的指针       | 00     | 轻量级锁定 |
| 指向重量级锁指针       | 10     | 重量级锁定 |
|                        | 11     | GC         |
| 偏向线程id,偏向时间戳  | 01     | 可偏向     |



```shell
# 查看class执行的指令码
javac x.java
javap -c    x.class
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

