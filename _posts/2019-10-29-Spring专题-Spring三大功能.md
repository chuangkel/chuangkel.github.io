---
layout:     post
title:	Spring专题
subtitle: 	Spring三大功能 
date:       2019-08-29
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 设计模式
---

# Spring三大功能 

一、Spring三大核心功能

1、IOC(DI)  IOC:Inversion of Control,DI:Dependency Injection

2、AOP:Aspect Oriented Programming

3、声明式事务

二、Spring框架的主要模块

1、Data Access/Integration : spring 封装数据访问层相关内容

（1） JDBC : Spring 对 JDBC 封装后的代码.
（2） ORM: 封装了持久层框架的代码.例如 Hibernate
（3） transactions:对应 spring-tx.jar,声明式事务使用

2、WEB: spring 完成 web 相关功能.

当tomcat 加载spring 配置文件时需要有spring-web包

3、AOP: 实现 AOP 功能需要依赖

4、Aspects:  切面 AOP 依赖的包

5、Core Container:核心容器.Spring 启动最基本的条件

（1）Beans : Spring 负责创建类对象并管理对象
（2）Core: 核心类
（3）Context: 上下文参数.获取外部资源（.properties文件）或管理注解等
（4）SpEl: expression.jar

6、Test: spring 提供测试功能

三、IOC

1、IOC概念：

（1）控制反转，依赖注入

（1.1）控制反转：控制指控制类的对象，反转是指将对象交给spring管理

（1.2）依赖注入：new方式 实例化对象的过程，不需要我们进行操作，由spring进行管理 

四、AOP面向切面编程

1、概念：

正常程序的执行流程是纵向流程，面向切面就是在纵向流程中的某个或某些方法添加横向切面

2、面向切面常用场景

非业务场景，如：日志、事务、安全等，这些代码往往都是重复的，辅助系统正常流程的代码，这一类功能通过切面概念，从正常流程中区分开来，进行独立处理。

业务需求：完全站在客户角度出发，对客户实际使用系统完全没影响的辅助性功能

非业务需求：站在开发人员角度出发，对系统正常运行流程起辅助作用的功能（可以理解为，你说的他听不懂的功能都是非业务需求，例如日志，异常，人家客户懒得和你讨论这玩意，说了他也不懂）

五、声明式事务

1、事务分两种，分别是编程式事务、声明式事务

2、编程式事务和声明式事务的主要区别是：编程式事务需要程序员自己处理，声明式事务有spring进行管理，不需要人为干预

3、编程式事务需要程序员人工显示调用beginTransaction()、commit()、rollback()等事务管理相关方法

4、声明式事务，只要添加了注解，或者在配置文件配置好相关内容，事务的管理就有spring来完成