---
layout:     post
title:	Condition
subtitle: 	Condition
date:       2019-08-06
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# Condition

> condition应用
>
> 生产者消费者设计模式实现：两个线程，A线程打印A,B线程打印B,要求打印ABABABABABABABABABAB

```java
public class PrintThread implements Runnable {
    private int COUNT = 10;
    private ReentrantLock reentrantLock;
    private Condition conditionA;
    private Condition conditionB;
    private String c;
    PrintThread(ReentrantLock reentrantLock,Condition conditionA,Condition conditionB,String c){
        this.reentrantLock = reentrantLock;
        this.conditionA = conditionA;
        this.conditionB = conditionB;
        this.c = c;
    }
    @Override
    public void run() {
        try{
            reentrantLock.lock();
            for(int i = 0;i < COUNT; i++){
                System.out.print(c);
                try {
                    conditionB.signal();
                    if(i < COUNT - 1){
                        conditionA.await();
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }finally {
            reentrantLock.unlock();
        }
    }
}

public class ReentranLockTest {
    static ReentrantLock reentrantLock = new ReentrantLock();
    static Condition conditionA = reentrantLock.newCondition();
    static Condition conditionB = reentrantLock.newCondition();
    public static void main(String[] args) {
       PrintThread printThreadA = new PrintThread(reentrantLock,conditionA,conditionB,"A");
       PrintThread printThreadB = new PrintThread(reentrantLock,conditionB,conditionA,"B");
        Thread tA = new Thread(printThreadA);
        Thread tB = new Thread(printThreadB);
        tA.start();
        tB.start();
        //输出BABABABABABABABABABA或者ABABABABABABABABABAB
    }
}
```



