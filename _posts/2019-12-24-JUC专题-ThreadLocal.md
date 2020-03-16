---
layout:     post
title:	JUC
subtitle: 	ThreadLocal+PageHelper实现分页原理
date:       2018-08-29
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - mybatis
---
# ThreadLocal



set方法

```java
//java.lang.ThreadLocal#set
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```



```java
//java.lang.Thread#threadLocals
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;

/*
 * Inheritable ThreadLocal values pertaining to this thread. This map is
 * maintained by the InheritableThreadLocal class.
 */
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

强引用 软引用 弱引用  虚引用

ThreadLocal 弱引用是key

如果不remove会产生大对象，线程池执行完任务可能并没有销毁，执行下一个任务的时候会还能读取得到。



### PageHelper实现分页原理

>  PageHelper.offsetPage(PAGE_NUM, PAGE_SIZE) （页码，每页显示的数量）。
>
> 实际是在 ThreadLocal中设置了分页参数，之后在查询执行的时候，获取当线程中的分页参数，执行查询的时候通过拦截器在sql语句中添加分页参数，之后实现分页查询，查询结束后在 finally 语句中清除ThreadLocal中的查询参数。

com.github.pagehelper.page.PageMethod#setLocalPage

```java
protected static final ThreadLocal<Page> LOCAL_PAGE = new ThreadLocal<Page>();

    /**
     * 设置 Page 参数
     *
     * @param page
     */
    protected static void setLocalPage(Page page) {
        LOCAL_PAGE.set(page);
    }
```

逻辑分页



物理分页

PageInterceptor拦截器拦截query()查询方法

分页的参数放在LocalThread里面



### 过滤器(Filter)



拦截器（Interceptor）

> 使用反射机制实现的

​	

#### 反射机制

##### jdk自带的InvocationHandler

> 只能实现接口实现类的代理

```java
public interface Animal {
    void eat();
}
```

```java
public class Cat implements Animal {
    /**
     * java 自带的动态代理 需要实现接口
     */
    @Override
    public void eat() {
        System.out.println("cat is eatting");
    }
}
```

> 自定义InvocationHandler的实现

```java
public class CatInvocationHandler implements InvocationHandler {
    private Object object;

    CatInvocationHandler(Object object) {
        this.object = object;
    }

    public Object newProxyInstance() {
        return Proxy.newProxyInstance(object.getClass().getClassLoader(), object.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("调用eat()之前");
        Object result =  method.invoke(object, args);
        System.out.println("调用eat()之后");
        return result;
    }
}
```

> 调用被动态代理的接口方法

```java
public class Main {
    public static void main(String[] args) {
        Cat cat = new Cat();
        CatInvocationHandler d = new CatInvocationHandler(cat);
        Animal animal = (Animal) d.newProxyInstance();
        animal.eat();
    }
}
```

> 结果:
> 调用eat()之前
> cat is eatting
> 调用eat()之后

##### cglib的MethodInterceptor

> 可以对接口和类进行代理