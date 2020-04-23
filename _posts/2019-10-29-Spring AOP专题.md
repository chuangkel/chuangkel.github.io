---
layout:     post
title:	Spring专题
subtitle: 	Spring AOP
date:       2019-10-29
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 设计模式
---

# Spring AOP

面向切面编程，在某一点（切点PointCut）加入特定功能的代码（增强Advice）。(切面Advisor = 切点+增强)



自动创建代理 

BeanNameAutoProxyCreator,DefaultAdvisorAutoProxyCreator,AbstractAdvisorAutoProxyCreator

例子：serial :com.github.chuangkel.aop.Main

### BeanNameAutoProxyCreator

```java
<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
    <property name="beanNames">
        <list>
            <value>AAA</value>
            <value>BBB</value>
            <value>tacRollbackFacade</value>
        </list>
    </property>
    <property name="interceptorNames">
        <list>
            <value>facadeInterceptor</value>
        </list>
    </property>
</bean>
```

上面代码和下面代码是一样的



```java
@org.springframework.context.annotation.Configuration
public class Configuration {

    @Bean
    public Person person(){
        return new Person();
    }
    @Bean
    public MethodInterceptor methodInterceptor(){
        return new MethodInterceptor();
    }
    @Bean
    public BeanNameAutoProxyCreator beanNameAutoProxyCreator(){
        BeanNameAutoProxyCreator beanNameAutoProxyCreator = new BeanNameAutoProxyCreator();
        beanNameAutoProxyCreator.setBeanNames("per*");
        beanNameAutoProxyCreator.setInterceptorNames("methodInterceptor");
        return beanNameAutoProxyCreator;
    }
}
```



对代理Bean的每个方法都做了切面处理，也可以做一下其他的复杂处理，比如获取当前系统日期，存储交易日期，进行入参的校验，初始化操作，日期初始化等操作

```java
public class MethodInterceptor implements org.aopalliance.intercept.MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        System.out.println("befor..");
        Object o = methodInvocation.proceed();
        System.out.println("after..");
        return o;
    }
}
```

**具体实现代理的源码请仔细看**

# 事务传播机制



### PROPAGATION_REQUIRED

> 被注解的方法如果已经处于一个事务中，则加入到该事务中；如果没有处在任何事务中，则新启一个事务。如果被注解的方法发生异常，则加入到对应的方法的事务也会回滚。是spring默认的事务传播机制。





### PROPAGATION_REQUIRES_NEW

> 表示当前方法必须运行在它自己的事务中。一个新的事务将启动，而且如果有一个现有的事务在运行的话，则这个方法将在运行期被挂起，直到新的事务提交或者回滚才恢复执行。





### PROPAGATION_NESTED

> 表示如果当前方法正有一个事务在运行中，则该方法应该运行在一个嵌套事务中，被嵌套的事务可以独立于被封装的事务中进行提交或者回滚。如果封装事务存在，并且外层事务抛出异常回滚，那么内层事务必须回滚，反之，内层事务并不影响外层事务。如果封装事务不存在，则同PROPAGATION_REQUIRED的一样。





### PROPAGATION_SUPPORTS

> 支持当前事务，如果当前没有事务，就以非事务方式执行



### PROPAGATION_NOT_SUPPORTED

> 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起



### PROPAGATION_MANDATORY

> 使用当前的事务，如果当前没有事务，就抛出异常



### PROPAGATION_NEVER

> 以非事务方式执行，如果当前存在事务，则抛出异常。