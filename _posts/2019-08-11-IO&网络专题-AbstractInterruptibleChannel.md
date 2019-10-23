---
layout:     post
title:	IO&网络专题
subtitle: 	AbstractInterruptibleChannel
date:       2019-08-08
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# AbstractInterruptibleChannel



```java
public int read(ByteBuffer dst) throws IOException {
    //...
    try {
        begin(); //为什么调用了implCloseChannel（）
        bytesRead = in.read(buf, 0, bytesToRead);
    } finally {
        end(bytesRead > 0);
    }
    //...
}
```



```java
protected final void begin() {
    if (interruptor == null) {
        interruptor = new Interruptible() {
                public void interrupt(Thread target) {
                    synchronized (closeLock) {
                        if (!open)
                            return;
                        open = false;
                        interrupted = target;
                        try {
                            AbstractInterruptibleChannel.this.implCloseChannel();
                        } catch (IOException x) { }
                    }
                }};
    }
    blockedOn(interruptor);//
    Thread me = Thread.currentThread();
    if (me.isInterrupted())
        interruptor.interrupt(me); //如果被中断 ，则停止线程
}
```





```java
protected final void end(boolean completed)
    throws AsynchronousCloseException
{
    blockedOn(null);
    Thread interrupted = this.interrupted;
    if (interrupted != null && interrupted == Thread.currentThread()) {
        interrupted = null;
        throw new ClosedByInterruptException();
    }
    if (!completed && !open)
        throw new AsynchronousCloseException();
}
```





### NIO

Channel（通道），Buffer（缓冲区），Selector（选择器）