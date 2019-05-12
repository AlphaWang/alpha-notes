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
