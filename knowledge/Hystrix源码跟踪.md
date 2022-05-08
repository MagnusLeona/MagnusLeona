## Hystrix源码跟踪

### 从@HystrixCommand注解开始说起：

SpringBoot中集成了Hystrix自动配置，使用方式如下：

```java
@HystrixCommand(fallbackMethod = "failureFallback", commandProperties = {
            @HystrixProperty(name = HystrixPropertiesManager.CIRCUIT_BREAKER_ERROR_THRESHOLD_PERCENTAGE, value = "50"),
            @HystrixProperty(name = HystrixPropertiesManager.CIRCUIT_BREAKER_REQUEST_VOLUME_THRESHOLD, value = "20"),
            @HystrixProperty(name = HystrixPropertiesManager.CIRCUIT_BREAKER_SLEEP_WINDOW_IN_MILLISECONDS, value = "20")
    })
@Override
public String testHystrixCommandFailure() {
    int i = 1 / 0;
    return "success";
}

public String failureFallback() {
    System.out.println("FailureFallBack invoked");
    return "failure---command";
}
```

其中，@HystrixCommand注解使用了AOP切面的方式。切面是HystrixCommandAspect.java。

- 开始执行：首先会进入到切点的环绕方法methodsAnnotatedWithHystrixCommand。

  ```java
  public Object methodsAnnotatedWithHystrixCommand(ProceedingJoinPoint joinPoint) throws Throwable {
      Method method = AopUtils.getMethodFromTarget(joinPoint);
      // .....
      HystrixCommandAspect.MetaHolderFactory metaHolderFactory = (HystrixCommandAspect.MetaHolderFactory)META_HOLDER_FACTORY_MAP.get(HystrixCommandAspect.HystrixPointcutType.of(method));
      MetaHolder metaHolder = metaHolderFactory.create(joinPoint);
      HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder);
      ExecutionType executionType = metaHolder.isCollapserAnnotationPresent() ? metaHolder.getCollapserExecutionType() : metaHolder.getExecutionType();
  	// .....
      Object result;
      if (!metaHolder.isObservable()) {
          result = CommandExecutor.execute(invokable, executionType, metaHolder);
      } else {
          result = this.executeObservable(invokable, executionType, metaHolder);
      }
  
      return result;
  }
  ```

- 进入metaHolderFactory.create方法可以看下是怎么创建的

  ```java
  //  跟踪到metaHolderFactory.create方法
  MetaHolder metaHolder = metaHolderFactory.create(joinPoint); 
  // 这个方法封装了一些基本的command配置到MetaHolder中，比如实际的方法，代理的类，fallback方法等。
  // 对于普通的熔断操作，command标识是SYNCHRONUS同步
  ```

- 进入HystrixCommandFactory.getInstance().create(metaHolder)方法，这时候开始根据MetaHolder创建command。

  ```java
  public HystrixInvokable create(MetaHolder metaHolder) {
      Object executable;
      if (metaHolder.isCollapserAnnotationPresent()) {
          executable = new CommandCollapser(metaHolder);
      } else if (metaHolder.isObservable()) {
          executable = new GenericObservableCommand(HystrixCommandBuilderFactory.getInstance().create(metaHolder));
      } else {
          executable = new GenericCommand(HystrixCommandBuilderFactory.getInstance().create(metaHolder));
      }
  
      return (HystrixInvokable)executable;
  }
  ```

  普通的熔断命令没有Collapser，且不是Observable，所以直接新建GenericCommand;

- 具体的执行在CommandExecutor.execute(invokable, executionType, metaHolder)，开始处理请求。

  执行流程：

  ```java
  // CommandExecutor.class:
  public static Object execute(HystrixInvokable invokable, ExecutionType executionType, MetaHolder metaHolder) {
      // ......
      switch(executionType) {
          case SYNCHRONOUS:
              return castToExecutable(invokable, executionType).execute();
          case ASYNCHRONOUS:
              HystrixExecutable executable = castToExecutable(invokable, executionType);
              if (metaHolder.hasFallbackMethodCommand() && ExecutionType.ASYNCHRONOUS == metaHolder.getFallbackExecutionType()) {
                  return new FutureDecorator(executable.queue());
       }
      // ......
  	return executable.queue();
  }
  ```

  castToExecutable将HystrixInvokable转换为HystrixExecutable。然后执行execute方法。开始执行HystrixCommand.execute方法。

  ```java
  public R execute() {
      try {
          return queue().get();
      } catch (Exception e) {
          throw Exceptions.sneakyThrow(decomposeException(e));
      }
  }
  ```

  

```java
// Hystrix中，熔断器的判断类是 HystrixCircuitBreaker接口的内部实现类HystrixCircuitBreaker
static class HystrixCircuitBreakerImpl implements HystrixCircuitBreaker
// 这是个静态内部类
```

