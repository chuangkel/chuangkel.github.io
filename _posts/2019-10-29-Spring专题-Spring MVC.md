---
layout:     post
title:	Spring专题
subtitle: 	Spring MVC
date:       2019-10-29
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - spring
---

# Spring MVC



MVC是一种通用的设计模式（视图 控制器 模型）

在Spring MVC里面，由于java只能返回一个对象，所以只能返回一个 既包含模型又包含视图的对象，即ModelAndView.

### 处理器(Handler)  -  HandlerMapping

绝大多数实现类继承AbstractHandlerMapping，返回执行链。

包含多个有序的interceptors

HandlerInterceptor （不带）

MappedInterceptor(带映射规则)

> MappedInterceptor会被适配成HandlerInterceptor
>
>  RequestMappingHandlerMapping是专属@RquesetMapping

### 处理器（Handler) 

*  主要有两种处理器（实现Controller接口的一种，标注了@RequestMapping的一种）：

  * 标注@RequestMapping注解的方法

  * 实现Controller接口的handleRequset方法

    ```java
    public interface Controller {
        ModelAndView handleRequest(HttpServletRequest var1, HttpServletResponse var2) throws Exception;
    }
    ```

    通过HandlerMapping找到执行链（执行链是一种数据结构，包含了Handler和拦截器)。

  ```java
  public interface HandlerMapping {
    @Nullable//返回的视图可空
     HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
  }
  ```

  

### 处理器（Handler）-  执行链

根据request找到HandleExecutionChain执行链(本质是一个数据结构，包含了一个Handler和拦截器数组)（责任链的设计模式）

* 委派集合执行
  * org.springframework.web.servlet.HandlerExecutionChain#applyPreHandle
  * org.springframework.web.servlet.HandlerExecutionChain#applyPostHandle
  * org.springframework.web.servlet.HandlerExecutionChain#triggerAfterCompletion(被拦截掉，这个方法会被执行)

### 处理器（Handler）-  拦截器HandlerInterceptor

所有的过滤器都实现了HandlerIntercepter

拦截器HandlerInterceptor和过滤器Filter的区别：

1. Filter是拦截Servlet的	，一旦被拦截，后续Servlet不被执行

   Filter1->Filter2->Filter3->servlet

   FilterChain = Filter* N + Servlet ,  HandlerInterptor = HandlerInterceptor * N + Handler

2. Filter前置后置处理需要开发者自己管理

```java
public class HeadFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {}
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        try {
            // do before
            chain.doFilter(request, response);
            //do after
        }catch (Exception e){
            throw new ServletException(e);
        }finally {
            //do finally
        }
    }
    @Override
    public void destroy() {}
}
```

3. Filter拦截请求形式

```java
public enum DispatcherType {
    FORWARD,
    INCLUDE,
    REQUEST,
    ASYNC,
    ERROR
}
```

4. 从下面可以看出Filter和Dispatcher是平行的，独立于Dispatcher的。而HandlerInterceptor和Dispatcher是绑定关系，多对一的关系（子关系）。

```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
</servlet>
<servlet-mapping>
	<servlet-name>dispatcher</servlet-name>
    <servlet-pattern>/abc</servlet-pattern>
</servlet-mapping>
<filter>
    <filter-name>filter</filter-name>
    <filter-class>com.amce.MyFilter</filter-class>
</filter>
<filter-mapper>
	<filter-name>filter</filter-name>
    <filter-pattern>/myfilter</filter-pattern>
</filter-mapper>
```

### HanderInterceptor 和 MappedInterceptor的区别

MappedInterceptor包含了Url，存在自己的URL映射关系，URL Pattern属于特殊的映射逻辑，默认AntPathMatcher,可以自定义PathMatcher，每次请求根据URL选择Interceptor，只对对应URL拦截。

HanderInterceptor是对所有的URL都拦截。

#### HanderInterceptor 和 WebRequestInterceptor的区别

WebRequestInterceptor是通用的拦截器，被拦截的请求对象是WebRequest

web的三种规范：Servlet，Portlet，JSF

### 处理器（Handler）-  拦截器 -  拦截器注册中心InterceptorRegistry

InterceptorRegistry为执行链HandlerExecutionChain和AbstractHandlerMapping提供数据来源，按照顺序注册HandlerInterceptor



![image-20200105193341868](./..\img\image-20200105193341868.png)



## Spring Web MVC

### Handler到底什么

* HandlerMethod

* Controller实现类

  Handler来源于HandlerExecutionChain org.springframework.web.servlet.HandlerMapping#getHandler(HttpServletRequest）方法实现

### 视图渲染

@VelocityLeyout

### Rest处理

@RestController

