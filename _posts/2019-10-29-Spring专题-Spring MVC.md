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









```java
public interface WebApplicationContext extends ApplicationContext {

   /**
    * Context attribute to bind root WebApplicationContext to on successful startup.
    * <p>Note: If the startup of the root context fails, this attribute can contain
    * an exception or error as value. Use WebApplicationContextUtils for convenient
    * lookup of the root WebApplicationContext.
    * @see org.springframework.web.context.support.WebApplicationContextUtils#getWebApplicationContext
    * @see org.springframework.web.context.support.WebApplicationContextUtils#getRequiredWebApplicationContext
    */
   String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";

   /**
    * Scope identifier for request scope: "request".
    * Supported in addition to the standard scopes "singleton" and "prototype".
    */
   String SCOPE_REQUEST = "request";

   /**
    * Scope identifier for session scope: "session".
    * Supported in addition to the standard scopes "singleton" and "prototype".
    */
   String SCOPE_SESSION = "session";

   /**
    * Scope identifier for global session scope: "globalSession".
    * Supported in addition to the standard scopes "singleton" and "prototype".
    */
   String SCOPE_GLOBAL_SESSION = "globalSession";

   /**
    * Scope identifier for the global web application scope: "application".
    * Supported in addition to the standard scopes "singleton" and "prototype".
    */
   String SCOPE_APPLICATION = "application";

   /**
    * Name of the ServletContext environment bean in the factory.
    * @see javax.servlet.ServletContext
    */
   String SERVLET_CONTEXT_BEAN_NAME = "servletContext";
}
```