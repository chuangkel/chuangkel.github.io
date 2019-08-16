---
layout:     post
title:	PageHelper实现分页原理
subtitle: 	PageHelper实现分页原理
date:       2019-08-11
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---
# PageHelper实现分页原理



>  PageHelper.offsetPage(PAGE_NUM, PAGE_SIZE) （页码，每页显示的数量）。
>
> 实际是在 ThreadLocal中设置了分页参数，之后在查询执行的时候，获取当线程中的分页参数，执行查询的时候通过拦截器在sql语句中添加分页参数，之后实现分页查询，查询结束后在 finally 语句中清除ThreadLocal中的查询参数。



```java
protected static final ThreadLocal<Page> LOCAL_PAGE = new ThreadLocal<Page>();

    /**
     * 设置 Page 参数
     *
     * @param page
     */
    protected static void setLocalPage(Page page) {
        LOCAL_PAGE.set(page);
    }
```

