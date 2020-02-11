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