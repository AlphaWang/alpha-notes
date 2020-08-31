# Zuul

## 网关

### 功能

#### 单点入口

#### 路由转发

#### 限流熔断

#### 日志监控

#### 安全认证

## 架构

### Filter Publisher

#### 存储到DB

### Filter Loader

#### 扫描DB，存到本地文件

### Filter Runner

#### 类型

##### Pre routing filter

##### Routing filter

##### Post routing filter

#### Request Context很关键

## 过滤器

### Type

#### PRE

##### 认证

##### 选路由

##### 请求日志

#### ROUTING

#### POST

##### 增加响应头

##### 收集统计和度量

##### 将响应以流的方式发送回客户端

#### ERROR

### Order

### Criteria

### Action

## 集成Apollo

### 场景

#### 动态设置zuul参数

##### zuul.could.set.debug

##### zuul.routing_table_string

### 原理

#### zuul已经集成netflix Archaius客户端

#### Archaius可从Apollo拉数据

### 配置

#### InitializeServletListener

System.setProperty(Constants.DEPLOY_CONFIG_URL, "");

Constants.DEPLOY_CONFIG_URL
Constants.DEPLOY_ENVIRONMENT
Constants.DEPLOY_APPLICATION_ID


#### com.netflix.config.ConfigurationManager

## 部署

### 案例

#### 案例

### 集成

#### 集成

## 路由管理

### 基于Eureka自发现，+Ribbon

### 基于域名，+服务治理中心（类似api-forge）

### 基于Apollo配置

## Zuul 2

### 非阻塞异步模式

#### 优势

##### 线程开销少

##### 连接数易扩展

#### 劣势

##### 编程模型复杂

##### 开发调试运维复杂

##### ThreadLocal不能用

#### 适用于IO密集型场景

### 亮点

#### 服务器协议

##### HTTP2

##### Mutal TLS

#### 弹性

##### Adaptive Retries 自适应重试

##### Origin Concurrency Protection 源并发保护

#### 运维

##### Request Passport

##### Status Categories

##### Request Attempts

## 最佳实践

### 使用异步AsyncServlet 优化连接数

### 使用Apollo配置中心集成动态配置

### 使用Hystrix熔断限流、信号量隔离

### 连接池管理

### 监控：CAT、Hystrix
