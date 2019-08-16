---
layout:     post
title:      spring注解
subtitle:   @Autowired 和 @Resource的区别
date:       2019-07-30
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spring
---


#  spring注解
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
    