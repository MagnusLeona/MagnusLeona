## Springboot学习概述

### Springboot自动装配原理：

- Springboot自动装配依赖于Jar包：autoconfigure。

- 注解@SpringbootApplication中，引入了关键性注解@EnableAutoConfiguration，这个注解的目的是引入AutoConfigurationImportSelector类。这个类调用了Spring-core中的spring服务发现机制：SpringFactoryLoader，这个类会使用AppClassLoader去扫描所有jar包中的META-INF/spring.factories文件，走这个里面拿到需要的自动配置类(通用的自动配置类全部都写在autoconfigure包中的META-INF/spring.factories中)。

- 关键点：<font color="red">通过AppClassLoader.getResources会扫描所有jar包中对应的目录</font>。

- 如果需要自定义starter，就可以使用springboot的服务发现机制：给自定义的starter包中加入spring.factories文件，然后写上我们的自动配置类:

  ```properties
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=xxx
  ```

  那么springboot启动的时候就会扫描到这个类并注入到容器中。

- Springboot根据自动配置类上的@ConditionalOnClass去决定某个配置类是否需要被导入到容器中。



### SpringBoot自动配置类中是怎么装配方法参数的

- 以Gateway自动配置类为例：

  Gateway引入的依赖为：

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
      <version>2.6.1</version>
  </dependency>
  ```

  spring.factories:

  ```properties
  # Auto Configure
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.gateway.config.GatewayClassPathWarningAutoConfiguration,\
  org.springframework.cloud.gateway.config.GatewayAutoConfiguration,\
  org.springframework.cloud.gateway.config.GatewayResilience4JCircuitBreakerAutoConfiguration,\
  org.springframework.cloud.gateway.config.GatewayNoLoadBalancerClientAutoConfiguration,\
  org.springframework.cloud.gateway.config.GatewayMetricsAutoConfiguration,\
  org.springframework.cloud.gateway.config.GatewayRedisAutoConfiguration,\
  org.springframework.cloud.gateway.discovery.GatewayDiscoveryClientAutoConfiguration,\
  org.springframework.cloud.gateway.config.SimpleUrlHandlerMappingGlobalCorsAutoConfiguration,\
  org.springframework.cloud.gateway.config.GatewayReactiveLoadBalancerClientAutoConfiguration,\
  org.springframework.cloud.gateway.config.GatewayReactiveOAuth2AutoConfiguration
  
  org.springframework.boot.env.EnvironmentPostProcessor=\
  org.springframework.cloud.gateway.config.GatewayEnvironmentPostProcessor
  
  # Failure Analyzers
  org.springframework.boot.diagnostics.FailureAnalyzer=\
  org.springframework.cloud.gateway.support.MvcFoundOnClasspathFailureAnalyzer
  ```

  这里重点关注GatewayAutoConfiguration.class:

  ```java
  @Bean
  public RouteLocator routeDefinitionRouteLocator(GatewayProperties properties,
                                                  List<GatewayFilterFactory> gatewayFilters, List<RoutePredicateFactory> predicates,
                                                  RouteDefinitionLocator routeDefinitionLocator, ConfigurationService configurationService) {
      return new RouteDefinitionRouteLocator(routeDefinitionLocator, predicates, gatewayFilters, properties,
                                             configurationService);
  }
  
  ```

  里面涉及一个重要的方法：routeDefinitionRouteLocator,这个方法是去注册RouteLocator作为Bean。但是这个方法里面涉及两个重要的参数：List<GatewayFilterFactory>,List<RoutePredicateFactory>。这两个参数是怎么被注入的呢？

- 因为routeDefinitionRouteLocator是在GatewayAutoConfiguration这个factoryBean中，所以在解析到创建这个bean的时候，调用方法为：`AbstractAutowireCapableBeanFactory.createBeanInstance`,然后解析到`instantiateUsingFactoryMethod`方法。

- Spring会根据List<T>中的泛型找到所有继承了这个接口的实例bean并注入到List中。

  


#### Springboot注册Servlet

​	springboot注册Servlet使用ServletRegistrationBean，注册这个Bean即可。

    ```java
    @Configuration
    public ServletConfiguration {
        @Bean
        public ServletRegistrationBean mainServlet() {
            return new ServletRegistration(new MainServlet());
        }
    }
    ```

