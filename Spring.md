# Spring知识图谱

## 外部属性文件

### PropertyPlaceholderConfigurer

#### 覆盖其convertProperty方法可实现加密属性值

### <context:property-placeholder location=/>

## 国际化

### Java支持

#### ResourceBundle.getBundle

#### MessageFormat.format

### MessageResource

```
@Bean
	public MessageSourceAccessor messageSourceAccessor() {
		return new MessageSourceAccessor(messageSource());
	}

	@Bean
	public MessageSource messageSource() {
		ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
		messageSource.setDefaultEncoding("UTF-8");
		messageSource.setBasenames("classpath:vitamin/messages/message");
		return messageSource;
	}
```    

#### MessageSourceAccessor

#### ApplicationContext继承MessageResource

## 容器事件

### 事件

#### ApplicationContextEvent

容器的启动、刷新、停止、关闭

#### RequestHandledEvent

HTTP请求被处理


### 监听器

#### ApplicationListener

实现 `onApplicationEvent(E event)`

### 事件源

### 监听器注册表

#### ApplicationEventMulticaster

### 事件广播器

#### ApplicationEventMulticaster

### 事件发送者

#### 实现ApplicationContextAware

#### ctx.publishEvent()

## Spring Core

### Bean

#### 生命周期

#### 作用域

##### singleton

##### prototype

每次getBean都会创建一个Bean，如果是cglib动态代理，则性能不佳


##### request

##### session

##### globalSession

##### 作用域依赖问题

prototype --> request, 动态代理

#### FactoryBean: 定制实例化Bean的逻辑

#### 配置方式

##### XML配置

##### Groovy配置

##### 注解配置

###### @Component

###### @Service

##### Java类配置

###### @Configuration

###### @Import

参考 `@EnableWebMvc`

####### 还可以@Import(ImportSelector.class)

更加灵活，可以增加条件分支，参考`@EnableCaching`

###### @Bean

#### 创建流程

##### ResourceLoader: 装载配置文件 --> Resouce

##### BeanDefinitionReader: 解析配置文件 --> BeanDefinition，并保存到BeanDefinitionRegistry

##### BeanFactoryPostProcessor: 对BeanDefinition进行加工

###### 对占位符<bean>进行解析

###### 找出实现PropertyEditor的Bean, 注册到PropertyEditorResistry

##### InstantiationStrategy: 进行Bean实例化

###### SimpleInstantiationStrategy

###### CglibSubclassingInstantiationStrategy

##### BeanWapper: 实例化时封装

###### Bean包裹器

###### 属性访问器：属性填充

###### 属性编辑器注册表

属性编辑器：将外部设置值转换为JVM内部对应类型


##### BeanPostProcessor

### DI

#### Bean Factory

##### IoC容器

##### 子类

###### DefaultListableBeanFactory

#### ApplicationContext

##### 应用上下文，Spring容器

##### 结合POJO、Configuration Metadata

##### 子类

###### ClassPathXmlApplicationContext

###### FileSystemXmlApplicationContext

###### AnnotationConfigApplicationContext

#### WebApplicationContext

#### 依赖注入

##### 属性注入

##### 构造函数注入

##### 工厂方法注入

##### 注解默认采用byType自动装配策略

#### 条件装配

##### @Profile

##### @Conditional

例： OnPropertyCondition

### AOP

#### 术语

##### JoinPoint 连接点

AOP黑客攻击的候选锚点
- 方法
- 相对位置


##### Pointcut 切点

定位到某个类的某个方法

##### Advice 增强

- AOP黑客准备的木马
- 以及方位信息: After, Before, Around


##### Target 目标对象

Advice增强逻辑的织入目标类

##### Introduction 引介

为类添加属性和方法，可继承 `DelegatingIntroductionInterceptor`

##### Weaving 织入

将Advice添加到目标类的具体连接点上的过程。
(连接切面与目标对象 创建代理的过程)

##### Aspect 切面

Aspect = Pointcut + Advice？


#### 原理

##### JDK动态代理

##### CGLib动态代理

###### 不要求实现接口

###### 不能代理final 或 private方法

###### 性能比JDK好，但是创建花费时间较长

#### 用法

##### 编程方式

###### ProxyFactory.addAdvice / addAdvisor

ProxyFactory.setTarget
ProxyFactory.addAdvice
ProxyFactory.getProxy() --> Target

```
public void addAdvice(int pos, Advice advice) {
  this.addAdvisor(pos, new DefaultPointcutAdvisor(advice));
}
```

###### 配置ProxyFactoryBean

<bean class="aop.ProxyFactoryBean"
p:target-ref="target"
p:interceptorNames="advice or adviso">
  
  

###### 自动创建代理

基于BeanPostProcessor实现，在容器实例化Bean时 自动为匹配的Bean生成代理实例。


####### BeanNameAutoProxyCreator

基于Bean配置名规则

####### DefaultAdvisorAutoProxyCreator

基于Advisor匹配机制

####### AnnotationAwareAspectJAutoPRoxyCreator

##### AspectJ

###### <aop:aspectj-autoproxy>

- 自动为匹配`@AspectJ`切面的Bean创建代理，完成切面织入。
- 底层通过 `AnnotationAwareAspectJAutoProxyCreator`实现。

## Spring MVC

### 流程

#### web.xml

##### web.xml 匹配DispatcherServlet映射路径

#### DispatcherServlet

##### request.setAttribute

###### localResolver

###### themeResolver

#### HandlerMapping

##### 通过HandlerMapping找到处理请求的Handler

#### HanlderAdapter

##### 通过HandlerAdapter封装调用Handler

##### HttpMessageConverter

###### MappingJackson2HttpMessageConverter

###### 例如构造RequestBody / ResponseBody

#### ViewResolver

##### 通过ViewResolver解析视图名到真实视图对象

##### 接口

###### AbstractCachingViewResolver

###### UrlBasedViewResolver

###### FreeMarkerViewResolver

###### ContentNegotiatingViewResolver

###### InternalResourceViewResolver

##### 源码

###### DispatcherServlet

####### initStrategies()

######## initViewResolvers()

######## 初始化所有 ViewResolver.class 

####### doDispatch()

######## applyDefaultViewName()

######## processDispatchResult()

######### 异常处理

######### render

### 自动装配

#### Spring SPI

##### 基础接口：WebApplicationInitializer

##### 编程驱动：AbstractDispatcherServletInitializer

##### 注解驱动：AbstractAnnotationConfigDispatcherServletInitializer

#### 流程

##### 1. 启动时查找ServletContainerInitializer实现类 

##### 2. 找到SpringServletContainerInitializer

##### 3.@HandlesTypes({WebApplicationInitializer.class})

### 编码

#### 入参

##### 入参种类

###### ModelMap / Map

SpringMVC在调用方法前会创建一个隐含的模型对象。如果方法入参为Map/Model，则会将隐含模型的引用传递给这些入参。


###### WebRequest

###### HttpServletRequest / HttpSession

###### @MatrixVariable

###### @CookieValue

###### @RequestHeader

###### @RequestParam

###### @RequestParam MultipartFile

####### MultipartResolver

####### 支持类型 multipart/form-data

######## consumes = MediaType.MULTIPART_FORM_DATA_VALUE

##### 入参原理

###### HttpMessageConverter

- MappingJackson2HttpMessageConverter
- ByteArrayHttpMessageConverter

###### Converter

##### 校验

###### POJO配置

####### @NotEmpty

####### @NotNull

###### 入参配置

####### @Valid

######## 校验失败返回400 + errors

####### BindingResult

######## 调用bindingResult.hasErrors() 对校验失败情况进行处理

#### 返回值

##### 缩进设置

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer c {
  return builder -> builder.indentOutput(true);
}
```

#### 异常处理

##### @ExceptionHandler

###### 添加到@Controller中

###### 或添加到@ControllerAdvice中

##### 核心接口 HandlerExceptionResolver

###### SimpleMappingExceptionResovler

###### DefaultHandlerER

###### ResponseStatusER

###### ExceptionHandlerER

#### 拦截器

##### 接口

###### 同步：HandlerInterceptor

###### 异步：AsyncHandlerInterceptor  -afterConcurrentHandlingStarted

##### 源码：DispatcherSevlet

```java
doDispatch() {
 if (!applyPreHandle())
    return;
    
 handleAdapter.handle();
 
 if (async)
	return;
 
 applyPostHandle();
 
 
 triggerAfterCompletion();
 
 finally {
	if (asyn) 
       applyAfterConcurrentHandlingStarted();
				
}
```

##### 配置方式

###### WebMvcConfigurer.addInterceptors()

###### SpringBoot: @Configuration WebMvcConfigurer

### 原理

#### WebApplicationContext

##### Servlet WebApplicationContext

WEB相关bean，继承自RootXxx

###### Controllers

###### ViewResolver

###### HandlerMapping

##### Root WebApplicationContext

middle-tier serv

###### Services

###### Repositories

##### AOP

###### 父子context的aop是独立的

###### 要想同时拦截父子：父Apspect @Bean, 子 <aop:aspectj-autoproxy/>

## Spring Data

### spring-data-jpa

#### 连接池

##### Hikari

##### c3p0

##### alibaba druid

https://github.com/alibaba/druid

- 通过Filter, 支持自定义pei

#### 事务

##### Spring事务抽象

###### PlatformTransactionManager

####### DataSourceTransactionManager

####### JpaTransactionManager

####### JmsTransactionManager

####### JtaTransactionManager

###### TransactionDefinition 

####### 传播

######## REQUIRED

######### 当前有就用当前的，没有就新建

######## SUPPORTS

######### 当前有就用当前的，没有就算了

######## MANDATORY

######### 当前有就用当前的，没有就抛异常

######## REQUIRES_NEW

######### 无论有没有，都新建

######## NOT_SUPPORTED

######### 无论有没有，都不用

######## NEVER

######### 如果有，抛异常

######## NESTED

######### 如果有，则在当前事务里再起一个事务

####### 隔离

######## READ_UNCOMMITTED

- 脏读
- 不可重复读
- 幻读

######## READ_COMMITTED

- 不可重复读
- 幻读

######## REPEATABLE_READ

- 幻读

######## SERIALIZABLE

- 

###### TransactionStatus

#### 注解

##### 实体

###### @Entity

###### @MappedSuperclass

####### 标注于父类

###### @Table

####### 表名

##### 主键

###### @Id

####### @GeneratedValue

####### @SequenceGenerator

##### 映射

###### @Column

###### @JoinTable  @JoinColumn

#### reactive

##### @EnableR2dbcRepositories

##### ReactiveCrudRepository

###### 返回值都是Mono或Flux

###### 自定义的查询需要自己写@Query

### spring-data-mongodb

#### 注解

##### @Document

##### @Id

#### mongoTemplate

##### save / remove

##### Criteria / Query / Update

##### find(Query) / findById()

##### updateFirst(query, new Update()...)

#### MongoRepository

##### @EnableMongoRepositories

### spring-data-redis

#### Jedis

##### JedisPool

###### 基于Apache Common Pool

###### JedisPoolConfig

###### JedisPool.getResource

####### internalPool.borrowObject()

##### JedisSentinelPool

###### 监控、通知、自动故障转移、服务发现

###### MasterListener

##### JedisCluster

###### 配置

####### JedisSlotBasedConnectionHandler

######## getConnection: 随机

######## getConnectionFromSlot: 基于slot选择

####### JedisClusterInfoCache

###### 技巧

####### 单例：内置了所有节点的连接池

####### 无需手动借还连接池

####### 合理设置commons-pool

###### 不支持读slave

#### Lettuce

##### 支持读写分离

###### 只读主、只读从

###### 优先读主、优先读从

###### LettuceClientConfigurationBuilderCustomizer -> readFrom(ReadFrom.MASTER_PREFERRED)

#### RedisTemplate

#### Repository

##### @EnableRedisRepository

##### @RedisHash

###### Class级别：@RedisHash(value = "springbucks-coffee", timeToLive = 60)

##### @Id

##### @Index

###### 二级索引，自动创建另一套key-value

#### Converter

##### @WritingConverter

##### @ReadingConverter

##### byte[]与对象互转

### reactor

```java
Flux.fromIterable(list)
  .publishOn(Schedulers.single())
  .doOnComplete(..)
  .flatMap(..)
  .concatWith()
  .onErrorResume(..)
  .subscribe()..;
```

#### Operators

##### publisher / subscriber

##### onNext() / onComplete() / onError()

##### Flux[0..N] / Mono[0..1]

#### Backpressure

##### Subscription

##### onRequest() / onCancel() / onDispose()

#### Schedulers

##### immediate() / single() / newSingle()

##### elastic() / parallel() / newParallel()

#### 错误处理

##### onError / onErrorReturn / onErrorResume

##### doOnError / doFinally

#### 配置

##### ReactiveRedisConnection / ReactiveRedisTemplate

##### ReactiveMongoDatabaseFactory / ReactiveMongoTemplate

```java
mongoTemplate.insertAll(list)
  .publishOn(Schedulers.elastic())
  .doOnNext(log..)
  .doOnComplete(..)
  .doOnFinally(..)
  .count()
  .subscribe();
```

## Spring Boot

### 六大特性

#### 创建独立的Spring应用

##### 运行方式

###### mvn spring-boot:run

前提：
```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
</parent>  
  
```


###### 可执行 JAR

前提：
```xml
<build>
  <plugins><plugin>
    <artifactId>spring-boot-maven-plugin</artifactId> 
```


####### 原理

######## JarLauncher

######## MANIFEST.MF

######### Main-Class = JarLauncher

######### Start-Class

#### 嵌入Web容器

##### Tomcat

###### Servlet/Reactive：TomcatWebServer Bean

##### Jetty

###### Servlet/Reactive：JettyWebServer Bean

##### Undertow

###### Servlet：UndertowServletWebServer Bean

###### Reactive：UndertowWebServer Bean

#### 提供固化的starter依赖，简化构建配置

##### 两种方式

###### spring-boot-starter-parent

####### <parent>

####### 缺点：单继承

###### spring-boot-dependencies

####### <dependency>

#### 自动装配

##### 作用

###### 根据所依赖的jar，尝试自动配置spring application

##### 实现

###### 1.@EnableAutoConfiguration

或 @SpringBootApplication == 
- @ComponentScan
- @EnableAutoConfiguration
- @SpringBootConfiguration -> @Configuration

###### 2. 自定义XXAutoConfiguration

####### 条件判断 @Conditional

####### 模式注解 @Configuration

####### @Enable模块：@EnableXX -> *ImportSelector -> *Configuration

###### 3.配置spring.factories (SpringFactoriesLoader)

#### 提供运维特性

#### 无需代码生成

### 模式注解

#### 派生性

#### 层次性

### 源码

#### SpringApplication

##### 准备阶段

###### 配置 Spring Boot Bean 源		

###### 推断Web应用类型

根据classpath

###### 推断引导类

根据 Main 线程执行堆栈判断实际的引导类

###### 加载ApplicationContextInitializer

spring.factorie

###### 加载ApplicationListener

spring.factories
例如`ConfigFileApplicationListener`

##### 运行阶段

###### 加载监听器 SpringApplicationRunListeners

spring.factories
getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args))

`EventPublishingRunListener` 
--> `SimpleApplicationEventMulticaster`

####### EventPublishingRunListener

####### SimpleApplicationEventMulticaster

###### 运行监听器 SpringApplicationRunListeners

listeners.starting();

###### 创建应用上下文 ConfigurableApplicationContext

createApplicationContext()
- NONE: `AnnotationConfigApplicationContext`
- SERVLET: `AnnotationConfigServletWebServerApplicationContext`
- REACTIVE: `AnnotationConfigReactiveWebServerApplicationContext` 

###### 创建Environment

getOrCreateEnvironment()
- SERVLET: `StandardServletEnvironment`
- NONE, REACTIVE: `StandardEnvironment` 

### 运维

#### Actuator

##### 解禁Endpoints

###### management.endpoints.web.exposure.include=*

###### 生产环境谨慎使用

## Spring Cache

### 标注

#### @EnableCaching

#### @Cacheable

#### @CacheEvict

#### @CachePut

#### @Caching

#### @CacheConfig(cacheNames = "")

## STOMP
