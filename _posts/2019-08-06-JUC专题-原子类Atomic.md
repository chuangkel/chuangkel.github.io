---
layout:     post
title:	JUC专题
subtitle: 	原子类Atomic
date:       2019-08-06
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# 原子类Atomic

AtomicInteger、AtomicLong、AtomicBoolean、AtomicRefrence

```java
public final int decrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
}
```



unsafe.class

自旋直到更新成功为止

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```