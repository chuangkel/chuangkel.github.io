---
layout:     post
title:	java反射和代理
subtitle: 	java反射和代理
date:       2019-08-11
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# java反射和代理





#### 注解在什么情况下失效？

* 同一个类中方法上的注解，注解在methodB()上，methodA()调用methodB()时，实际使用的原对象.methodB()而不是代理对象.methodB()
* 注解在private方法或属性上，子类无法继承该方法，导致注解失效。



