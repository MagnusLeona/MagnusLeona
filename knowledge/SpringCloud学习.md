## SpringCloud学习

### SpringCloud注册中心

注册中心：所有的服务都会注册到注册中心。

##### Eureka:

eureka分为服务端和客户端：服务端负责接收客户端的注册请求，并给客户端分发所有服务列表。

服务端pom依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-netflix-eureka-server</artifactId>
</dependency>
```

客户端pom依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### 服务调用：

微服务之间互相调用：

##### restTemplate

​	restTemplate基于Http请求去处理其他服务的调用。

```java
// 引入bean
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
	return new RestTemplate();
}

// 调用方式：
Map forObject = restTemplate.getForObject("http://SERVICE-PAYMENT/payment/get/" + paymentId, Map.class);

// 需要注意的是，因为加了LoadBalanced注解，表示需要使用ribbon的负载均衡策略，这时候使用restTemplate就不需要写死服务器的ip，只需要写服务注册到eureka的名称即可。但是这个名称不能包含下划线，否则会报错找不到！（SERVICE-PAYMENT）
```

​	restTemplate如果想实现负载均衡，需要加上@LoadBalanced注解。这个默认是使用spring-cloud的load-balancer组件。通过注入loadbalancerInterceptor这个插件来实现负载均衡。



- RestTemplate和LoadBalancer结合使用解析：

  RestTemplate将web请求封装为ClientHttpRequest的实现类，然后调用ClientHttpRequest.execute实现请求其余应用。

  ```java
  // ClientHttpRequest 继承关系为：
  // InterceptingClientHttpRequest     ---extends--->     AbstractBufferingClientHttpRequest    ---implements--->    ClientHttpRequest
      
  //InterceptingClientHttpRequest内部封装了List<ClientHttpRequestInterceptor> interceptors,这个Interceptors就包含了LoadBalancerInterceptor，然后LoadBalancerInterceptor这个类的intercept方法就会使用注入的LoadBalancerClient去帮我们处理请求（默认是BlockingLoadBalancerClient）。
      
  @Override
  public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
                                      final ClientHttpRequestExecution execution) throws IOException {
      final URI originalUri = request.getURI();
      String serviceName = originalUri.getHost();
      Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
      return this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
  }
  
  
  // LoadBalancer组件的所有bean注入依赖关系：
  // LoadBalancerAutoConfiguration:  注入了loadBalancedRestTemplateInitializerDeprecated,loadBalancerRequestFactory这两个bean
  // LoadBalancerAutoConfiguration.LoadBalancerInterceptorConfig内部类注入了loadBalancerInterceptor，restTemplateCustomizer两个bean，并且后面这个bean给restTemplate注入了loadBalancerInterceptor
  
  @Bean
  @ConditionalOnMissingBean
  public RestTemplateCustomizer restTemplateCustomizer(final LoadBalancerInterceptor loadBalancerInterceptor) {
      return restTemplate -> {
          List<ClientHttpRequestInterceptor> list = new ArrayList<>(restTemplate.getInterceptors());
          list.add(loadBalancerInterceptor);
          restTemplate.setInterceptors(list);
      };
  }
  
  // loaderBalancerClient是在BlockingLoadBalancerClientAutoConfiguration中注册的
  @Bean
  @ConditionalOnBean({LoadBalancerClientFactory.class})
  @ConditionalOnMissingBean
  public LoadBalancerClient blockingLoadBalancerClient(LoadBalancerClientFactory loadBalancerClientFactory) {
      return new BlockingLoadBalancerClient(loadBalancerClientFactory);
  }
  ```



##### feign

配置方法：

```xml
// 引入pom依赖
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <version>3.1.0</version>
</dependency>
```

```java
// 主类注入OpenFeign --- @EnableFeignClients
@SpringBootApplication
@EnableFeignClients
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}
```

```java
// 定义需要用到的其他服务接口，通过注解来WebRest接口调用
@Component
@FeignClient(value = "SERVICE-PAYMENT")
public interface PaymentFeignService {

    @GetMapping("/payment/get/{id}")
    public MagnusResponse getPaymentById(@PathVariable Long id);
}
```

```java
// 在需要调用其他服务的地方直接注入即可
@Autowired
PaymentFeignService paymentFeignService;
```

- Feign底层使用LoadBalancer实现负载均衡
- 如果想要定义同一个name的不同FeignClient，必须要加入contextId参数去标识不同的client



##### eureka源码分析：	

- EurekaClient源码分析:

  - eurekaclient源码最关键的处理类是DiscoveryClient,被注入到Spring容器中的是它的子类CloudDiscoveryClient。Discovery中，存放着client自己的实例信息`private final InstanceInfo instanceInfo;`,存放着所有注册到eureka server的服务实例`private final AtomicReference<Applications> localRegionApps;`。

  - 在Discovery初始化的时候，会设置两个定时任务：

    ```java
    this.scheduler = Executors.newScheduledThreadPool(2, (new ThreadFactoryBuilder()).setNameFormat("DiscoveryClient-%d").setDaemon(true).build());
                    this.heartbeatExecutor = new ThreadPoolExecutor(1, this.clientConfig.getHeartbeatExecutorThreadPoolSize(), 0L, TimeUnit.SECONDS, new SynchronousQueue(), (new ThreadFactoryBuilder()).setNameFormat("DiscoveryClient-HeartbeatExecutor-%d").setDaemon(true).build());
                    this.cacheRefreshExecutor = new ThreadPoolExecutor(1, this.clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0L, TimeUnit.SECONDS, new SynchronousQueue(), (new ThreadFactoryBuilder()).setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d").setDaemon(true).build());
    ```

    这两个定时任务是为了给server端续约(renew)、获取最新的注册列表(refreshRegistry)。另外在初始化的时候，会立刻获取一次Server端的注册信息`boolean primaryFetchRegistryResult = this.fetchRegistry(false);`。



#### Gateway

- id

  路由的id，不可以重复。Gateway处理网关的请求基于routes路由，只有路由能正确映射时，才能正确处理请求。

- uri

  表示目标请求地址。如果为lb,则表示走注册中心查询服务名称对应的服务器ip。

- Predicate

  Predicate断言的含义是，满足要求的请求将被这个路由处理。

- Filter

  - RequestRateLimiter: 限流组件

    - RedisRateLimiter:

      ```xml
      <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-redis-reactive -->
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
          <version>2.6.1</version>
      </dependency>
      ```

      RedisRateLimiter使用的是令牌桶限流算法（Token Bucket Algorithm）.

      ```properties
      #每秒用户可以请求的数量 ---每秒生成的新的令牌数量
      redis-rate-limiter.replenishRate 
      #每秒最大请求数量 --- 最大令牌数
      redis-rate-limiter.burstCapacity 
      #每次请求消耗的令牌数量  ---每次请求需要消耗的令牌数量
      redis-rate-limiter.requestedTokens 
      ## Spring官网给出的案例： 设置replenishRate为1， burstCapacity为60， requestedTokens为60，则可以限制每分钟请求一次
      ```

      配置模板：

      ```yaml
      spring:
      	redis:
      		database: 1
      		host: localhost
      		port:6379
      	cloud:
      		gateway:
      			routes:
      				- id: some-id
      				  uri: https://example.org
      				  filters:
      				  	- name: RequestRateLimiter
      				  	  args: 
      				  	  	redis-rate-limiter.relenishRate: 10
      				  	  	redis-rate-limiter.burstCapacity: 20
      				  	  	redis-rate-limiter.requestedTokens: 1
      ```

      需要限流的路径，需要通过KeyResolver去指定：

      ```java
      @Bean
      KeyResolver userKeyResolver() {
          return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));
      }
      // 表示需要拦截的请求是请求参数中包含user的
      ```

      ```java
      @Bean
      KeyResolver ipKeyResolver() {
          return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
      }
      // 表示根据ip地址来限流
      ```

      ```java
      @Bean
      KeyResolver apiKeyResolver() {
          return exchange -> Mono.just(exchange.getRequest().getPath().value());
      }
      // 表示根据请求路径去做限流
      ```

      将限流的条件加入限流配置中：

      ```yaml
      spring:
      	cloud:
      		gateway:
      			routes:
      				- id: some-id
      				  uri: https://example.org
      				  filters:
      				  	- name: RequestRateLimiter
      				  	  args: 
      				  	  	redis-rate-limiter.relenishRate: 10
      				  	  	redis-rate-limiter.burstCapacity: 20
      				  	  	redis-rate-limiter.requestedTokens: 1
      				  	  	key-resolver: "#{@ipKeyResolver}"
      				  	  	key-resolver: "#{@apiKeyResolver}"
      ```

  - RedirectTo: 转发过滤器

    - 将请求转发到另外一个服务器，包含两个参数，status请求状态，url转发服务器路径

      ```yaml
      spring:
      	cloud:
      		gateway:
      			routes:
      				- id: prefix-path
      				  uri: http://example.org
      				  filters:
      				  	- RedirectTo=302,http://baidu.com
      ```

  - RewritePath: 路径重写

    - 重新请求路径

      ```yaml
      spring:
      	cloud:
      		gateway:
      			routes:
      				- id: prefix-path
      				  uri: http://example.org
      				  predicates:
      				  	- Path=/red/**
      				  filters:
      				  	- RewritePath=/red/?(?<segment>.*), /$\{segement}
      ```

- Gateway+Eureka实现注册中心服务转发：

  - 首先需要引入eureka-client的依赖项：

    ```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>	
    ```

  - 然后设置启动类：

    ```java
    @SpringBootApplication
    @EnableDiscoveryClient
    public class MagnusApplication {
    	public static void main(String[] args) {
    		SpringApplication.run(MagnusApplication.class, args);
    	}
    }
    // @EnableDiscoveryClient和@EnableEurekaClient的区别： @EnableDiscoveryClient适用于大部分的注册中心（Consule, Zuul,Eureka），但是@EnableEurekaClient仅能用于Eureka
    ```

  - 配置文件：

    ```yaml
    spring:
        application:
          name: gateway
        cloud:
          gateway:
            discovery:
              locator:
                enabled: true  # 从注册中心获取服务列表
                lower-case-service-id: true
     		routes:
              - id: service-payment
                uri: lb://service-payment
                order: -1
                predicates:  #断言是用来决定某个路由是否由这个route处理
                  - Path=/service-payment/**
                  - Before=2030-12-16T15:53:22.999+08:00[Asia/Shanghai]
                filters:
                  - StripPrefix=1
    ```

  - 使用配置中心+gateway的时候，前端请求都走gateway即可，需要加上应用注册的实例名，如：

    ```js
    http://localhost:7000/service-payment/payment/get/1
    ```

    那么，我们配置文件中的predicates里面就得对service-payment做请求断言。

    gateway中的uri表示目标请求地址，也就是我们需要访问的实际服务。

    整个请求链路为： `http://localhost:7000/service-payment/payment/get/1` -> `http://localhost:8000/payment/get/1` 

    只有当请求正确的走到我们预设的断言route中，才能对请求做相应的filter过滤处理。



- Gateway源码分析：

  ```java
  ```

  

### 熔断

- Hystrix

  - 基本使用方法：

    Hystrix将需要执行的操作封装到Command中：

    ```java
    public class HystrixHelloWorld extends HystrixCommand<String> {
    
        public String name;
    
        public HystrixHelloWorld(String name) {
            super(HystrixCommandGroupKey.Factory.asKey("HelloWord"));
            this.name = name;
        }
    
        @Override
        protected String run() throws Exception {
            return "hello word!" + this.name;
        }
    
        public static void main(String[] args) throws ExecutionException, InterruptedException {
            HystrixHelloWorld hystrixHelloWorld = new HystrixHelloWorld("Magnus");
            String execute = hystrixHelloWorld.execute();
            System.out.println(execute);
        }
    }
    ```

    封装到HystrixCommand中，然后通过execute方法去调用。

