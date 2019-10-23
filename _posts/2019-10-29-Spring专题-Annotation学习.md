---
layout:     post
title:      Spring专题
subtitle:   注解原理
date:       2019-10-29
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spring
---

# 注解原理

注解是怎么起作用的？

下面是jdk文档的一段论述：

```
An annotation type declaration specifies a new annotation type, 
a special kind of interface type. To distinguish an annotation 
type declaration from a normal interface declaration, the keyword 
interface is preceded by an at-sign (@).
...
The direct superinterface of every annotation type is java.lang.annotation.Annotation.
```

```java
/**
 * The common interface extended by all annotation types.  Note that an
 * interface that manually extends this one does <i>not</i> define
 * an annotation type.  Also note that this interface does not itself
 * define an annotation type.
 *
 * More information about annotation types can be found in section 9.6 of
 * <cite>The Java&trade; Language Specification</cite>.
 *
 * The {@link java.lang.reflect.AnnotatedElement} interface discusses
 * compatibility concerns when evolving an annotation type from being
 * non-repeatable to being repeatable.
 *
 * @author  Josh Bloch
 * @since   1.5
 */
public interface Annotation {
    //...
}
```

## spring注解

* @Autowired 和 @Resource的区别
    * 共同点：都可以作用于字段和方法
    * 不同点：
        * Autowired只通过Type注入,@Resource可以Name注入和Type注入。
    * @Resource注入顺序
        * Type和Name 都指定的情况：会按照type和id查找，找不到抛异常。
        * 只指定了Type,在上下文中查找唯一的bean,找到多个或没找到报异常。
        * 只指定了Name,在上下文中通过id查找注入，没找到报异常。
     * @Qualifier是对@Autowired的补充，使其可以通过Name来注入
    
     
    
* 组件
    * Controller
    * Service
    * Repository 标注数据访问组件，dao层
    * Component 上面三个都是相当于继承Component
    