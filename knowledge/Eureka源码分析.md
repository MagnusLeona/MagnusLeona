### Eureka源码分析

#### EurekaServer

#### 一、EurekaServer启动

​		万物起源：EurekaBootStrap。这个类继承了ServletContextListener接口，在tomcat容器初始化的时候，将会执行这个方法里面的contextInitialized方法。

```java
public void contextInitialized(ServletContextEvent event) {
    try {
        initEurekaEnvironment();
        initEurekaServerContext();

        ServletContext sc = event.getServletContext();
        sc.setAttribute(EurekaServerContext.class.getName(), serverContext);
    } catch (Throwable e) {
        logger.error("Cannot bootstrap eureka server :", e);
        throw new RuntimeException("Cannot bootstrap eureka server :", e);
    }
}
```

