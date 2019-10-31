---
layout:     post
title:	Spring专题
subtitle: 	IOC系列一
date:       2019-08-29
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 设计模式
---

# IOC系列一

## bean的属性

id

name

class

scop 作用域singleton，prototype原型模式，request，session

atowire bean的自动装配模式，即依赖注入方式，no不自动装配，byName通过name属性自动装配，byType通过class类全名自动装配，constructor通过构造器自动装配

init-method 

destroy-method

factory-method 静态工厂方法

factory-bean 





## 三种bean的定义

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
   //获取bean
   AnnotationConfigApplicationContext ctx=new AnnotationConfigApplicationContext(MyConfig.class);
           Person person1 = (Person) ctx.getBean("getPerson");
   ```



### 分析

先看下AnnotationConfigApplicationContext的UML类图继承关系：

![1571974164861](D:\fileSystem\persons\Github\chuangkel.github.io\img\1571974164861.png)

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