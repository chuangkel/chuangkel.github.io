---
layout:     post
title:	Spring专题
subtitle: 	springMvc原理
date:       2019-08-25
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spring
---

# SpringMvc原理

> Interface to be implemented by objects that define a mapping between requests and handler objects
>
> 对象(定义请求和请求处理器之间映射关系的对象)会实现HandlerMapping接口

```java
public interface HandlerMapping {
//...
   @Nullable
   HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```

