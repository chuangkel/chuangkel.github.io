---
layout:     post
title:	Spring专题
subtitle: 	Bean的生命周期
date:       2019-08-24
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spring
---

# Bean的生命周期

```
bean的生命周期？

ioc依赖反转原理？(spring如何管理bean的，ioc需要理解的点列一下)
怎么解决循环依赖的问题的

aop原理？（面向切面的编程实例，比如@Transactional 是怎么实现aop的？ java反射和动态代理机制）
利用（java动态代理和cglib）生成代理类，在目标方法调用的前后加入需要加入的代码逻辑，比如开始事务，事务提交或回滚。
spring中的设计模式？（工厂模式，适配器模式，阅读对应代码画出uml图）

组件扫描（如何找到一个组件并将其添加到容器）
bean的实例化（何时初始化，如何初始化，如何依赖注入）
钩子函数（常见的钩子函数及其作用）
面向切面编程的原理（何时、如何为@Transactional创建代理类）
如何实现自动配置、如何去xml
为什么在启动类上加@EnableTransactionManagement注解就能让系统支持事务能力
```

### 静态注入

* xml注入

* @Autowired

* @PostConstructed



```java
/**
 * Interface to be implemented by beans that need to react once all their properties
 * have been set by a {@link BeanFactory}: e.g. to perform custom initialization,
 * or merely to check that all mandatory properties have been set.
 *
 * <p>An alternative to implementing {@code InitializingBean} is specifying a custom
 * init method, for example in an XML bean definition. For a list of all bean
 * lifecycle methods, see the {@link BeanFactory BeanFactory javadocs}.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @see DisposableBean
 * @see org.springframework.beans.factory.config.BeanDefinition#getPropertyValues()
 * @see org.springframework.beans.factory.support.AbstractBeanDefinition#getInitMethodName()
 */
public interface InitializingBean {

   /**
    * Invoked by the containing {@code BeanFactory} after it has set all bean properties
    * and satisfied {@link BeanFactoryAware}, {@code ApplicationContextAware} etc.
    * <p>This method allows the bean instance to perform validation of its overall
    * configuration and final initialization when all bean properties have been set.
    * @throws Exception in the event of misconfiguration (such as failure to set an
    * essential property) or if initialization fails for any other reason
    */
   void afterPropertiesSet() throws Exception;

}
```

