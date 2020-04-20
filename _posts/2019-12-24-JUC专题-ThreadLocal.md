---
layout:     post
title:	JUC
subtitle: 	ThreadLocal
date:       2018-08-29
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - mybatis
---
# ThreadLocal

ThreadLocal存在的必要，如果是基本数据类型，即线程私有的，无需什么ThreadLocal。由于java内存结构，对象存放在公共区堆，线程A new一个对象，线程B中可以对A创建的对象进行修改，这是肯定的，若存在该对象的类型需要每个线程私有对象实例，ThreadLocal就是为这种情况开发的吧（我主管看法）。

ThreadLocal只是对一种类型进行封装，并不存储对象实例，对象实例真正存放在线程本地ThreadLocal.ThreadLocalMap变量中，即Map结构中，所以一个线程可以存放多种类型的实例数据，存储在同一个Map中，数据存放在Value中，key为ThreadLocal实例。 可以定义多个ThreadLocal相同类型。

ThreadLocal.ThreadLocalMap是Thread类中的属性之一，可以值是弱引用，GC时会回收，但是该ThreadLocal 是new出来的，所以有强引用指向它。

ThreadLocalMap 是存放在数组中的，扩容是 2 * oldLen ，初始长度是16，扩容后长度和初始长度都和HashMap一样。

问题：

1. 为什么ThreadLocal使用的是弱引用，并且在Map的key值上使用了弱引用？



父子线程上下文传递：

```java
public class InheritableThreadLocalDemo {
    static InheritableThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();
    static ThreadLocal<String> threadLocal = new ThreadLocal<>();
    public static void main(String[] args) {
        inheritableThreadLocal.set("父线程 : ThreadLocal 会传递到子线程");
        threadLocal.set("父线程 : ThreadLocal 不会传递到子线程");
        Thread thread1 = new Thread(() -> {
            System.out.println(inheritableThreadLocal.get());
            System.out.println(threadLocal.get());
        });
        thread1.start();
    }
}
//输出：
//父线程 : ThreadLocal 会传递到子线程
//null
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

#### 强引用 软引用 弱引用  虚引用

ThreadLocal 弱引用是key

如果不remove会产生大对象，线程池执行完任务可能并没有销毁，执行下一个任务的时候会还能读取得到。

弱引用通过isEnQueued()方法判断是否在标记队列，是否被垃圾回收器标记

注意：

使用线程池，线程使用结束后 需remove ThreadLocal，否则或干扰后面任务的执行。



### ThreadLocal源码分析

#### java.lang.ThreadLocal#set

```java
// 首先会获取当前线程的Map数据结构，再进行数据set
public void set(T value) {
    Thread t = Thread.currentThread(); //获取当前线程
    ThreadLocalMap map = getMap(t); //获取当前线程本地变量
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value); //当前线程ThreadLocal实例为空，创建一个，若父线程有继承，则会继承父线程的InheriableThreadLocal
}
	//java.lang.ThreadLocal#createMap
 void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue); //this 是ThreadLocal的实例，也是Map的key,即虚引用。
    }
```

#### java.lang.ThreadLocal.ThreadLocalMap#remove

```java
/**  Remove the entry for key. */
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear(); //java.lang.ref.Reference#referent = null
            expungeStaleEntry(i);
            return;
        }
    }
}
```



### WeakHashMap源码分析





### ThreadLocal案例





#### PageHelper实现分页原理

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



##### 过滤器(Filter)



拦截器（Interceptor）

> 使用反射机制实现的

​	

##### 反射机制

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