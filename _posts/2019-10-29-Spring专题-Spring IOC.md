---
layout:     post
title:	Spring专题
subtitle: 	Spring IOC
date:       2019-12-25
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spring
---

# Spring IOC 

### IOC概论

控制反转和依赖注入，控制反转主要是对创建对象的控制权交个容器处理，对象有单例模式和原型模式（prototype)，在web的容器中（也是继承ApplicationContext)，Bean的作用于还有requst，session，application。主要用于管理bean的整个生命周期。

为什么要有观察者模式呢 ？为什么不直触发事件呢？ （比如dubbo暴露服务到注册中心，在回调事件里面刷新缓存。个人觉得是为了扩展性，为了程序能实现自己特定功能，在系统启动的时候）



spring容器启动做了哪些事？
1. 加载环境配置（jdk路径，java版本等等），加载配置文件，
2. 实例化工厂单例bean 定位 加载 注册
3. 启动监听



spring容器的两种类型：
1. Spring BeanFactory类型，最简单的容器，用于DI (Dependency Injection 依赖注入)。依赖注入就是等对象创建完之后，扫描所有属性，如果发现有@Autowiere(自动注入的一种)的，就获取bean设置一下。注入方式有setter方法注入，构造器注入，自动注入【@Autowired(Class类型注入),@Resource(通过类型，名字（id)自动注入）】

   ![1571968229424](/..\img\1571968229424.png)

2. ApplicationContext类型

   ApplicationContext是继承BeanFactory的，包含了BeanFactory的全部功能。

![1571967291878](/..\img\1571967291878.png)





### 容器启动过程

 IOC容器启动过程中的refresh方法

加载和刷新持久化的配置（配置可能是XML文件（bean定义等），属性文件，关联数据库表）。refresh作为一个启动方法，在失败的时候应该销毁已经创建的单例，避免资源的占用。换句话说，在调用该方法之后，要么单例全部创建，要么没有单例被创建。

1. 创建容器，也就是BeanFactory实现类

2. 准备容器。设置类加载器和后置处理器。加载BeanDefinition，并且注入到DefaultListableBeanFactory中

3. 实例化并且调用所有的BeanFactoryPostProcessor(BeanDefinition加载之后，单例实例化之前)，postProcessorBeanFactory方法主要是做了初始化工厂，添加一些特别的后置处理器，比如BeanDefinitionRegistryPostProcessor接口的实现类（功能是在BeanDefinition完全加载之后再对Bean进行修改） 。

   顺序是先调用BeanDefinitionRegistry接口（也实现了BeanFactoryPostProcessor）的回调（有实现@Order @PriorityOrdered，进行排序再回调） ，然后才是BeanFactoryPostProcessor的回调方法postProcessorBeanFactory方法。

4. 
   1. 初始化应用实践多路广播器
   2. 注册应用监听
   3. 触发广播器，回调监听

5. 

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    @Override
    public void refresh() throws BeansException, IllegalStateException {
       synchronized (this.startupShutdownMonitor) {
         // 1. 准备上下文刷新。
          prepareRefresh();
         // 2. 调用子类方法创建BeanFactory，并刷新它。
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
    //...
}
```

1. **准备上下文刷新。**设置启动时间和激活标识，执行配置文件的的初始化（替换所有占位符）

2. **调用子类方法创建BeanFactory，并刷新它。**

   调用子类方法创建一个BeanFactory并且清除它内部的信息。如果上下文刷新超过一次，或者bean工厂初始化失败将抛出异常。初始化BeanFactory：根据配置文件实例化BeanFactory，getBeanFactory()方法由具体子类实现。在这一步里，Spring将配置文件的信息解析成为一个个的BeanDefinition对象并装入到容器的Bean定义注册表（BeanDefinitionRegistry）中，但此时Bean还未初始化；obtainFreshBeanFactory()会调用自身的refreshBeanFactory(),而refreshBeanFactory()方法由子类AbstractRefreshableApplicationContext实现，该方法返回了一个创建的DefaultListableBeanFactory对象，这个对象就是由ApplicationContext管理的BeanFactory容器对象。

   ```java
   public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
       protected DefaultListableBeanFactory createBeanFactory() {
          return new DefaultListableBeanFactory(getInternalParentBeanFactory());
       }
   }
   ```

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

该接口被事件监听器实现，基于标准的事件监听器接口是采用的观察者模式。在spring3.0里，这个ApplicationListener通常被是对事件类型关联的。当spring的ApplicationContext上下文注册监听器的时候，事件对应的监听器被调用。

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
   /**
    * 处理 application event.
    */
   void onApplicationEvent(E event);
}
```



### Bean的属性

id

name

class

scop 作用域singleton，prototype原型模式，request，session

atowire bean的自动装配模式，即依赖注入方式，no不自动装配，byName通过name属性自动装配，byType通过class类全名自动装配，constructor通过构造器自动装配

init-method 

destroy-method

factory-method 静态工厂方法

factory-bean 



#### Bean的生命周期



实例化-> 初始化-> 销毁

![image-20200104170434081](/..\img\bean_life.png)



实例化

BeanPostProcessor

* postProcessBeforeInstantiation
* postProcessAfterInstantiation

InstantiationAwareBeanPostProcessor

* postProcessBeforeInstantiation

初始化

InitializingBean

* afterPropertiesSet()

销毁

DisposableBean

* destroy

ObjectProvider （懒汉）和 FactoryBean（非懒汉） 的区别





org.springframework.beans.factory.config.BeanFactoryPostProcessor#postProcessBeanFactory



```java
<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
    <property name="beanNames">
        <list>
            <value>AAA</value>
            <value>BBB</value>
        </list>
    </property>
    <property name="interceptorNames">
        <list>
            <value>facadeInterceptor</value>
        </list>
    </property>
</bean>
```



Bean的生命周期 ，打印方法调用过程

```java
com.github.chuangkel.beanCyclieLife.MyBeanFactoryPostProcessor.postProcessBeanFactory
19:27:25.121 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'myBeanPostProcessor'
19:27:25.122 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'myInstantiationAwareBeanPostProcessor'
19:27:25.129 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'teacher'
com.github.chuangkel.beanCyclieLife.MyInstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation
com.github.chuangkel.beanCyclieLife.MyInstantiationAwareBeanPostProcessor.postProcessAfterInstantiation
com.github.chuangkel.beanCyclieLife.MyInstantiationAwareBeanPostProcessor.postProcessProperties
com.github.chuangkel.beanCyclieLife.Teacher.setBeanName
com.github.chuangkel.beanCyclieLife.Teacher.setBeanClassLoader
com.github.chuangkel.beanCyclieLife.Teacher.setBeanFactory
com.github.chuangkel.beanCyclieLife.MyBeanPostProcessor.postProcessBeforeInitialization
com.github.chuangkel.beanCyclieLife.MyInstantiationAwareBeanPostProcessor.postProcessBeforeInitialization
com.github.chuangkel.beanCyclieLife.Teacher.afterPropertiesSet
com.github.chuangkel.beanCyclieLife.Teacher.myInit
com.github.chuangkel.beanCyclieLife.MyBeanPostProcessor.postProcessAfterInitialization
com.github.chuangkel.beanCyclieLife.MyInstantiationAwareBeanPostProcessor.postProcessAfterInitialization
19:27:25.237 [SpringContextShutdownHook] DEBUG org.springframework.context.support.ClassPathXmlApplicationContext - Closing org.springframework.context.support.ClassPathXmlApplicationContext@2a3046da, started on Tue Feb 11 19:27:24 GMT+08:00 2020
com.github.chuangkel.beanCyclieLife.Teacher.destroy
com.github.chuangkel.beanCyclieLife.Teacher.myDestroy
```

#### Bean生命周期走势图

![1581426228359](/..\img\1581426228359.png)

![1581426712180](/..\img\1581426712180.png)

#### Bean生命周期中方法的级别

* Bean自身的方法：Bean本身的方法和配置文件中<bean>中init-method和destroy-method方法

* 容器级别：BeanPostProcessor和InstantiationAwareBeanPostProcessor的接口方法 后置处理器，在容器加载完配置文件之后立即调用
  * 工厂后处理器方法：AspectJWeavingEnabler, ConfigurationClassPostProcessor, CustomAutowireConfigurer 

* Bean级别：BeanFactoryAware,BeanNameAware,BeanClassLoaderAware,InitializingBean,DisposableBean这几个接口的方法



#### Bean的作用域

```java
public interface WebApplicationContext extends ApplicationContext {
   String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";
   String SCOPE_REQUEST = "request";
   String SCOPE_SESSION = "session";
   String SCOPE_GLOBAL_SESSION = "globalSession";
   String SCOPE_APPLICATION = "application";
   String SERVLET_CONTEXT_BEAN_NAME = "servletContext";
}	
```



### 三种bean的定义

|          | 基于XML配置                                         | 基于注解配置                                                 | 基于Java类配置                                               |
| -------- | --------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Bean定义 | 在XML文件中定义<bean class="com.github.xxx"></bean> | 在类上添加注解@Component及衍生类（@Repository@Service、@Controller）定义Bean | 在标注了@Configuration的类中，通过类方法上@Bean定义一个Bean，方法必须提供Bean的实例化逻辑 |

1. 第一种：基于XML配置

   ```java
   /*
   <bean id="person" class="com.github.chuangkel.springdemo.Model.Person">
       <property name="name" value="Tom"/>
   </bean>
   */
   ClassPathResource classPathResource = new ClassPathResource("bean.xml");
           DefaultListableBeanFactory defaultListableBeanFactory = new DefaultListableBeanFactory();
           XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(defaultListableBeanFactory);
           xmlBeanDefinitionReader.loadBeanDefinitions(classPathResource);
           Person person = (Person) defaultListableBeanFactory.getBean("person");
           defaultListableBeanFactory.addBeanPostProcessor(person);
   ```

2. 第二种：基于注解配置

   ```java
   @Autowired
   private Person personAutowired;
   ```

3. 第三种：基于java类配置

   ```java
   @Configuration
   public class MyConfig {
       @Bean
       public Person getPerson(){
           return new Person();
       }
   }
   ```
   
   ```java
   //获取bean
   AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MyConfig.class);
   Person person1 = (Person) ctx.getBean("getPerson");	
   ```
   
   



#### 分析

先看下AnnotationConfigApplicationContext的UML类图继承关系：

![1571974164861](/..\img\1571974164861.png)

先不急，分析一下AnnotationConfigApplicationContext的直接父类，实例化了DefaultListableBeanFactory工厂。DefaultListableBeanFactory是Spring Ioc容器中非常重要的一个类，它实现了Ioc的很多功能，这里可以提供给我们的主角AnnotationConfigApplicationContext使用。

```java
public GenericApplicationContext() {
   this.beanFactory = new DefaultListableBeanFactory();
}
```

GenericApplicationContext的父类AbstractApplicationContext和父类的父类DefaultResourceLoader，都是用来加载资源的。

OK，父类看完了，我们开始分析：

1. 这是AnnotationConfigApplicationContext的构造器，看起来挺简单的

```java
public AnnotationConfigApplicationContext(String... basePackages) {
   this();
   scan(basePackages);
   refresh();
}
```

this()调用了自己的无参构造器，只有两行代码

```java
public AnnotationConfigApplicationContext() {
   this.reader = new AnnotatedBeanDefinitionReader(this);
   this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

先看无参构造器的第一行代码：实例化了一个AnnotatedBeanDefinitionReader，入参就是AnnotationConfigApplicationContext自己

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
   this(registry, getOrCreateEnvironment(registry));
}
```

getOrCreateEnvironment(registry)从工厂register中取，如果没有新建一个，属于懒加载的方式。

```java
private static Environment getOrCreateEnvironment(BeanDefinitionRegistry registry) {
   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   if (registry instanceof EnvironmentCapable) {
      return ((EnvironmentCapable) registry).getEnvironment();
   }
   return new StandardEnvironment();
}
```

接着看this(registry, getOrCreateEnvironment(registry))，同样调用了自己的构造函数

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   Assert.notNull(environment, "Environment must not be null");
   this.registry = registry;
   this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
   AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

ConditionEvaluator这个类先跳过。看最后一行，最终调到了这个方法，**这个方法是重中之重啦！！！**

**在给定的注册表registry（也是BeanFactory）中，注册所有相关联的注释后处理器。source是已经提取的配置源元素，可能为空，注册表将被配置源元素触发**

```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
      BeanDefinitionRegistry registry, Object source) {

   DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
   if (beanFactory != null) {
      if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
         beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
      }
      if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
         beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
      }
   }

   Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(8);

   if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
   if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
   if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition();
      try {
         def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
               AnnotationConfigUtils.class.getClassLoader()));
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
      }
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
   }

   if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
      RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
      def.setSource(source);
      beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
   }

   return beanDefs;
}
```

挺长的，先看第一部分：

```java
DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
if (beanFactory != null) {
   if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
      beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
   }
   if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
      beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
   }
}
```

从注册表中拿beanFacory，还记得之前的父类GenericApplicationContext的构造参数吗，实例化了 DefaultListableBeanFactory，后面是设置比较器和解析器了。

```java
Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(8);

if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
   RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
   def.setSource(source);
   beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
}
```

后面很多个if判断，代码逻辑都是一样的，只截取了第一段来分析。其实就是set BeanDefinitionHolder（是BeanDefinition）。如果registry包含了内置配置注释处理器，就新建一个相应的根Bean定义，添加到Set数据结构中。

```java
public static final String CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME =
"org.springframework.context.annotation.internalConfigurationAnnotationProcessor";public static final String CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME =
      "org.springframework.context.annotation.internalConfigurationAnnotationProcessor";
```

这个就是返回一个BeanDefinitionHolder

```java
private static BeanDefinitionHolder registerPostProcessor(
      BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

   definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   registry.registerBeanDefinition(beanName, definition);
   return new BeanDefinitionHolder(definition, beanName);
}
```

到这里，这个最长的方法我们分析完了。整个方法的后半部分是为了注册spring支持的各种注解的解析器。

Spring三大核心功能

1、IOC(DI)  IOC:Inversion of Control,DI:Dependency Injection

2、AOP:Aspect Oriented Programming

3、声明式事务

1. 容器启动要做的几件事

   - 定位

     1 代码指明了bean的路径

   - 加载

     第 4 步进行了加载

   - 注册

     第2、3步进行了注册

```java
public static void main(String[] args) {
    ClassPathResource classPathResource = new ClassPathResource("bean.xml"); //1.定位Bean路径 
    DefaultListableBeanFactory defaultListableBeanFactory = new DefaultListableBeanFactory(); //2.创建容器
    XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(defaultListableBeanFactory); //3.注册BeanDesfinition
    xmlBeanDefinitionReader.loadBeanDefinitions(classPathResource); // 4
}
```

1. 定位

   1. 有几种资源方式呢？

      UrlResource是URL路径

      ClassPathResource两种是一种非URL路径或者"classpath:" pseudo-URL

      ![1572858461732](/..\img\1572858461732.png)![1572858546322](./..\img\1572858546322.png)

2. 加载



3. 注册



BeanFactory和FactoryBean的区别

```
Interface to be implemented by objects used within a {@link BeanFactory} which
are themselves factories for individual objects. If a bean implements this
interface, it is used as a factory for an object to expose, not directly as a
bean instance that will be exposed itself.
```

实现了FactoryBean的接口的类，它作为一个BeanFactory来对对象暴露，不直接作为一个bean的实例来暴露它自己，拿到FactoryBean实例就可以拿到Bean实例。

ApplicationListener接口实现方法一般用来在启动时初始化缓存，注册服务（dubbo服务暴露），一些数据的加载。 可以采用多线程进行执行回调方法。

```java
public class MyAppListener implements ApplicationListener {
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("myAppListener...spring容器最后一步，触发事件监听");
    }
}		
```

说明ApplicationListener是最后一个触发的

```java
@Component
@Order(1)
public class AppInitializer implements ApplicationContextAware,ApplicationListener {
    private static Logger logger = LoggerFactory.getLogger(ApplicationContextAware.class);
    private RedisConfigManager redisConfigManager ;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        RedisConfigManager redisConfigManager = applicationContext.getBean(RedisConfigManager.class);
        //redisConfigManager.resetSequence();
        logger.info("=> 获取缓存管理");
        this.redisConfigManager = redisConfigManager;
    }

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        redisConfigManager.resetSequence();
        logger.info("=> 更新缓存");
    }
}
```

```java
public class BeanDefinitionHolder implements BeanMetadataElement {

   private final BeanDefinition beanDefinition;

   private final String beanName;

   private final String[] aliases;
}
```

核心点

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      for (BeanDefinition candidate : candidates) {
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
         if (candidate instanceof AbstractBeanDefinition) {
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         if (candidate instanceof AnnotatedBeanDefinition) {
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
         if (checkCandidate(beanName, candidate)) {
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            definitionHolder =
                  AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            beanDefinitions.add(definitionHolder);
            registerBeanDefinition(definitionHolder, this.registry);
         }
      }
   }
   return beanDefinitions;
}
```



### 注入方式 

1. setter注入
2. 构造器注入
3. 自动注入（Resource、Autowired)

Bean创建模式

1. 单例模式
2. 原型模式

### 循环依赖问题汇总

以下组合场景在循环依赖问题下能否解决：

| Bean类型\注入方式 | setter注入 | 构造器注入 | 自动注入 |
| ----------------- | ---------- | ---------- | -------- |
| 单例模式          |            |            |          |
| 原型模式          |            |            |          |



### 单例、set注入的循环依赖问题

背景：A依赖B,B依赖C,C依赖A,都是单例模式。

#### 单例的循环依赖的问题

可以理解成三级缓存，实际上也是缓存在内存中的

* 一级缓存：singletonObjects

* 二级缓存：earlySingletonObjects

* 三级缓存：singletonFactories

例子：比如说A依赖B，先创建A，实例化完成初始化时发现依赖了对象B，这个时候A对象还没有创建完成，但是通过singletonFactories提前暴露了出来。创建B，先实例化B，实例化完成之后初始化B时发现依赖了A对象，这个时候就会去三级缓存中寻找，顺序依次是singletonObjects、earlySingletonObjects、singletonFactories，肯定只有在singletonFactories中找到A对象的引用，这个时候就完成了B的初始化，B创建完成。A也就可以初始化完成。



```java
//缓存所有的单例 
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
//缓存早期单例对象，这个早期单例对象什么时候清空呢
private final Map<String, Object> earlySingletonObjects=new HashMap<String, Object>(16);
//缓存单例工厂的工厂，通过单例工厂可以产生单例获取单例
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
//标记已经创建的Bean
private final Set<String> alreadyCreated=Collections.newSetFromMap(new ConcurrentHashMap<>(256));
//记录Bean是否是在创建中
private final Set<String> singletonsCurrentlyInCreation=Collections.newSetFromMap(new ConcurrentHashMap<>(16));
//registeredSingletons只是BeanName的String类型的set集合
```

获取单例

参数allowEarlyReference 是否允许从singletonFactories中get对象

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               singletonObject = singletonFactory.getObject();
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

#### 获取当前beanName是否在创建中

```java
public boolean isSingletonCurrentlyInCreation(String beanName) {
   return this.singletonsCurrentlyInCreation.contains(beanName);
}
```

单例把自己暴露出来，目前还处于创建中，通过工厂暴露，后面依赖它的Bean就可以获取到它的依赖进行初始化了，解决循环依赖的问题，为什么要异常earlySingletonObjects呢？

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
   Assert.notNull(singletonFactory, "Singleton factory must not be null");
   synchronized (this.singletonObjects) {
      if (!this.singletonObjects.containsKey(beanName)) {
         this.singletonFactories.put(beanName, singletonFactory);
         this.earlySingletonObjects.remove(beanName);
         this.registeredSingletons.add(beanName);
      }
   }
}
```

添加到单例的缓存singletonObjects和已经注册的单例registeredSingletons

```java
protected void addSingleton(String beanName, Object singletonObject) {
   synchronized (this.singletonObjects) {
      this.singletonObjects.put(beanName, singletonObject);
      this.singletonFactories.remove(beanName); //清空
      this.earlySingletonObjects.remove(beanName);//清空
      this.registeredSingletons.add(beanName);
   }
}
```

#### 获取早期的Bean引用

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
   Object exposedObject = bean;
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
            SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
            exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
         }
      }
   }
   return exposedObject;
}
```

一个单例的两个对象A,B循环依赖，会先创建A的单例工厂，初始化的时候发现依赖了B，再创建B的单例工厂，把两个对象的单例工厂加到singletonFactories里面去，下图加入到工厂中。

![1581468449195](/..\img\1581468449195.png)

#### 循环依赖的引用是怎么填进去的？

即A<--->B,A的依赖是怎么放入到B中的，B的依赖是怎么放入到A中的。 

##### applyPropertyValues进行属性值的设置

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyPropertyValues

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
   if (pvs.isEmpty()) {
      return;
   }

   if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
      ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
   }

   MutablePropertyValues mpvs = null;
   List<PropertyValue> original;

   if (pvs instanceof MutablePropertyValues) {
      mpvs = (MutablePropertyValues) pvs;
      if (mpvs.isConverted()) {
         // Shortcut: use the pre-converted values as-is.
         try {
            bw.setPropertyValues(mpvs);
            return;
         }
         catch (BeansException ex) {
            throw new BeanCreationException(
                  mbd.getResourceDescription(), beanName, "Error setting property values", ex);
         }
      }
      original = mpvs.getPropertyValueList();
   }
   else {
      original = Arrays.asList(pvs.getPropertyValues());
   }

   TypeConverter converter = getCustomTypeConverter();
   if (converter == null) {
      converter = bw;
   }
   BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

   // Create a deep copy, resolving any references for values.
   List<PropertyValue> deepCopy = new ArrayList<>(original.size());
   boolean resolveNecessary = false;
   for (PropertyValue pv : original) {
      if (pv.isConverted()) {
         deepCopy.add(pv);
      }
      else {
         String propertyName = pv.getName();
         Object originalValue = pv.getValue();
         if (originalValue == AutowiredPropertyMarker.INSTANCE) {
            Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
            if (writeMethod == null) {
               throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
            }
            originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
         }
         Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
         Object convertedValue = resolvedValue;
          //这里是查询属性name是否可写
         boolean convertible = bw.isWritableProperty(propertyName) &&
               !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
         if (convertible) {
             //这里是关键，进行属性的设置
            convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
         }
         // Possibly store converted value in merged bean definition,
         // in order to avoid re-conversion for every created bean instance.
         if (resolvedValue == originalValue) {
            if (convertible) {
               pv.setConvertedValue(convertedValue);
            }
            deepCopy.add(pv);
         }
         else if (convertible && originalValue instanceof TypedStringValue &&
               !((TypedStringValue) originalValue).isDynamic() &&
               !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
            pv.setConvertedValue(convertedValue);
            deepCopy.add(pv);
         }
         else {
            resolveNecessary = true;
            deepCopy.add(new PropertyValue(pv, convertedValue));
         }
      }
   }
   if (mpvs != null && !resolveNecessary) {
      mpvs.setConverted();
   }

   // Set our (possibly massaged) deep copy.
   try {
      bw.setPropertyValues(new MutablePropertyValues(deepCopy));
   }
   catch (BeansException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Error setting property values", ex);
   }
}
```

##### **convertForProperty这里是对Bean的属性设置传入的值**

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#convertForProperty

```java
@Nullable
private Object convertForProperty(
      @Nullable Object value, String propertyName, BeanWrapper bw, TypeConverter converter) {

   if (converter instanceof BeanWrapperImpl) {
      return ((BeanWrapperImpl) converter).convertForProperty(value, propertyName);
   }
   else {
      PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
      MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
      return converter.convertIfNecessary(value, pd.getPropertyType(), methodParam);
   }
}
```

### 构造器注入的循环依赖的问题

构造器的注入方式，A，B相互通过构造器依赖，A实例化的时候会调用构造器，会依赖B的依赖，B这个时候还没有实例化，B如果去实例化，也需要A的依赖，这个时候A是没有实例化完成的，所以无法获取到A的引用，B就无法完成实例化，同时A也无法获取到B的引用来完成实例化，这个过程在理论上是不可行的。

### 创建指定的Bean 

做事情的方法：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean()

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {
   // Instantiate the bean.
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
      instanceWrapper = createBeanInstance(beanName, mbd, args); //创建Bean
   }
   final Object bean = instanceWrapper.getWrappedInstance();
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }
    
   //....
 
    //EarlyingSingleton 在这里解决循环依赖的问题当被BeanFactoryAware触发
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isTraceEnabled()) {
         logger.trace("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
       //循环依赖：把依赖的SingletonFactory添加进来，即以工厂的方式将自己的引用暴露出去，由于是单例，堆内存地址是唯一的，入参函数是获取早起的Bean引用，即工厂接口的方法
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }
   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
      populateBean(beanName, mbd, instanceWrapper);
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }
   if (earlySingletonExposure) {
      Object earlySingletonReference = getSingleton(beanName, false);
      if (earlySingletonReference != null) {
         if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
         }
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  actualDependentBeans.add(dependentBean);
               }
            }
         }
      }
   }
   // Register bean as disposable.
   try {
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }
   return exposedObject;
}
```

**获取单例，当A依赖B，A,B都实例化之后，B初始化时需要A的依赖，会通过该方法来寻找到A的引用，即通过三级缓存工厂方法来获取**

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   Object singletonObject = this.singletonObjects.get(beanName);
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         singletonObject = this.earlySingletonObjects.get(beanName);
         if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
                //从工厂方法获取单例
               singletonObject = singletonFactory.getObject();
               this.earlySingletonObjects.put(beanName, singletonObject);
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}
```

### 获取Bean

doGetBean-->doCreateBean 

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

   final String beanName = transformedBeanName(name);
   Object bean;

   // Eagerly check singleton cache for manually registered singletons.
   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      BeanFactory parentBeanFactory = getParentBeanFactory();
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         String nameToLookup = originalBeanName(name);
         if (parentBeanFactory instanceof AbstractBeanFactory) {
            return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                  nameToLookup, requiredType, args, typeCheckOnly);
         }
         else if (args != null) {
            // Delegation to parent with explicit args.
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else if (requiredType != null) {
            // No args -> delegate to standard getBean method.
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
         else {
            return (T) parentBeanFactory.getBean(nameToLookup);
         }
      }

      if (!typeCheckOnly) {
         markBeanAsCreated(beanName);
      }

      try {
         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

          //保证当前bean所依赖的bean的初始化。
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
            for (String dep : dependsOn) {
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               registerDependentBean(dep, beanName);
               try {
                  getBean(dep);
               }
               catch (NoSuchBeanDefinitionException ex) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
               }
            }
         }

         // Create bean instance.
         if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
               try {
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

         else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            Object prototypeInstance = null;
            try {
               beforePrototypeCreation(beanName);
               prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
               afterPrototypeCreation(beanName);
            }
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

         else {
            String scopeName = mbd.getScope();
            final Scope scope = this.scopes.get(scopeName);
            if (scope == null) {
               throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
            }
            try {
               Object scopedInstance = scope.get(beanName, () -> {
                  beforePrototypeCreation(beanName);
                  try {
                     return createBean(beanName, mbd, args);
                  }
                  finally {
                     afterPrototypeCreation(beanName);
                  }
               });
               bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {
               throw new BeanCreationException(beanName,
                     "Scope '" + scopeName + "' is not active for the current thread; consider " +
                     "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                     ex);
            }
         }
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }

   // Check if required type matches the type of the actual bean instance.
   if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
         T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
         if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
         }
         return convertedBean;
      }
      catch (TypeMismatchException ex) {
         if (logger.isTraceEnabled()) {
            logger.trace("Failed to convert bean '" + name + "' to required type '" +
                  ClassUtils.getQualifiedName(requiredType) + "'", ex);
         }
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
   }
   return (T) bean;
}
```



org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean

获取Bean定义

判断Bean为单例，调用getSingleton，getSingleton传入lambda单例工厂，工厂调用doCreateBean创建Bean,

doCreateBean创建Bean需要依赖其他对象

org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean


