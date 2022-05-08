# Mybatis阅读笔记：



### 1.SqlSessionFactoryBuilder：

  这个类是用来实例化SqlSessionFactory的，一般只在实例化的时候用到。 在Spring中注入SqlSessionFactory只需要使用SqlSessionFactoryBean即可。

### 2.SqlSessionFactory： 

  作用域最好是单例，一旦创建就会一直存在。

###  3.SqlSession:

  每个线程都需要有自己的实例，因为这个是线程不安全的  sqlSessionFactory.openSession。在Spring中使用Mybatis，直接使用SqlSessionTemplate去生成一个sqlsession即可，这个类是线程安全的。

### 4.获取映射器实例：

  session.getMapper。

### 5.SqlSessionTemplate:

   是sqlSession接口的实现类，可以替代我们在代码中自己生成的SqlSession，并且是线程安全的，可以被多个线程共享使用。

### 6.SqlSession实现BATCH批量操作:

​	必须和Spring的事务结合起来才可以，否则会出现每一次执行都会新建sqlSession的情况，极大降低效率！！！

​	想要实现批量操作，必须有一个基于批量的Executor(注意，执行批量的操作必须在事务内进行，否则会失效):

```java
SqlSession sqlSession = sqlSessionFactory.openSession(Executor.BATCH);
UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
for (int i = 0; i < 100000; i++) {
    Map map = new HashMap();
    map.put("id", i);
    map.put("name", i);
    map.put("description", i + 1);
   	userMapper.insertUser(map);
}
```



### 7.可以使用TransactionTemplate.execute手动触发事务，但是貌似这个手动触发的批量，耗时很长，暂时还没有去研究为啥(研究出来了，触发的时长是正常的，前面测试的慢是因为我没有提交事务)。

```java
TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager);
transactionTemplate.execute(txStatus -> {
  for (int i = 0; i < 100000; i++) {
	Map map = new HashMap();
	map.put("id", i);
	map.put("name", i);
	map.put("description", i + 1);
	sqlSession.insert("com.magnus.managee.main.business.mappers.UserMapper.insertUser", map);
  }
  return null;
});
```

  需要传入一个transactionManager，作为事务的执行器。



### 8. 使用MapperScan自动扫描并注入的Mapper对象原理：

在Spring中使用自动扫描所有的Mapper，需要在一个配置类上加上@MapperScan(basePackage),原理如下：

- @MapperScan 注解内部实现了引入一个类 @Import({MapperScannerRegistrar.class})，表示在Spring容器加载这个配置类的时候，自动引入MapperScannerRegistrar类，并作为一个bean存到容器中。
- 加载MapperScannerRegistrar： 这个类实现了ImportBeanDefinitionRegistrar,表示在解析配置类（@Configuration）的时候，会处理实现了这个接口的类，并注册相应的beanDefinition。（这里需要关注ImportBeanDefinitionRegistrar这个接口。实现了这个接口的类，只能通过其他类的@Import注解来调用，作用也是注册自己的beanDefinition。子类会在BeanFactoryPostProcessor之前就被调用，所以实际上比BeanDefinitionRegistryPostProcessor先执行）。在执行接口实现方法registerBeanDefinition的时候，MapperScannerRegistrar类实例化了一个基于MapperScannerConfigurer的genericBeanDefinition，并把这个bean放到容器中，所以实际上注册的bean是MapperScannerConfigurer的bean，这个bean才是真正执行了扫描的方法类。
- MapperScannerConfigurer类： 这个类实现了BeanDefinitionRegistryPostProcessor接口、InitializingBean接口这两个最重要的接口。在BeanDefinitionRegistryPostProcessor接口实现方法中（这个方法会在beanDefinition注册完后，作为BeanFactoryPostProcessor去调用），postProcessBeanDefinitionRegistry，去实例化了一个Mybatis自己定义的ClassPathMapperScanner实例，这个实例设置了一些扫描规则（基于接口）(ClassPathMapperScanner.registerFilters())，然后又继承了Spring中的一个扫描类：ClassPathBeanDefinitionScanner，并重写了部分scan方法内容。通过调用父类的scan方法，读取了我们传入的basePackage下的所有符合过滤条件的.classs文件。这里需要非常重点的关注：过滤规则不包含基于@Mapper注解，所以理论上哪怕你的接口没有@Mapper注解，一样会被当成是Mapper去做处理！！！扫描到之后，会先基于这个.class文件生成一个ScannedGenericBeanDefinition对象，然后把这个对象存到BeanDefinitionHolder中，最后把这个BeanDefinition注册到registry中（DefaultListableBeanFactory）。
- 开始解析BeanDefinition: 执行了ClassPathMapperScanner.processBeanDefinition方法，设置这个bean的class为MapperFactoryBean，同时设置bean的实例化构造方法的参数为我们定义的mapper接口，这个MapperFactoryBean非常非常重要！！！这个Bean就是真正帮我们做事的类。这是一个FactoryBean，所以spring开始实例化各种bean的时候，读到我们的Mapper，就会调用MapperFactoryBean,把他作为FactoryBean去生成我们实际的Mapper的代理对象（根据构造方法传入的mapper接口），这个代理对象就是基于我们mapper接口实现的。所以，我们能够在其余地方去注入我们的Mapper实例！！！同时，在这里设置了AutowireMode，注入的规则为2，即根据Type去注入依赖，所以会调用MapperFactoryBean的setSqlSessionFactory方法，把我们定义的sqlSessionFactory注入到这个类中去生成sqlSession。同时也会注入sqlSesssionTemplate，这里注入的bean，就是我们定义的满足默认获取条件的sqlSession的bean。
- 来看下我们的MapperFactoryBean: 这个类继承了SqlSessionDaoSupport，在解析这个类的时候，容器会根据set方法注入sqlSession,所以MapperFactoryBean仅仅是做了sqlSession.getMapper的方法去生成我们需要的Mapper实例。

​	并且是使用的在Spring容器中能够获取到的优先级最高的SqlSession进行动态代理的处理（在容器中有多个SqlSession的情况下，原理请参考上述内容）。

​	所以当Spring中有多个SqlSession时，一定要区分去处理Mapper的注入，当然也可以使用手动的方式去注入，sqlsession.getMapper的方式，使用指定的SqlSessionTemplate手动生成代理对象。  



### 9.Mybatis使用同一个sqlSession的时候，如果执行的语句完全一样，会有相应的结果缓存。 Spring在默认非事务的情况下，mybatis会在每一个执行的mapper方法的时候去获取一个新的sqlsession。但是如果在事务处理环境中，则会使用同一个sqlSession对象(可能是通过SessionHolder的类去保存这个SqlSession实例的)。

### 10.Mybatis缓存使用方法

​	一级缓存 基于sqlsession，同一个sqlsession，默认开启缓存。

​	二级缓存是基于Mapper的命名空间，相同命名空间下能够读取到上一次查询的缓存，不同命名空间则不行，所以如果在某些比较条件下，不要随便使用二级缓存，很容易造成读取数据不一致的情况。比如两个不同的Mapper里面有一组相同的Sql语句，执行的时候有可能一个是读缓存，一个是读数据库，造成数据不一致。



  基于JavaApi的设置缓存的方法：

  ```java
  // 先配置一个包含了cache的sqlSessionFactory，然后再设置Mapper读取的cache命名空间
  @Bean
  @Primary
  public SqlSessionFactory sqlSessionFactory() throws Exception {
      SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
      sqlSessionFactoryBean.setDataSource(dataSource);
      PerpetualCache perpetualCache = new PerpetualCache("cache");
      EhcacheCache ehcacheCache = new EhcacheCache("ehcache");
      // 设置使用的cache类型，默认是perpetualCache，也可以使用自定义的Cache，比如Ehcache。
      sqlSessionFactoryBean.setCache(perpetualCache);
      org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
      SpringManagedTransactionFactory springManagedTransactionFactory = new SpringManagedTransactionFactory();
      Environment environment = new Environment("1", springManagedTransactionFactory, dataSource);
      configuration.setEnvironment(environment);
      sqlSessionFactoryBean.setConfiguration(configuration);
      // Mybatis配置日志系统，输出日志采用Log4j的形式。
      configuration.setLogImpl(Log4j2Impl.class);
      configuration.setCacheEnabled(true);
      SqlSessionFactory object = sqlSessionFactoryBean.getObject();
      return object;
  }
  
  // Mapper设置：
  @Mapper
  @CacheNameSpace("cache")
  public interface UserMapper {
      @Select("select * from magnus where id=#{id}")
      @Options(cache = false)
      public User getUserById(Integer id)
  }
  
  // 如果Mapper需要做数据缓存（二级），则需要加上@CaCheNameSpace注解，并且标明是使用的哪一个cache。入参字段需要和我们之前注册的Cache对象名称保持一致，要不然会报错找不到对应的Cache。
  // 如果在某个方法上不想做缓存，则加上@Options(cache = false)就可以了。
  ```

在Mapper上使用的，表示缓存命名空间的注解有两个，一个是@CacheNamespace,一个是@CacheNamespaceRef。前后功能类似，但是后者的目的是为了和同时配置了Mapper和XML两种方式的情况下能够使用同一个命名空间。

```java
// @CacheNamespace 使用方法：
// @CacheNamespace标记这个Mapper接口开启缓存，里面可以传入几个参数：
// 1.implementation： 表示具体使用的缓存组件，可以是任意实现了Cache接口的类，默认是mybatis自带的PerpetualCache，原理是一个HashMap。常用的是
//                    Ehcache
// 2.Properties：表示实现了Cache接口的类需要的配置项，加载这个缓存实现类的时候，会注入这些参数。
// 3.readWrite： 表示是否需要读写分离
// 4.eviction：表示使用的缓存策略，LRU，等
// 5.flushInterval: 缓存刷新间隔，表示多久刷新一次
```

### 11.MyBatis处理一对多，多对多的数据查询：

管理表情景： 一个老师可能教很多个学生，一个学生可能有多个老师，所以有三张表： 教师，学生，教师学生关系表。查询条件如下： 查询所有的老师，并且查询出每个老师教的学生，放到老师的Map下面去。

```java
public interface ClassMapper {
    @Select("select * from teacher")
    @Results({
        @Result(property="id", column="id"),
        @Result(property="name", column="name"),
        @Result(property="students", column="id", many=@Many("getStudentsByTeacherId"))
    })
    public List<Map> getTeachersWithStudents();
    
    @Select("select * from teacher_student left join student on teacher_student.student_id=student.id where teacher_id=#{id}")
    public List<Student> getStudentsByTeacherId(int id);
}
```

具体的处理逻辑应该是，先执行一个sql语句，然后拿到结果后，根据结果遍历，去查@Many标注的sql语句（标注的是我们Mapper中的某个执行方法，需要根据具体参数去查询，这里的参数应该是column）。整体查完后，形成一个完整的数据结构返回结果。（感觉实际效果并不好，不过应该也没有更好的方案了，毕竟是关系数据库）。



这里需要再去考虑基于主键的多对多关联表数据插入： 比如新增一个学生，插入学生表的时候，需要同步插入学生-教师关系表，所以是一对多，或者是多对多的情况。这个时候，因为在学生数据的时候，会自动生成主键，然后我们拿这个主键去插入学生-教师关系表。如果按照以前的做法，是再去数据库查这个主键信息。但是JDBC3.0之后，JDBC新增了一个操作叫getGeneratedKeys，就是获取到我们插入的数据生成的主键值。Mybatis对这个进行了处理，就是在插入语句的地方，加入选项@Options的参数，useGeneratedKeys=true，然后指定主键名。这个返回的主键会放到我们传入的参数中，所以需要注意字段映射。

```java
public interface ClassMapper {
    
    @Options(useGeneratedKeys=true, keyProperty="id", keyColumn="id")
    @Insert("insert into student(name) values(#{name})")
    public void insertStudent(Student student)
}
```

