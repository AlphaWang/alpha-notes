# Java架构知识图谱

## NoSql

### Memcached

#### memcached是lazy clean up, 那么如何保证内存不被占满？

### Redis

#### Redis知识图谱

http://naotu.baidu.com/file/3200a19ccc62cf25a318cdf75def4211

### MongoDB

#### 常用命令

##### show dbs / collections

##### use db1

##### db.collection1.find();

## MQ

### Kafka

#### 集群

##### 集群成员：对等，没有中心主节点

#### CA

### RabbitMQ

## 分布式

### ZooKeeper

#### 特性

##### 顺序一致性

对同一个客户端来说

##### 原子性

所有事务请求的处理结果在集群中所有机器上的应用情况是一致的。


##### 单一视图

客户端无论连接哪个zk服务器，看到的数据模型都是一致的


##### 可靠性

##### 实时性

#### CP (ZAB协议保证一致性)

Zookeeper Atomic Broadcast.
- 所有事务请求必须由全局唯一的Leader服务器来协调处理。
- Leader将客户端的事务请求转换成一个`事务Proposal`，将其分发给所有Follower，并等待Follower的反馈.
- 一旦超过半数的Follower进行了正确的反馈，则Leader再次向所有Follower分发`commit消息`，要求其将前一个Proposal进行提交


##### 单一主进程

- 使用单一的主进程来接收并处理所有事务请求（事务==写），
- 并采用ZAB原子广播协议，将服务器数据的状态变更以事务Proposal的形式广播到所有副本进程。

###### 单一的主进程来接收并处理所有事务请求

###### 对每个事务分配全局唯一的ZXID

####### zxid低32位：单调递增计数器

对每个事务请求，生成Proposal时，将计数器+1


####### zxid高32位：Leader周期的epoch编号

###### 数据的状态变更以事务Proposal的形式广播到所有副本进程

##### 顺序应用

必须能保证一个全局的变更序列被顺序应用，从而能处理依赖关系

##### 1. (发现)崩溃恢复模式：选举要保证新选出的Leader拥有最大的ZXID

崩溃恢复：Leader服务器出现宕机，或者因为网络原因导致Leader服务器失去了与过半 Follower的联系，那么就会进入崩溃恢复模式从而选举新的Leader。
当选举产生了新的Leader，同时集群中有过半的服务器与该Leader完成了状态同步（即数据同步）之后，Zab协议就会退出崩溃恢复模式，进入消息广播模式。

######  

则可保证新Leader一定具有所有已提交的Proposal；
同时保证丢弃已经被跳过的Proposal

##### 2. (同步) 检查是否完成数据同步

###### 对需要提交的，重新放入Proposal+Commit

###### 对于Follower上尚未提交的Proposal，回退

###### 同步阶段的引入，能有效保证Leader在新的周期中提出Proposal之前，
所有的进程都已经完成了对之前所有Proposal的提交。

##### 3. (广播) 消息广播模式：Proposal (ZXID), ACK, Commit

消息广播：所有的事务请求都会转发给Leader，Leader会为事务生成对应的Proposal，并为其分配一个全局单调递增的唯一ZXID。
当Leader接受到半数以上的Follower发送的ACK投票，它将发送Commit给所有Follower通知其对事务进行提交，Leader本身也会提交事务，并返回给处理成功给对应的客户端。

###### 1.Leader为事务生成对应的Proposal，分配ZXID

必须将每一个事务Proposal按照其ZXID的先后顺序来进行排序与处理。

###### 2.半数以上的Follower回复ACK投票

###### 3.发送Commit给所有Follower通知其对事务进行提交

###### 4.返回给处理成功给对应的客户端

###### 类似一个2PC提交，移除了中断逻辑

#### 原理

##### version保证原子性

###### version表示修改次数

###### 乐观锁

##### watcher

###### 特性

####### 一次性

####### 客户端串行执行

####### 轻量

######## 推拉结合

######## 注册watcher时只传输ServerCnxn

###### 流程

####### 客户端注册Watcher

####### 客户端将Watcher对象存到WatchManager: Map<String, Set<Watcher>>

####### 服务端存储ServerCnxn

######## watchTable: Map<String, Set<Watcher>>

######## watch2Paths: Map<Watch, Set<String>>

####### 服务端触发通知

######## 1.封装WatchedEvent

######## 2.查询并删除Watcher

######## 3.process: send response (header = -1)

####### 客户端执行回调

######## 1.SendThread接收通知， 放入EventThread

NIO

######## 2.查询并删除Watcher

######## 3.process: 执行回调

###### WatchedEvent

网络传输时序列化为 `WatcherEvent`

####### KeeperState

- SyncConnected
- Disconnected
- Expired
- AuthFailed


####### EventType

- None
- NodeCreated
- NodeDeleted
- NodeDataChanged
- NodeChildrenChanged

###### Curator 如何解决一次性watcher问题？

##### ACL

###### Scheme

####### IP:192.168.1.1:permission

####### Digest:username:sha:permission

####### World:anyone:permission

####### Super:username:sha:permission

###### Permission

####### C, Create

####### D, Delete

####### R, Read

####### W, Write

####### A, Admin

###### 权限扩展体系

####### 实现AuthenticationProvider

####### 注册

######## 系统属性 -Dzookeeper.authProvider.x=

######## zoo.cfg: authProvider.x=

##### 客户端

###### 通讯协议

####### 请求

######## RequestHeader

######### xid

记录客户端发起请求的先后顺序

######### type

- 1: OpCode.Create
- 2: delete
- 4: getData


######## Request

####### 响应

######## ReplyHeader

######### xid

原值返回


######### zxid

服务器上的最新事务ID


######### err

######## Response

###### ClientCnxn：网络IO

####### outgoingQueue

待发送的请求Packet队列

####### pendingQueue

已发送的、等待服务端响应的Packetdui'lie

####### SendThread: IO线程

####### EventThread: 事件线程

######## waitingEvents队列

##### Session

###### SessionID: 服务器myid + 时间戳

###### SessionTracker: 服务器的会话管理器

####### 内存数据结构

######## sessionById:     HashMap<Long, SessionImpl>

######## sessionWithTimeout: ConcurrentHashMap<Long, Integer>

######## sessionSets:     HashMap<Long, SessionSet>超时时间分桶

####### 分桶策略

- 将类似的会话放在同一区块进行管理。
- 按照“下次超时时间”
- 好处：清理时可批量处理

####### 会话激活

- 心跳检测
- 重新计算下一次超时时间
- 迁移到新区块


######## 客户端发送任何请求时

######## sessionTimeout / 3时，发送PING请求

####### 超时检测：独立线程，逐个清理

####### 会话清理

######## 1. isClosing设为true

######## 2.发起“会话关闭”请求

######## 3.收集需要清理的临时节点

######## 4.添加“节点删除”事务变更

######## 5.删除临时节点

######## 6.移除会话、关闭NIOServerCnxn

####### 重连

######## 连接断开

- 断开后，客户端收到None-Disconnected通知，并抛出异常`ConnectionLossException`；
- 应用需要捕获异常，并等待客户端自动完成重连；
- 客户端自动重连后，收到None-SyncConnected通知

######## 会话失效

- 自动重连时 超过了会话超时时间。
- 应用需要重新实例化ZooKeeper对象，重新恢复临时数据

######## 会话转移

- 服务端收到请求时，检查Owner 如果不是当前服务器则抛出`SessionMovedExceptio`

#### 角色

##### Leader: 读写

##### Follower: 读。参与Leader选举、过半写成功

##### Observer: 读

#### 应用

##### 配置中心：数据发布订阅

推拉结合的方式：
- 推：节点数据发生变化后，发送Watcher事件通知给客户端。
- 拉：客户端收到通知后，需要主动到服务端获取最新的数据


##### 负载均衡：域名注册、发现

##### 命名服务：全局ID生成器（顺序节点）

##### 分布式协调、通知：任务注册、任务状态记录

##### 集群管理：分布式日志收集系统、云主机管理

日志收集系统要解决：
1. 变化的日志源机器
2. 变化的收集器机器

- 注册日志收集器，非临时节点：`/logs/collectors/[host]`
- 节点值为日志源机器。
- 创建子节点，记录状态信息 `/logs/collectors/[host]/status`
- 系统监听collectors节点，当有新收集器加入，或有收集器停止汇报，则要将之前分配给该收集器的任务进行转移。

##### Master选举：

利用zk强一致性，保证客户端无法重复创建已存在的节点

##### 分布式锁

###### 排他锁

排他锁，X锁（Exclusive Locks），写锁，独占锁：
- 同时只允许一个事务，其他任何事务不能进行任何类型的操作。
- 创建临时节点


###### 共享锁

共享锁，S锁（Shared Locks），读锁：
- 加共享锁后当前事务只能进行读操作；其他事务也只能加共享锁。
- `W`操作必须在当前没有任务事务进行读写操作时才能进行。
- 创建临时顺序节点 `/s_locks/[hostname]-请求类型-序号`。
- 节点上表明是`R`还是`W`。
- 如果是`R`，且比自己序号小的节点都是`R`，则加锁成功；
- 如果是`W`，如果自己是最小节点，则加锁成功。

优化：
R节点只需要监听比他小的最后一个W节点；
W节点只需要监听比他小的最后一个节点。

##### 分布式队列

###### FIFO

- 注册临时顺序节点；
- 监听比自己小的最后一个节点。

###### Barrier

- 父节点`/queue_barrier`，值为需要等待的节点数目N。
- 监听其子节点数目；
- 统计子节点数目，如果数目小于N，则等待

##### 实例

###### Hadoop

####### ResourceManager HA

多个ResourceManager并存，但只有一个处于Active状态。
- 有父节点 `yarn-leader-election/pseudo-yarn-rm-cluster`, RM启动时会去竞争Lock**临时**子节点。
- 只有一个RM能竞争到，其他RM注册Wather


####### ResourceManager 状态存储

Active状态的RM会在初始化阶段读取 `/rmstore` 上的状态信息，并据此信息继续进行相应的chu'li

###### HBase

####### RegionServer系统容错

####### RootRegion管理

###### Kafka

####### Broker注册: /broker/ids/[brokerId]

`/broker/ids/[BrodkerId]` (临时节点)

####### Topic注册: /brokers/topics/[topic]

每个topic对应一个节点`/brokers/topics/[topic]`；
Broker启动后，会到对应Topic节点下注册自己的ID **临时节点**，并写入Topic的分区总数，`/brokers/topics/[topic]/[BrokerId] --> 2`


####### Producer负载均衡

Producer会监听`Broker的新增与减少`、`Topic的新增与减少`、`Broker与Topic关联关系的变化`

####### Consumer负载均衡:  /consumers/[groupId]/owners/[topic]/[brokerId-partitionId]

当消费者确定了对一个分区的消费权利，则将其ConsumerId写入到分区**临时节点**上：
`/consumers/[groupId]/owners/[topic]/[brokerId-partitionId] --> consumerId`

####### Consumer注册: /consumers/[groupId]/ids/[consumerId]

- 消费者启动后，注册**临时节点** `/consumers/[groupId]/ids/[consumerI]`。并将自己订阅的Topic信息写入该节点 
- 每个消费者都会监听ids子节点。
- 每个消费者都会监听Broker节点：`/brokers/ids/`

####### 消费进度offset记录: /consumers/[groupId]/offsets/[topic]/[brokerId-partitionId]

消费者重启或是其他消费者重新接管消息分区的消息消费后，能够从之前的进度开始继续进行消费：`/consumers/[groupId]/offsets/[topic]/[brokerId-partitionId] --> offset`

### 缓存

#### 缓存更新策略

##### LRU/LFU/FIFO 算法删除

###### maxmemory-policy

##### 超时剔除

###### expire

##### 主动更新

#### 缓存粒度控制

##### 缓存全量属性

###### 通用性好、可维护性好

##### 缓存部分属性

###### 占用空间小

#### 缓存穿透问题

##### 原因

###### 业务代码自身问题

###### 恶意攻击、爬虫

##### 发现

###### 业务响应时间

###### 监控指标：总调用数、缓存命中数、存储层命中数

##### 解决

###### 缓存空对象

###### 布隆过滤器拦截？

#### 缓存雪崩问题

##### 问题

###### cache服务器异常，流量直接压向后端db或api，造成级联故障

##### 优化

###### 保证缓存高可用

####### redis sentinel

####### redis cluster

####### 主从漂移：VIP + keepalived

###### 依赖隔离组件为后端限流（降级）

###### 提前演练

#### 无底洞问题

##### 问题

###### 加机器性能不升反降

###### 客户端批量接口需求（mget, mset）

###### 后端数据增长与水平扩展需求

##### 优化

###### 命令本身优化

####### 慢查询keys

####### hgetall bigkey

###### 减小网络通信次数

####### 串行mget --> 串行IO --> 并行IO 

####### hash_tag

###### 降低接入成本

####### 客户端长连接、连接池

####### NIO

### 分布式事务

#### 事务

##### ACID 特性

###### A 原子性

####### 要么转账成功，要么转账失败

###### C 一致性

####### 总钱数不会变

###### I 隔离性

####### A转账、B查余额，依赖于隔离级别

###### D 持久性

####### 重启后不变

##### 隔离级别

###### Read Uncommitted

###### Read Committed

####### 查询只承认在`语句`启动前就已经提交完成的数据

####### 解决：脏读

###### Repeatable Read

####### 查询只承认在`事务`启动前就已经提交完成的数据

####### 解决：脏读、不可重复读

###### Serialized

####### 对相关记录加读写锁

####### 解决：脏读、不可重复读、幻读

##### 传播

###### REQUIRED: 当前有就用当前的，没有就新建

###### SUPPORTS: 当前有就用当前的，没有就算了

###### MANDATORY: 当前有就用当前的，没有就抛异常

###### REQUIRES_NEW: 无论有没有，都新建

###### NOT_SUPPORTED: 无论有没有，都不用

###### NEVER: 如果有，抛异常

###### NESTED: 如果有，则在当前事务里再起一个事务

#### Spring分布式事务实现

##### 种类

###### XA与最后资源博弈

####### 两阶段提交

1. start MQ tran
2. receive msg
3. start JTA tran on DB
4. update DB
5. Phase-1 commit on DB tran
6. commit MQ tran
7. Phase-2 commit on DB tran

###### 共享资源

####### 原理

######## 两个数据源共享同一个底层资源

######## 例如ActiveMQ使用DB作为存储

######## 使用DB上的connection控制事务提交

####### 要求

######## 需要数据源支持

###### 最大努力一次提交

####### 原理

######## 依次提交事务

######## 可能出错

######## 通过AOP或Listener实现事务直接的同步

####### 例：JMS最大努力一次提交+重试

1. start MQ tran
2. receive msg
3. start DB tran
4. update DB
5. commit DB tran
6. commit MQ tran

Step4 数据库操作出错，消息会被放回MQ，重新触发该方法；
Step6 提交MQ事务出错，消息会被放回MQ，重新触发该方法；此时会重复数据库操作，需要忽略重复消息；

######## 适用于其中一个数据源是MQ，并且事务由读MQ消息开始

######## 利用MQ消息的重试机制

######## 重试时需要考虑重复消息

###### 链式事务

####### 原理

######## 定义一个事务链

######## 多个事务在一个事务管理器里依次提交

######## 可能出错

第二个提交执行中 如果数据库连接失败，第一个提交无法回滚。


####### 实现

######## ChainedTransactionManager

######### DB + DB

```java
@Bean
public PlatformTransactionManger trxManager() {
  DataSourceTransactionManager userTM = new DataSourceTransactionManager(userDataSource());
  DataSourceTransactionManager orderTM = ...
  
  // 顺序敏感，在前的后提交
  ChainedTransactionManager tm = new ChianedTransactionManager(orderTM, userTM);
}
```

######### JPA + DB

JPA factory:
```java
@Bean
public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
  HibernateJpaVendorAdapter va = new ...;
  LocalContainerEntityManagerFactoryBean factory = new ...;
  factory.setJpaVendorAdapter(va);
  factory.setDataSource(userDataSrouce());
  return factory;
}

```

TRX Manger:
```java
@Bean
public PlatformTransactionManger trxManager() {
  JpaTransactionManager userTM = new ...;
  userTM.setEntityManagerFactory(entityManagerFactory().getObject());
  
  DataSourceTransactionManager orderTM = new DataSourceTransactionManager(orderDataSource())
  
  // 顺序敏感，在前的后提交
  ChainedTransactionManager tm = new ChianedTransactionManager(orderTM, userTM);
}
```

##### 选择

###### 强一致性

####### JTA

####### （性能差，只适用于单个服务内）

###### 弱、最终一致性

####### 最大努力一次提交、链式事务

####### （设计相应错误处理机制）

###### MQ-DB

####### 最大努力一次提交 + 重试

###### DB-DB

####### 链式事务

###### 多个数据源

####### 链式事务、或其他事务同步方式

### 定理

#### CAP定理

##### 含义

###### Consistency 一致性

####### 数据在多个副本之间能够保持一致

###### Availability 可用性

####### 系统提供的服务必须一直处于可用的状态

###### Partition Tolerance 分区容错性

####### 出现网络分区错误时，系统也需要能容忍

##### 示例

###### CP

####### BigTable

####### Hbase

####### MongoDB

####### Redis

###### AP

####### DynamoDB

####### Cassandra

####### Eureka

###### CA

####### Kafka

####### zookeeper

## 微服务

### Spring Cloud

### service mesh

#### consumer端 sidecar

##### 服务发现

##### 负载均衡

##### 熔断降级

#### provider端 sidecar

##### 服务注册

##### 限流降级

##### 监控上报

#### Control Plane

##### 服务注册中心

##### 负载均衡配置

##### 请求路由规则

##### 配额控制

### 高可用

#### 降级

##### hystrix fallback 原理

#### 熔断

#### 限流

### 常用框架工具

#### 配置中心

##### 概念、功能

###### 定义

####### 可独立于程序的可配变量

####### 例如连接字符串、应用配置、业务配置

###### 形态

####### 程序hard code

####### 配置文件

####### 环境变量

####### 启动参数

####### 基于数据库

###### 治理

####### 权限控制、审计

####### 不同环境、集群配置管理

####### 框架类组件配置管理

####### 灰度发布

###### 分类

####### 静态配置

######## 环境相关

######### 数据库/中间件/其他服务的连接字符串

######## 安全配置

######### 用户名，密码，令牌，许可证书

####### 动态配置

######## 应用配置

######### 请求超时，线程池、队列、缓存、数据库连接池容量，日志级别，限流熔断阈值，黑白名单

######## 功能开关

######### 蓝绿发布，灰度开关，降级开关，HA高可用开关，DB迁移

######### 开关驱动开发（Feature Flag Driven Development）

######### TBD, Trunk Based Development

########## 新功能代码隐藏在功能开关后面

######## 业务配置

######### 促销规则，贷款额度、利率

##### 框架

###### Ctrip Apollo

https://github.com/ctripcorp/apollo

###### Ali Diamond

###### Netflix Archaius

https://github.com/Netflix/archaius

###### Facebook Gatekeeper

###### Baidu Disconf

https://github.com/knightliao/disconf

###### Spring Cloud Config

#### 网关

##### 功能

###### 单点入口

###### 路由转发

###### 限流熔断

###### 日志监控

###### 安全认证

##### 框架

###### zuul

 https://github.com/Netflix/zuul

###### kong

https://github.com/Kong/kong

###### tyk

https://github.com/TykTechnologies/tyk

#### 链路跟踪

## 设计模式

### SOLID

#### Single Responsibility

##### 程序中的类或方法只能有一个改变的理由

#### Open-Closed

##### 对扩展开放，对修改关闭

#### Liskov substitution

#### Interface Segregation

#### Dependency inversion

##### 抽象不应该依赖细节，细节应该依赖抽象

### 创建型

#### Factory

##### 案例

###### Spring BeanFactory/ApplicationContext

#### Abstract Factory

#### Singleton

##### 双重检查锁定

##### 内部类持有静态对象：利用对象初始化锁

##### 案例

###### Spring Bean单例

#### Builder

##### 案例

###### HttpRequest构造

#### Prototype

### 结构型

#### Bridge

#### Adapter

#### Decorator

##### 案例

###### Java IO: FileInputStream/ByteArrayInputStream  -> InputStream

#### Proxy

#### Composite

#### Facade

#### Flyweight

### 行为型

#### Strategy

##### lambda简化

###### 可去除策略子类

#### Interpreter

#### Command

##### lambda简化

###### 可去除命令子类

###### 命令类是 FunctionalInterface，可通过lambda/方法引用传入；这样就能去除所有命令子类

#### Observer

##### 案例

###### 事件监听器

##### lambda简化

###### 可去除Observer子类

#### Iterator

#### Template Method

##### 定义

###### 有一个通用算法，步骤上略有不同

##### 案例

###### JdbcTemplate

#### Visitor

## 运维

### SLA

#### 含义

##### Service Level Agreement

##### 服务等级协议

#### SLA 指标

##### 可用性 Availability

###### 3 个 9

####### 一天服务间断期：86秒

###### 4个9

####### 一天服务间断期：8.6秒

##### 准确性 Accuracy

###### 用错误率衡量

###### 评估

####### 性能测试

####### 查看系统日志

##### 系统容量 Capacity

###### QPS / RPS

###### 如何给出

####### 限流

####### 性能测试

######## 注意缓存

####### 分析日志

##### 延迟 Latency

###### Percentile

#### 扩展指标

##### 可扩展性 Scalability

###### 水平扩展

####### 可提高可用性

###### 垂直扩展

##### 一致性 Consistency

###### 强一致性

####### 牺牲延迟性

####### 例如Google Cloud Spanner

###### 弱一致性

###### 最终一致性

##### 持久性 Durability

###### 数据复制
