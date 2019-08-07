---
layout:     post
title:	当HashMap遇ConcurrentHashMap
subtitle: 	当HashMap遇ConcurrentHashMap
date:       2019-08-10
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java基础
---

# 当HashMap遇ConcurrentHashMap

> 将hashCode无符号右移，即低位等于高位和低位异或运算，防止高位相同低位不同的hashCode导致hash碰撞（因为使用的是低位，减少hash碰撞）

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

