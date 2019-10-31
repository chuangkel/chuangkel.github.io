---
layout:     post
title:	Spring专题
subtitle: 	IOC系列二
date:       2019-08-29
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 设计模式
---

# IOC系列二

## 基本认识

1. spring容器是什么

   直观的说是ApplicationContext的一个实例，包含了BeanFactory的所有功能，主要用于**管理bean的整个生命周期**，同时监听？、个性化设置。

2. spring容器启动做了哪些事？
   1. 加载环境配置（jdk路径，java版本等等），加载配置文件，
   2. 实例化工厂单例bean 定位 加载 注册
   3. 启动监听



3. spring容器的两种类型：
   1. Spring BeanFactory类型，最简单的容器，用于**DI** (Dependency Injection 依赖注入)。依赖注入就是等对象创建完之后，扫描所有属性，如果发现有@Autowiere的，就获取bean设置一下。

      ![1571968229424](/..\img\1571968229424.png)

   2. ApplicationContext类型

      ApplicationContext是继承BeanFactory的，包含了BeanFactory的全部功能。

![1571967291878](/..\img\1571967291878.png)



1. 容器启动要做的几件事

   - 定位

     1 代码指明了bean的路径

   - 加载

     第 4 步进行了加载

   - 注册

     第2、3步进行了注册

```java
public static void main(String[] args) {
    ClassPathResource classPathResource = new ClassPathResource("bean.xml"); // 1 
    DefaultListableBeanFactory defaultListableBeanFactory = new DefaultListableBeanFactory(); // 2 创建容器
    XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(defaultListableBeanFactory); // 3
    xmlBeanDefinitionReader.loadBeanDefinitions(classPathResource); // 4
}
```

2. 有多个工厂？
   * 是的。工厂是单例。

## SpringApplication(容器)启动过程

### 构造器初始化时初始化

1. 存在ApplicationContextInitializer的数组，一组初始化

```java
private List<ApplicationContextInitializer<?>> initializers; //如下图
```

```java
private List<ApplicationListener<?>> listeners; //一组监听器
```

```java
/**
 该接口被事件监听器实现，基于标准的事件监听器接口是采用的观察者模式。在spring3.0里，这个ApplicationListener通常被是对事件类型关联的。当spring的ApplicationContext上下文注册监听器的时候，事件对应的监听器被调用。
 */
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
   /**
    * 处理 application event.
    */
   void onApplicationEvent(E event);
}
```



### SpringApplicationRunListener

```java
// SpringApplicationRunListener的集合
class SpringApplicationRunListeners {
    //..
}
//SpringApplicationRunListeners listeners = getRunListeners(args); 
```



## AbstractApplicationContext

### refresh方法

加载和刷新持久化的配置（配置可能是XML文件，属性文件，关联数据库表）。refresh作为一个启动方法，在失败的时候应该销毁已经创建的单例，避免资源的占用。换句话说，在调用该方法之后，要么单例全部创建，要么没有单例被创建。

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
     // 1. 准备上下文刷新。
      prepareRefresh();
     // 2. 告诉子类刷新内部bean工厂。
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
      //3. 准备bean工厂在上下文使用。
      prepareBeanFactory(beanFactory);

      try {
      	 // 4. 允许上下文子类对bean工厂的后置处理器处理。
         postProcessBeanFactory(beanFactory);
         // 5. 调用在上下文中注册的BeanFactory后置处理器。 
         invokeBeanFactoryPostProcessors(beanFactory); 
         // 6. 注册拦截创建bean的后置处理器。
         registerBeanPostProcessors(beanFactory);
         // 7. 为上下文初始化消息源。
         initMessageSource();
         // 8. 为上下文初始化事件广播器。观察者模式中的Subject角色。
         initApplicationEventMulticaster();
         // 9. 在特定的上下文子类中初始化其他特定的bean。
         onRefresh();
         // 10. 检查监听器并注册监听器。观察者模式中的Observer角色。
         registerListeners();
         // 11. 实例化所有保留的非懒加载单例。
         finishBeanFactoryInitialization(beanFactory);
         // 12. 最后一步，发布相应的事件。
         finishRefresh();
      }

      catch (BeansException ex) {
         // 销毁已经创建的单例避免资源的占用。
         destroyBeans();
         // 重置active标识
         cancelRefresh(ex);
         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // 重置spring核心的缓存
         resetCommonCaches();
      }
   }
}
```

1. **准备上下文刷新。**设置启动时间和激活标识，执行配置文件的的初始化（替换所有占位符）

2. **告诉子类刷新内部bean工厂。**告诉子类刷新内部bean工厂的方法将在其他初始化工作之前被调用，创建一个bean工厂并且持有工厂引用，或者返回单例工厂实例。如果上下文刷新超过一次，或者bean工厂初始化失败将抛出异常。初始化BeanFactory：根据配置文件实例化BeanFactory，getBeanFactory()方法由具体子类实现。在这一步里，Spring将配置文件的信息解析成为一个个的BeanDefinition对象并装入到容器的Bean定义注册表（BeanDefinitionRegistry）中，但此时Bean还未初始化；obtainFreshBeanFactory()会调用自身的refreshBeanFactory(),而refreshBeanFactory()方法由子类AbstractRefreshableApplicationContext实现，该方法返回了一个创建的DefaultListableBeanFactory对象，这个对象就是由ApplicationContext管理的BeanFactory容器对象。
3. **准备bean工厂在上下文使用。**
4. **允许上下文子类对bean工厂的后置处理器处理。**在bean工厂初始化之后，修改应用上下文内部bean工厂。所有的bean定义已经被加载，但是到现在为止没有bean被生成。这个方法允许为了注册特别的工厂后置处理器等在具体的上下文实现中。
5. **调用在上下文中注册的BeanFactory后置处理器。 **调用bean工厂后置处理器，根据反射机制从bean工厂BeanDefinitionRegistry中找出所有BeanFactoryPostProcessor类型的Bean，并调用其postProcessBeanFactory()接口方法。
6. **注册拦截创建bean的后置处理器。**
7. **为上下文初始化消息源。**
8. **为上下文初始化事件广播器。观察者模式中的Subject角色。**
9. **在特定的上下文子类中初始化其他特定的bean。**
10. **检查监听器并注册监听器。观察者模式中的Observer角色。**
11. **实例化所有保留的非懒加载单例。**
12. **最后一步，发布相应的事件。**

调用工厂后处理器：根据反射机制从BeanDefinitionRegistry中找出所有BeanFactoryPostProcessor类型的Bean，并调用其postProcessBeanFactory()接口方法。

经过第一步加载配置文件，已经把配置文件中定义的所有bean装载到BeanDefinitionRegistry这个Beanfactory中，对于ApplicationContext应用来说这个BeanDefinitionRegistry类型的BeanFactory就是Spring默认的DefaultListableBeanFactory

public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
                                                           implements ConfigurableListableBeanFactory, BeanDefinitionRegistry

在这些被装载的bean中，若有类型为BeanFactoryPostProcessor的bean（配置文件中配置的），则将对应的BeanDefinition生成BeanFactoryPostProcessor对象

容器扫描BeanDefinitionRegistry中的BeanDefinition，使用java反射自动识别出Bean工厂后处理器（实现BeanFactoryPostProcessor接口）的bean，然后调用这些bean工厂后处理器对BeanDefinitionRegistry中的BeanDefinition进行加工处理，可以完成以下两项工作(当然也可以有其他的操作，用户自己定义)：

1 对使用到占位符的<bean>元素标签进行解析，得到最终的配置值，这意味着对一些半成品式的BeanDefinition对象进行加工处理并取得成品的BeanDefinition对象。

2 对BeanDefinitionRegistry中的BeanDefinition进行扫描，通过Java反射机制找出所有属性编辑器的Bean（实现java.beans.PropertyEditor接口的Bean），并自动将它们注册到Spring容器的属性编辑器注册表中（PropertyEditorRegistry），这个Spring提供了实现：CustomEditorConfigurer，它实现了BeanFactoryPostProcessor，用它来在此注册自定义属性编辑器。



BeanFactoryPostProcessor接口代码如下，实际的操作由用户扩展并配置（**扩展点，如何扩展？**）

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
   /*BeanFactory后置处理器，钩子函数。在标准的实例化之后修改应用上下文中的内部bean工厂。所有的bean定义都被加载，但是没有beans没有被初始化。 这个方法允许对期望初始化的beans覆写或者添加属性*/
   void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

上面的装载方法：

```java
/*通过 XmlBeanDefinitionReader 加载bean定义。*/
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   // 为传入的 BeanFactory 新建一个 XmlBeanDefinitionReader。
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
   // 用上下文的资源配置bean定义阅读器。
   beanDefinitionReader.setEnvironment(this.getEnvironment());
   beanDefinitionReader.setResourceLoader(this);
   beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
   // 允许子类提供自定义的阅读器的初始化。然后处理加载bean定义。
   initBeanDefinitionReader(beanDefinitionReader);
   loadBeanDefinitions(beanDefinitionReader);
}
```