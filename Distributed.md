

## 分布式理论

### CAP 定理

分区不可避免，C/A要进行取舍

#### Consistency 一致性

- 数据在多个副本之间能够保持一致

- 强调数据正确

  - 每次读取都能读取到最新写入的数据

  - 但如果数据不一致，则无法访问

#### Availability 可用性

- 系统提供的服务必须一直处于可用的状态

- 强调可用性

  - 每次读取都能得到响应数据

  - 但不保证数据时正确的

#### Partition Tolerance 分区容错性

- 出现网络分区错误时，系统也需要能容忍

- 不可用方法

  - 1. 激进剔除

       一旦发现节点不可达，则剔除，并选出新主

       问题：导致双主

  - 2. 保守停止

       一旦发现节点不可达，则停止自己的服务

       问题： 整个系统不可用

########## 整个系统不可用

- 解决方法

  - 1. **Static Quorum**

       - 固定票数

         大于固定票数的分区为活动分区

         固定票数 <= 总节点数 <= 2*固定票数 - 1

       - 问题

         分区多时，不容易找到符合条件的分区

         票数固定，不适用于动态加入节点

  - 2. **Keep Majority**

       - 保留多数节点的子集群

         > 偶数时如何解决？
         >
         > 叠加策略：保留节点ID最小的

       - 问题

         分区多时，不容易找到符合条件的分区

  - 3. **仲裁机制**

    - 仲裁者为第三方组件

      选主：拥有全局心跳信息，据此判断有多个少分区、保留那个子集群

    - 问题：仲裁者的可靠性

  - 4. **共享资源方式**

########## 仲裁者的可靠性

​				- 分布式锁：哪个子集群获得锁，就保留谁

​				- 问题：如果获得锁后发生故障，但未释放锁？

#### CA 场景

单机系统

**含义**

- 关注一致性、可用性

- 需要全体一致性协议：例如 2PC

- 不能容忍网络错误：此时整个系统会拒绝写请求，变成只读

**例子**

- Kafka  --> CP?

- zookeeper --> CP?

  在选出新leader之前，不对外提供服务，岂不是不保证A?

- 单机版 MySql

#### CP 场景

例如金钱交易

**含义**

- 关注一致性、分区容忍性

-  需要多数人一致性协议：例如Paxos

- 保证大多数节点数据一致，少数节点在没有同步到最新数据时会变成不可用状态。
  在等待期间系统不可用

**例子**

- Hbase

- Etcd

- Consul

- BigTable

- MongoDB

- Redis --> AP?
-  Kafka 
  - Unclean 领导者选举，会导致 CP --> AP

#### AP 场景

最终一致性

####### 放弃强一致性

**含义**

- 关心可用性、分区容忍性
- 这样的系统不可能达成一致性

**例子**

- DynamoDB

- Cassandra
- Eureka



### BASE 理论

#### Basiccally Available 基本可用

分布式系统出现故障的时候，允许损失部分可用性。

例如：Latency 损失、降级。

#### Soft State 软状态

允许系统存在中间状态，而该中间状态不影响系统整体可用性。

例如：副本同步的延时。

#### Eventual Consistency 最终一致性

系统中所有数据副本经过一定时间后，最终能够达到一致的状态

变种：

- 因果一致性 Causal Consistency

- 读己之所写 Read Your Writes

- 会话一致性 Session Consistency

- 单调读一致性 Monotonic read consistency

- 单调写一致性 Monotonic write consistency

### 挑战

#### 通讯异常

- 内存访问 10ns，网络访问 0.1-1ms

- 延迟 100 倍

#### 网络分区

- 脑裂

- 出现局部小集群

#### 三态

- 成功

- 失败

- 超时：发送时丢失、响应丢失

#### 节点故障





### 分布式中间件

#### NoSql

##### Memcached

memcached是lazy clean up, 那么如何保证内存不被占满？

**内存机制：Slab**

####### Slab

-  解决了内存碎片问题
- 但无法有效利用内存

**vs. Reids**

- 数据类型

- 持久化

- 分布式

- 内存管理 slot



##### Redis

Redis知识图谱 http://naotu.baidu.com/file/3200a19ccc62cf25a318cdf75def4211

##### MongoDB

常用命令

- show dbs / collections

- use db1

- db.collection1.find();



#### MQ

##### RabbitMQ

特点

- 轻量级

- Exchange：处于Producer、Queue之间，路由规则灵活

问题

- 对消息堆积不友好。会导致性能急剧下降

- 性能不佳

- Erlang语言小众



##### RocketMQ

特点

- 时延小

- Broker事务反查：支持事务消息

  > Broker等不到produer的commit，则访问生产者接口，回查事务状态，决定是提交还是回滚

问题

- 与周边生态系统的集成和兼容不佳

##### Kafka

特点

- 集群。集群成员：对等，没有中心主节点
- 与周边生态系统集成好
- 性能好
- CA

问题

- 异步收发消息时延小，但同步时延高

- 批量发送，数据量小时反而时延高

##### Pulsar

特点

- 存储与计算分离

  - ZK：存储元数据

  - BookKeeper：存储 Ledger - Write Ahead Log

  - Broker

    - *Load Balancer*

      哪些 Broker 管理哪些 分区；

      Brokder无状态，分区与Broker对应关系是动态的

    - *Managed Ledger*

      创建 Ledger，一次性写入限制；

      只有这个Broker对这个Ledger有写权限，资源不共享，避免加锁；

    - *Cache*

      缓存一部分 Ledger

- Broker 天然支持水平扩展，故障转移更简单快速；

- 分离后，计算节点开发者、存储系统开发者 各自更专注，开发难度降；

问题

- BookKeeper 依然要解决数据一致性、故障转移、选举、复制等问题

- 性能有损失：多一次请求BookKeeper



#### ZooKeeper

See ref.



## 分布式技术

### 分布式协同

#### 分布式互斥/锁

作用

- 排他性的资源访问

  Distributed Mutual Exclusion

  Critical Resource 临界资源

场景

- 订单 消息处理

  订单ID是共享资源，处理时要加锁；防止重复消息

  但锁并不保证幂等，需要业务保证 （例如处理之前查询订单状态）

##### 算法

###### 1. 集中式算法

- 每个程序在需要访问临界资源时，先给协调者发送一个请求。如果当前没有程序使用这个资源，协调者直接授权请求程序访问；否则，按照先来后到的顺序为请求程序“排一个号”。如果有程序使用完资源，则通知协调者，协调者从“排号”的队列里取出排在最前面的请求，并给它发送授权消息。拿到授权消息的程序，可以直接去访问临界资源。
- 简单、容易实现
- 引入协调者；可用性、性能受协调者影响

###### 2. 分布式算法

- 当一个程序要访问临界资源时，先向系统中的其他程序发送一条请求消息，在接收到所有程序返回的同意消息后，才可以访问临界资源。

- 消息数量指数增加；可用性低

  > 消息要发给所有节点；一个节点挂了 就不可用；
  >
  > --> 改进：检测到节点故障则忽略它。

- 适合节点少，且变动不频繁的系统：Hadoop 修改 HDFS 文件

###### 3. 令牌环算法

- 所有程序构成一个环结构，令牌按照顺时针（或逆时针）方向在程序之间传递，收到令牌的程序有权访问临界资源，访问完成后将令牌传送到下一个程序；若该程序不需要访问临界资源，则直接把令牌传送给下一个程序。
- 通信效率高，公平；但也有无效通信。
- 适用于规模较小，每个程序使用临界资源的频率高，且用时短的场景。

##### 实现方式

###### 1. 数据库

1. 唯一索引
2. for update: `select id from order where order_no= 'xxxx' for update`

原理

- 加锁：增加一条记录；

- 放锁：删除

- 通过唯一性约束保证互斥

问题

- 单点故障

- 死锁：若记录一直删不掉？



###### 2. Redis

性能最好

原理

- SETNX + Expire

```
public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {

    Long result = jedis.setnx(lockKey, requestId);// 设置锁
    if (result == 1) {// 获取锁成功
        // 若在这里程序突然崩溃，则无法设置过期时间，将发生死锁
        jedis.expire(lockKey, expireTime);// 通过过期时间删除锁
        return true;
    }
    return false;
}

```

25.12之后：

```
String result = jedis.set(
  key, 
  requestId,
  "NX",
  "PX", expireTi);
```



问题

- 无法续租：执行业务时间可能超过过期时间

- AP
  - 集群同步是异步的，Master获取锁后，在同步之前崩溃了；新master是不认识这个锁的
  - 而若用单实例，则可能阻塞业务流程

Redisson RedLock算法 (?)

- 节点超半数

- CP ?



###### 3. ZooKeeper

首选！

原理

- 临时顺序节点 + Watch前一节点
  - 避免羊群效应

- 最小节点获得锁

- Paxos --> ZAB 协议

问题

- 频繁创建删除节点，性能不及redis

- 如何实现续租 (?)



###### 4. Chubby

设计亮点：

- 客户端缓存
  - 作用： 提升性能

########## 提升性能

- 缓存一致性

  - 租期

    - 修改 Master 元数据时：

      > 先向所有客户端通过KeepAlive响应发送缓存过期信号；
      >
      > 客户端返回：要求更新缓存、或允许缓存租期过期；
      >
      > 然后 Master 再继续进行之前的修改操作

    - 客户端租期到期时：向服务器续订租期

  - 类似 Expire + MQ 更新缓存

- 分布式锁错乱

  - Client-1 获取到锁，并发出请求，但迟迟没有到达服务器；被认为失败，并让Client-2 获取到锁，执行请求；

  - 之后 Client-1的请求达到服务器并被处理，覆盖了 Client-2的请求；

  - 方案：

    - 锁延迟 lock-delay

      如果客户端主动释放锁，则马上放锁；

      如果客户端异常（例如无响应）而释放锁，则服务器再保留一定的时间；

      Chubby 用此方案

########### Chubby 用此方案



​			- 锁序列器

​					> 客户端操作资源时，同时带上锁序列器（锁名，模式，序号）

​				    > 服务器会检查锁序列器是否有效

​					> Chubby 也提供此方案

###### 5. etcd

原理

- Raft 协议



#### 分布式选举

**作用**

- 负责对其他节点的协调和管理；保证其他节点的有序运行

##### 算法

###### 1. Bully 算法

> 在所有存活节点中，选取ID最大的为主节点

- 角色
  - 普通节点
  - 主节点

- 流程
  1. 节点判断自己的ID是否为当前存活的最大ID，如果是，则直接向其他节点发送 Victory 消息，宣誓自己的主权。 
  2. 节点向比自己ID大的节点发送 Election 消息，等待 Alive 回复。
  3. 如果给定时间内未收到 Alive 回复，则认为自己成为主节点，向其他节点发送 Victory 消息
  4. 如果收到 Alive 回复，则继续等待 Victory 消息；

- 特点

  - 优点：选举速度快、算法复杂度低；
  - 缺点：每个节点有全局的节点信息，额外信息存储多；
  - 缺点：ID 大的节点不稳定时会触发频繁切主；

  

######## 缺点：每个节点有全局的节点信息，额外信息存储多

######## 缺点：ID 大的节点不稳定时会触发频繁切主

###### 2. Raft 算法

> 多数派投票选举

- 角色
  - Leader 节点
  - Candidate 节点
  - Follower 节点

- 流程
  1. 初始化时都是Follower；开始选主时所有节点转化为Candidate，并向其他节点发送选举请求；
  2. 其他节点根据收到的选举请求的先后顺序，回复是否同意成为主；
  3. 若获得超过一半投票，则成成为主，状态变为 Leader；其他节点 Candidate --> Follower；
  4. Leader 任期到了，则Leader --> Follower，进入新一轮选主

- 特点

  - 优点：选举速度快、算法复杂度低

  - 优点：稳定度较 Bully好：新节点加入时会触发选主，但不一定会触发切主
  - 缺点：节点互相通信，通信量大

  

###### 3. ZAB 算法

> 在Raft基础上，保证数据新的节点优先成为主：server_id + server_zxID

ZooKeeper Atomic Broadcast

每个节点都有唯一的三元组：
- server_id: 本节点ID
- server_zxID: 本节点存放的数据ID
- epoch: 当前选举轮数

原则：
- server_zxID 最大者成为Leader;
- 若相同，则 server_id 最大者成为Leader;



- 角色
  - Leader
  - Follower
  - Observer

- 流程
  1. 刚启动时，都推选自己，选票信息 `<epoch, vote_id, vote_zxID>`
  2. 因为 epoch\zxID 都相同，server_id较大者会成为推选对象；其他节点会更新自己的投票并广播

- 特点

  - 优点：性能高
  - 优点：稳定性好，新节点加入会触发选主，但不一定触发切主
  - 缺点：广播方式发送信息，通信量大
  - 缺点：选举时间较长，除了投票还要对比节点ID和数据ID

  

#### 分布式共识

区块链

##### 算法

###### 1. PoW: Proof of Work

比计算能力


###### 2. PoS: Proof of Stake

权益是指占有货币的数量和时间


###### 3. DPoS: Delegated Proof of Stake

解决PoS的垄断wen't





#### 分布式聚合

目的

- 聚合不同微服务的数据

##### 手段

###### 1. Aggregator / BFF

- 每次计算，性能不好

###### 2. Denormalize + Materialize the view

- 消费Stream，实时预聚合

###### 3. CQRS

> Command Query Responsibility Segregation

- 技术点
  - Command: SQL 数据库
  - Query: Cassandra / ES / Redis...
  - 同步: CDC / MQ

- 问题

  - 最终一致性，不实时

  - 解决
    - UI 乐观更新：
      写入后 UI直接显示最新值；如果写入失败再回滚
    - UI 拉模式：
      UI 写入时带上version，轮询读服务查询version 更新UI
    - UI 发布订阅：
      UI 写入后，订阅读服务，当有通知是更新UI

#### 分布式事务

##### ACID 特性

A 原子性

- 要么全部执行成功，要么全部不执行
  - 实现：Write-ahead log

- 要么转账成功，要么转账失败

C 一致性

- 事务操作前后，数据的完整性保持一致
  - 实现：事务语义

- 总钱数不会变

I 隔离性

- 多个事务并发执行，不会互相干扰
  - 实现：Lock

- A转账、B查余额，依赖于隔离级别

D 持久性

- 重启后不变
  - 实现：Write-ahead log

- 一旦事务成功，则其做的更新被永远保存下来

##### 隔离级别

Read Uncommitted

- 允许脏读、不可重复度、幻读

Read Committed

- 查询只承认在`语句`启动前就已经提交完成的数据

- 解决：脏读

Repeatable Read

- 查询只承认在`事务`启动前就已经提交完成的数据

- 解决：脏读、不可重复读

Serialized

- 对相关记录加读写锁
  - 串行化，不允许并发执行

- 解决：脏读、不可重复读、幻读

##### 传播 Propagation

- REQUIRED: 当前有就用当前的，没有就新建

- SUPPORTS: 当前有就用当前的，没有就算了

- MANDATORY: 当前有就用当前的，没有就抛异常

- REQUIRES_NEW: 无论有没有，都新建

- NOT_SUPPORTED: 无论有没有，都不用

- NEVER: 如果有，抛异常

-  NESTED: 如果有，则在当前事务里再起一个事务

##### Spring分布式事务

**种类**

- **XA与最后资源博弈**
  - 两阶段提交

1. start MQ tran
2. receive msg
3. start JTA tran on DB
4. update DB
5. Phase-1 commit on DB tran
6. commit MQ tran
7. Phase-2 commit on DB tran

- **共享资源**

  - 原理
    - 两个数据源共享同一个底层资源
    -  例如ActiveMQ使用DB作为存储
    - 使用DB上的connection控制事务提交

  - 要求
    - 需要数据源支持

- **最大努力一次提交**
  - 原理
    - 依次提交事务
    - 可能出错
    - 通过AOP或Listener实现事务直接的同步
  - 例：JMS最大努力一次提交+重试

1. start MQ tran
2. receive msg
3. start DB tran
4. update DB
5. commit DB tran
6. commit MQ tran

Step4 数据库操作出错，消息会被放回MQ，重新触发该方法；
Step6 提交MQ事务出错，消息会被放回MQ，重新触发该方法；此时会重复数据库操作，需要忽略重复消息；

​			- 适用于其中一个数据源是MQ，并且事务由读MQ消息开始

​			- 利用MQ消息的重试机制

​			- 重试时需要考虑重复消息

- **链式事务**

  - 原理

    - 定义一个事务链

    - 多个事务在一个事务管理器里依次提交

    - 可能出错

      第二个提交执行中 如果数据库连接失败，第一个提交无法回滚。

  - 实现：ChainedTransactionManager

    - DB + DB

```java
@Bean
public PlatformTransactionManger trxManager() {
  DataSourceTransactionManager userTM = new DataSourceTransactionManager(userDataSource());
  DataSourceTransactionManager orderTM = ...
  
  // 顺序敏感，在前的后提交
  ChainedTransactionManager tm = new ChianedTransactionManager(orderTM, userTM);
}
```

​			- JPA + DB

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

**选择**

- 强一致性
  -  JTA
  - （性能差，只适用于单个服务内）

- 弱、最终一致性
  -  最大努力一次提交、链式事务
  - （设计相应错误处理机制）

- MQ-DB
  - 最大努力一次提交 + 重试

- DB-DB
  - 链式事务

- 多个数据源
  - 链式事务、或其他事务同步方式

**锁的实现方式**

-  JmsListener.concurrent=1

- @Transactional + 数据库记录操作update where

- 分布式锁



##### 刚性事务

https://matt33.com/2018/07/08/distribute-system-consistency-protocol/ 

###### XA 协议

- TM 事务管理器
  - 协调者
  - 负责各个本地资源的提交和回滚

- RM 资源管理器
  - 参与者
  - 例如 数据库

- AP 应用程序

  - 定义事务边界，访问边界内的资源

  

###### 2PC：两阶段提交

**流程**

- *阶段一：投票 Voting*
  - CanCommit 
    - TM 协调者：向本地资源管理器发起执行操作的 CanCommit 请求；
  - LOG
    - RM 参与者：收到请求后，执行事务操作，记录日志但不提交；返回操作结果
    - RM 参与者：Undo / Redo log

- *阶段二：提交 Commit*

  - DoCommit
    - TM 协调者：收到所有参与者的结果，如果全是YES 则发送 DoCommit 消息；
    - RM 参与者：完成剩余的操作并释放资源，向协调者返回 HaveCommitted 消息；
    - 如果此时有参与者提交失败
      - 重试：此阶段不回滚！
      - 提交是很轻量的，重试问题不大

  - DoAbort
    - TM 协调者：如果结果中有 No，则向所有参与者发送 DoAbort 消息；
    - RM 参与者：之前发送Yes的参与者会按照回滚日志进行回滚；返回HaveCommitted 消息。（基于 Undo log）

- TM 协调者收到 HaveCommitted 消息，就意味着整个事务结束了

  

>  疑问：如果 阶段二 DoCommit 执行失败怎么办？
>
>  A：通过预留阶段的确认，保证确认阶段不会出错



**不足**

- 同步阻塞问题
  - 所有节点都是事务阻塞型的
  - 参与者在等待其他参与者的响应过程中，无法进行其他操作
  - 不适合长事务

- 单点故障问题
  - 事务管理器（协调者）是单点

- 数据不一致问题
  - 协调者发送 DoCommit 后如果发生网络局部故障，会导致部分节点无法提交

- 太过保守
  - 任意一个节点的失败 都会导致整个事务的失败
  - 强一致不都这样么？

**场景**

- 适合强一致、并发量不大的场景
  - 例如下单+使用优惠券
  - 类比：组织爬山

**案例**

- Zookeeper 消息广播
  - Leader --> Follower

- Paxos 生成提案

- MySQL WAL (write ahead log)
  - 更新 redo log --> 写入binlog --> 提交 redo log

- Kafka 精确一次语义
  - 事务
  - 保证消息原子性地写入多个分区，要么全部成功，要么全部失败

- Practical Microservices Architectural Pattern
  - Atomikos

https://github.com/Apress/practical-microservices-architectural-patterns/tree/master/Christudas_Ch13_Source/ch13/ch13-01/XA-TX-Distributed



###### 3PC：三阶段提交

**流程**

- *阶段一：CanCommit*
  - 协调者发出询问
  - 参与者检查、反馈

- *阶段二：PreCommit*

  - 正常：都是Yes

    - 协调者：发送PreCommit请求
    - 参与者：收到PreCommit后执行事务操作，并将 Undo 、Redo信息记录到事务日志
    - 执行成功后发回 ACK，并等待最终指令

  - 异常：包含NO

    - 协调者：发送Abort消息

    - 参与者：收到Abort / 或超时后仍未收到协调者消息，则执行中断操作

      > 默认会中断

- *阶段三：DoCommit*

  - 正常：提交

    - 协调者：收到所有ACK后，向所有参与者发送 DoCommit
    - 参与者：正式提交事务，释放资源

  - 异常

    - 中断

      - 协调者：向所有参与者发送 Abort
      - 参与者：利用在PreCommit阶段记录的 Undo 日志，进行回滚，释放资源

    - 超时

      - 协调者：超时会收到回复，发送中断请求

        > 默认中断

      - 参与者：如果长时间没有得到协调者响应，参与者会自动提交

        > 默认会提交！！

      - 避免2PC的阻塞

**特点**

- 解决数据不一致问题
  - 引入超时机制
  - 增加PreCommit阶段，在此阶段排除一些不一致的情况
  - 但仍有风险
    - 最后一步如果出现分区，部分参与者收不到消息 仍然会提交事务

- 出现单点故障后，仍然能达成一致

- 解决同步阻塞问题

  - 引入 PreCommit 阶段

  

###### Seata

- 思想
  - 是优化的 2PC

- 特性
  - 支持多种事务模式
    AT
    TCC
    Saga
  - 

##### 柔性事务

- 原则
  - 假定网络、服务不可靠
  - 将全局事务建模成一组本地 ACID 事务
  - 引入事务补偿机制处理失败场景
  - 不管成功或失败，事务要始终处在一种明确的状态
  - 最终一致
  - 考虑隔离性
  - 考虑幂等性
  - 异步响应式，尽量避免直接同步调用

###### TCC

**流程**

- Try
  - 尝试执行业务：完成业务检查、预留必要的业务资源

- Confirm
  - 真正执行业务
- Cancel
  - 释放 Try 阶段预留的业务资源

**案例：转账**

- 汇款服务
  - Try
    - 检查：检查 A 账户余额；检查 A 账户有效性：状态是否为“转账中”、“冻结”
    - 扣减500元，并设置状态为“转账中”
      - 为什么不在Confirm 步骤中执行？
    - 预留业务资源
      - 将转账500元的事件存入消息，或日志
  - Confirm
    - 无操作
    - 状态设置为“成功”？
  - Cancel
    - A 账户 加回500元
    - 释放业务资源：回滚消息、日志
- 收款服务
  - Try
    - 检查：检查 B 账户有效性
  - Confirm
    - 读取消息、日志，B 账户增加500元
    - 释放业务资源：从消息、日志中释放
  -  Cancel
    - 无操作

**特点**

- 业务侵入大；资源锁定由业务方负责



###### Saga

- 思路
  - 将分布式事务 拆分成 多个本地事务
  - 每个本地事务 都有相应的执行模块、补偿模块
    - 状态机
  - 若某个本地事务失败，则向后补偿所有已完成的事务

**分类**

- 协同式 Saga

  - Choreography
  - 各系统之间两两MQ联系

- 编排式 Saga

  - Orchestration
  - 各系统都与集中式 Orchestrator 联系
  - 缺点：SPOF
  - 优点：交互复杂度低、方便集中管理

  

########### 方便集中管理

###### 异步MQ 事务消息

- 概念
  - 半消息：发到MQ服务端，但标记为暂不投递；直到MQ服务端收到二次确认（Commit）
  - 消息回查：消息长期处于半消息状态，则MQ服务器主动向生产者询问
  - 2PC?

- 步骤

  - 发送半消息
    - 发送方：发送半消息到MQ服务器
    - 这一步需要处理幂等，可能会发送重复半消息
    - 类似 prepare
  - 执行本地事务
    - 发送方：收到MQ服务器返回值后，执行本地事务

  - 提交或回滚
    - 发送方：执行成功后，发送commit / rollback 到MQ服务器
    - MQ服务器：投递该消息给订阅方，或删除消息
    - MQ消费端：由MQ保证消费成功
  - 回查
    - MQ服务器：扫描发现长期处于半消息状态，请求发送发回查事务状态
    - 这对 MQ 服务器要求太高
    - 发送方：检查本地事务状态，再次发送 commit / rollback

- 缺点
  - MQ要支持半消息：RocketMQ
  - 业务要提供回查
  - 发送消息非幂等
    - 可能发送多个半消息，例如当MQ返回值未收到时



###### 异步本地消息表 (事务性发件箱)

- 思路
  - 两张表：业务表、消息表 （事务性发件箱）
  - 解决MQ事务消息的缺点
    - 更常用！
    - 无需回查

- 步骤

  - 执行本地事务
    - 业务方：写入业务数据
    - 业务方：写入消息表、而不是直接写入MQ

  - 写入MQ
    - 业务方：读取消息表，写入MQ
    - MQ： 返回回执

  - 删除消息
    - 业务方：收到MQ回执 写入成功，则删除本地消息
    - 若回执丢失？
      - 重新写入 MQ
      - At Least Once

- 实现
  - Killbill Common Queue https://github.com/killbill/killbill-commons/tree/master/queue

- 案例
  - 创建订单、清空购物车
    - 正常使用订单库的事务去更新订单数据
    - 同时在本地记录一条日志，内容是“清空购物车”操作
      - 文件，或“订单库”中的一个表
    - 启动`异步服务`，读取本地消息，调用购物车服务清空购物车
    - 清空后，把本地消息状态更新为已完成



###### 异步变更数据捕获CDC 

> Change Data Capture

- 思路
  - 消费事务日志
  - Transacation Log Miner 捕获日志，并写入MQ
    - At least once

- 实现

  - Canal

    - 基于 MySQL binlog

  - Redhat Debezium

    - 基于 Kafka Connect

  - Zendesk Maxwell

    - 基于 MySQL binlog

  - Airbnb SpinalTrap

    - 基于 MySQL binlog

  - Eventuate-Tram

    - 学习用

    

###### 同步场景分布式事务

**思路**

- 基于异步补偿

- 关键点
  - 记录请求调用链路
  - 基于补偿机制
  - 提供幂等补偿接口

**实现**

- 业务方逻辑层
  - Proxy
    - 补偿服务提供的 jar：AOP @Around
    - 职责
      - 事务开始前，生成 txid，记入表：TDB.
      - 当全部执行成功，修改 status = 2（成功）；删除请求调用信息
      - 当有一步失败，则修改 status = 3 （事务失败）；返回失败：查询调用链反向补偿
  - RPC Client
    - 执行各个步骤之前，记录请求调用信息，记入表：TDB.

- 业务方数据层
  - 提供补偿接口
  - 可基于注解 标注业务方法对应的补偿方法

- 事务补偿服务
  -  TDB
    - 事务组表：记录事务状态： txid, status, ts
    - 事务调用组表：请求调用链路： txid, actionId, callmethod,  params
  - tschedule
    - 扫描事务组表 中执行失败（status=3）的txid
    - 查询事务调用组表，逐个调用补偿方法；设置 status = 4
      - 如果补偿失败？重试；重试还失败，告警人工介入
  - RPC Client
    - 负责调用补偿方法

**案例**

- 下单、减库存、付款

  - 类似 Saga：向后恢复

  

### 分布式计算

#### MapReduce

#### Stream

#### Actor

#### 流水线



### 分布式通信

#### RPC

框架

- dubbo
- gRPC

基本流程

- 协议
  - 扩展性
  - 协议体、协议头不定长

- 序列化

  - 序列化协议

    -  JDK：ObjectOutputStream

    - JSON：

      额外空间开销大

      没有类型，要通过反射，性能不好

    - Hessian

    - Protobuf

      预定义 IDL

    - kryo

  - 选型依据
    - 性能
    - 空间开销
    - 通用性、兼容性

通信模型

	- 阻塞IO
 - IO多路复用
   	- Netty

动态代理



治理

- 服务发现

- 健康监测

- 路由策略

- 负载均衡

- 异常重试

- 优雅关闭

- 优雅启动

- 熔断限流

- 业务隔离



#### 发布订阅

#### 消息队列





### 分布式存储

#### 三要素

- 数据生成者、消费者
- 数据存储

- 数据索引

  - 设计原则
    - 数据均匀
    - 数据稳定（迁移少）
    - 节点异构性
    - 隔离故障域（备份到其他节点）
    - 性能稳定

  - 分片技术
  - 一致性哈希
    - 有限负载；
    - 虚拟节点；

#### 数据复制

- 同步复制
  - 所有备库均操作成功，才返回chneg'g

- 异步复制
  - 写入主库，则返回成功
  - MySQL默认复制模式，binlog --> relay log

- 半同步复制

  - 至少一个备库写入成功
  - 至少一半备库写入成功

  

#### 分布式数据库

TBD

#### 分布式文件系统

TBD

#### 分布式缓存

##### 缓存更新策略

- LRU/LFU/FIFO 算法删除
  -  maxmemory-policy

- 超时剔除
  - expire

- 主动更新



##### 缓存读写策略

###### **Cache Aside 旁路缓存策略**

- 读：先从Cache取，没有则从数据库取，成功后放入缓存

- 写：先存入数据库，成功后再让缓存失效：删除缓存

- 问题：

  - 能否成功后`更新`缓存？--> 防止两个并发写导致不一致

    >  e.g. 数据库值=19
    >
    > 请求A: 更新为20，待更新缓存；
    > 请求B: 更新为21，并更新到缓存；
    > 请求A: 更新缓存为20
    >
    > --> db=21, cache=20

  - 能否先删除缓存，再更新数据库？--> 防止并发读写导致不一致

    >  e.g. 数据库值=19
    >
    > 请求A: 更新为20 --> 删除缓存，待更新数据库；
    > 请求B: 读取 --> 缓存未命中，查数据库19，并更新到缓存；
    > 请求A: 更新数据库为20
    >
    > --> db=20, cache=19

  - 还是存在读写并发的问题

    > e.g. 数据库值20，缓存为空；
    > 请求A: 查询, 待放入缓存；
    > 请求B: 修改为21，并清空缓存；
    > 请求A: 放入缓存20；--> 与数据库不一致
    >
    > 
    >
    > 实际很难出现B已经更新db并清空缓存，A才更新完缓存的情况。

- 变通

  - 插入数据后，可以直接写入缓存

    > 避免数据库主从同步延迟。
    > 新数据不会有并发更新问题。

###### Read/Write Through 读穿/写穿策略
- 读：从Cache取，没有则由Cache触发加载数据库
- 写：
  - 若缓存中已存在，则更新缓存，然后Cache触发更新数据库
  - 若缓存中不存在（Write Miss, 写失效）
    - Write Allocate：写入缓存，再由缓存组件同步更新到数据库
    - No-write Allocate：直接更新到数据库

########## 直接更新到数据库

- 思想

  - 用户只与缓存打交道
  - 例如 Guava Cache

  

###### Write Back 写回策略
- 思想
  - 写入数据时，只写缓存；并把缓存块标记为脏
  - 脏块只有被再次使用时才会将其中的数据写入到后端存储
- 读：读到后发现是脏的，则写入数据库
- 写

######## 若缓存中不存在，则Write Allocate

###### Write Behind Caching

- 写

  - 只更新缓存，不更新数据库
  - 异步存数据库：缓存要失效时

  

##### vs. CDN缓存

针对静态资源

1. DNS cname: 域名 --> CDN域名

2. GSLB 全局负载均衡：选取离用户最近的CDN节点



##### 问题

###### 缓存粒度控制

- 缓存全量属性
  - 通用性好、可维护性好
- 缓存部分属性
  - 占用空间小

###### 缓存热点问题

- 解决

  - 多级缓存

- 如何识别热点？

  - 客户端
  - 监控系统

  

###### 缓存并发回源问题

- 问题
  - 同一个key同时并发回源
- 解决
  - 回源时加锁 + 双重检查

```java
@Autowired
private RedissonClient redissonClient;
@GetMapping("right")
public String right() {
    String data = stringRedisTemplate.opsForValue().get("hotsopt");
    if (StringUtils.isEmpty(data)) {
        RLock locker = redissonClient.getLock("locker");
        //获取分布式锁
        if (locker.tryLock()) {
            try {
                data = stringRedisTemplate.opsForValue().get("hotsopt");
                //双重检查，因为可能已经有一个B线程过了第一次判断，在等锁，然后A线程已经把数据写入了Redis中
                if (StringUtils.isEmpty(data)) {
                    //回源到数据库查询
                    data = getExpensiveData();
                    stringRedisTemplate.opsForValue().set("hotsopt", data, 5, TimeUnit.SECONDS);
                }
            } finally {
                //别忘记释放，另外注意写法，获取锁后整段代码try+finally，确保unlock万无一失
                locker.unlock();
            }
        }
    }
    return data;
}
```



​	- 缺点是限制了访问并发性

​	- 优化

		- 用进程内锁，而非分布式锁：允许不同节点并发回源
		- 用Semaphore限制并发数：允许并发回源，但限制同时并发数



########### 允许并发回源，但限制同时并发数

###### 缓存穿透问题

- 原因
  - 大量请求未命中缓存，而访问后端系统
  - 业务代码自身问题
  - 恶意攻击、爬虫

- 发现
  - 业务响应时间
  - 监控指标：总调用数、缓存命中数、存储层命中数
- 解决
  - 缓存空对象
    - 要注意空对象对内存占用的影响
  - 布隆过滤器拦截
    - 实践：创建数据时要同时修改布隆过滤器
    - 缺点：存在误判、不支持删除元素
    - Guava BloomFilter

```java

private BloomFilter<Integer> bloomFilter;

@PostConstruct
public void init() {
    //创建布隆过滤器，元素数量10000，期望误判率1%
    bloomFilter = BloomFilter.create(Funnels.integerFunnel(), 10000, 0.01);
    //填充布隆过滤器
    IntStream.rangeClosed(1, 10000).forEach(bloomFilter::put);
}

@GetMapping("right2")
public String right2(@RequestParam("id") int id) {
    String data = "";
    //通过布隆过滤器先判断
    if (bloomFilter.mightContain(id)) {
        String key = "user" + id;
        //走缓存查询
        data = stringRedisTemplate.opsForValue().get(key);
        if (StringUtils.isEmpty(data)) {
            //走数据库查询
            data = getCityFromDb(id);
            stringRedisTemplate.opsForValue().set(key, data, 30, TimeUnit.SECONDS);
        }
    }
    return data;
}
```



 - 狗桩效应 dog-pile effect

   	- 极热点缓存项失效后大量“并发”请求穿透
   	- 方案1：启动后台线程加载缓存，未加载前所有请求直接返回
   	- 方案2：分布式锁，只有获取到锁的请求才能穿透

   

###### 缓存雪崩问题

- 问题
  - cache服务器异常，流量直接压向后端db或api，造成级联故障

- 优化

  - 保证缓存高可用
    - redis sentinel
    -  redis cluster
    - 主从漂移：VIP + keepalived

  - 保证不在同一时间过期
    - 过期时间 + 扰动值
  - 不主动过期
    - 后台线程定时更新
  - 依赖隔离组件为后端限流（降级）
  - 提前演练

  

###### 一致性问题

- 缓存值与 DB 值不一致

- 四个方案

  - 通过过期时间来更新缓存，DB更新后不会触发缓存更新

  - 在更新DB的同时更新缓存

  - 基于2，MQ异步更新缓存

  - 将DB更新、缓存更新放到一个事务中

  - 5. 订阅binlog更新缓存

    

###### 无底洞问题

- 问题
  - 加机器性能不升反降
  - 客户端批量接口需求（mget, mset）
  - 后端数据增长与水平扩展需求

- 优化

  - 命令本身优化

    - 慢查询keys
    - hgetall bigkey

  - 减小网络通信次数

    - 串行mget --> 串行IO --> 并行IO 
    - hash_tag

  - 降低接入成本

    - 客户端长连接、连接池
    -  NIO

    

## 分布式指标

### 高可靠

#### 负载均衡

- 目标
  - 避免服务器过载

- 手段

  - 轮询
  - 带权重的轮询

  

#### 降级 Degradation

- 手段
  - 降低一致性
    - 使用异步简化流程
    - 降低数据一致性：缓存
  - 停止次要功能
    - 先限流再停止
    - 例如双十一停止退货服务
  - 简化功能
  - 拒绝部分请求 --> 限流？
  - 限流降级、开关降级

- 触发
  - 吞吐量过大
  - 响应时间过慢
  - 失败次数过多
  - 网络或服务故障

- 原理

  - hystrix fallback 原理

    https://segmentfault.com/a/1190000005988895

  - 滚桶式统计
    - 10秒滚动窗口
    -  每1秒一个桶
  -  RxJava Observable.window()实现滑动窗口

#### 熔断 Circuit Breaker

保护服务调用发

- 作用
  - 防止程序不断尝试执行可能会失败的操作
  - 防止浪费CPU时间去等待长时间的超时产生
  - 防止雪崩

- vs. 降级
  - 降级概念更广，熔断是降级的一种

- 原理

  - Hystrix实现

    https://github.com/Netflix/Hystrix/wiki/How-it-Works#CircuitBreaker

  - resilience4j https://resilience4j.readme.io/docs/circuitbreaker

  - Jedis 封装示例

    - open状态时定期检测可用性

```java
new Timer("RedisPort-Recover", true).scheduleAtFixedRate(new TimerTask() {
  @Override
  public void run() {
    if (breaker.isOpen()) {
      Jedis jedis = null;
      try {
  jedis = connPool.getResource();
  jedis.ping(); // 验证 redis 是否可用
  successCount.set(0); // 重置连续成功的计数
  breaker.setHalfOpen(); // 设置为半打开态
      } catch (Exception e) {
      } finally {
        if (jedis != null) {
          jedis.close();
      }
    }
   }
   }
}, 0, recoverInterval); // 初始化定时器定期检测 redis 是否可用
```



​		- 操作数据时判断状态

```java
//1. 断路器打开则直接返回空值
if (breaker.isOpen()) { 
  return null;  
}

K value = null;
Jedis jedis = null;

try {
  jedis = connPool.getResource();
  value = callback.call(jedis);
  
  //2. 如果是半打开状态
  if(breaker.isHalfOpen()) {
if(successCount.incrementAndGet() >= SUCCESS_THRESHOLD) {// 成功次数超过阈值
failCount.set(0);  // 清空失败数   
breaker.setClose(); // 设置为关闭态
          }
     }
     return value;
} catch (JedisException je) {

// 3. 失败：如果是关闭态
if(breaker.isClose()){ 
if(failCount.incrementAndGet() >= FAILS_THRESHOLD){ // 失败次数超过阈值
 breaker.setOpen();  // 设置为打开态
   }
} 

//4. 失败：如果是半打开态
else if(breaker.isHalfOpen()) {  
  breaker.setOpen();    // 直接设置为打开态
  } 
  throw  je;
  
} finally {
     if (jedis != null) {
           jedis.close();
     }
}
```



 - 状态

    - Closed
      	- 记录失败次数
    - Open
      	- 失败次数超过阈值，则断开
      	- 并且开启一个超时时钟 --> how?

   - Half-Open
     - 超时时钟到期，允许一定数量的请求调用服务

- 实践
  - 错误类型
    - 有些错误先走重试，比如限流、超时
    - 有些错误直接熔断，比如远程服务器挂掉
  - 日志监控
  - 测试服务是否可用
    - ping
    - 不必等到真实流量才切回closed
  - 手动重置
  - 并发问题: atomic
  - 资源分区
    - 只对有问题的分区熔断，而不是整体

#### 限流 Throttle

保护服务提供方

- 作用
  - 对并发访问进行限速，对整体流量做塑形
  - 保护服务调用方

- 部署位置
  - API 网关
  - RPC 客户端

- 后果
  - 拒绝服务
  - 服务降级
  - 特权请求: 多租户
  - 延时处理

- 设计

  - 限流规则

    - 阈值的设置
      - 配置中心，动态调整
      -  yml, properties

    ```
    configs:
    
    - appId: app-1
      limits:
      - api: /v1/user
        limit: 100
        unit：60
      - api: /v1/order
        limit: 50
    - appId: app-2
      limits:
      - api: /v1/user
        limit: 50
      - api: /v1/order
        limit: 50
    ```



 - ​	配置值确定
   	-  压测
   	- 粒度不能过大：起不到保护作用
   	- 粒度不能过小：容易误杀

########## 容易误杀

	- 限流算法
 - 限流模式
   	- 单机
    - 分布式
      	- 集中式管理计数器；例如 存到Redis；注意 Redis超时情况，设置合理超时时间

########## 例如 存到Redis

########## 注意 Redis超时情况，设置合理超时时间

- 限流算法
  - 固定窗口算法

```java
private AtomicInteger counter;

public boolena isRateLimit() {
  return counter.incrementAndGet() >= allowedLimit;
}


ScheduledExecutorService timer = Executors.newSingleThreadScheduledExecutor();

timer.scheduleAtFixedRate(new Runnable(){
    @Override
    public void run() {
        counter.set(0);
    }
}, 0, 1, TimeUnit.SECONDS);
```



- ​	计数器方式

  - 缺点

  - 无法限制短时集中流量：窗口边界流量

    例如限制 10次/s，但无法限制下列场景：

    前一秒最后10ms --> 10次请求
    后一秒最前10ms --> 10次请求



 - ​	滑动窗口算法
   	- 划分为多个小窗口
 - 队列算法
   	- 普通队列
   	- 优先级队列
   	- 带权重的队列：对优先级队列的优化，避免低优先级队列被饿死



- 漏桶算法 Leaky Bucket
  - 用队列实现，队满则拒绝
  - 速率控制较均匀、精确
  - 不能处理突发流量

- 令牌通算法 Token Bucket
  - 中间人：往桶里按照固定速率放入token，且总数有限制
  - 限速不精确，能处理突发流量
    - 桶的大小决定了突发流量处理量
    - 流量小时积攒token，流量大时可以快速处理
  - 令牌的存储
    - 单机： 变量
    - 分布式：Redis
      - 性能优化：每次取一批令牌，减少请求redis的次数

- 基于响应时间的动态限流
  -  TCP: Round Trip Time拥塞控制算法

- 全局流控
  - Redis INCR + Lua

```
-- 操作的 Redis Key
local rate_limit_key = KEYS[1]
-- 每秒最大的 QPS 许可数
local max_permits = ARGV[1]
-- 此次申请的许可数
local incr_by_count_str = ARGV[2]

-- 当前已用的许可数
local currentStr = redis.call('get', rate_limit_key)

local current = 0
if currentStr then
    current = tonumber(currentStr)
end

-- 剩余可分发的许可数
local remain_permits = tonumber(max_permits) - current

local incr_by_count = tonumber(incr_by_count_str)
-- 如果可分发的许可数小于申请的许可数，只能申请到可分发的许可数
if remain_permits < incr_by_count then
    incr_by_count = remain_permits
end

-- 将此次实际申请的许可数加到 Redis Key 里面
local result = redis.call('incrby', rate_limit_key, incr_by_count)
-- 初次操作 Redis Key 设置 1 秒的过期
if result == incr_by_count then
    redis.call('expire', rate_limit_key, 1)
end

-- 返回实际申请到的许可数
return incr_by_co

```



 - 考虑点
    - 流控粒度问题
      	- 分成若干 N毫秒 的桶 + 滑动窗口
    - 流控依赖资源存在瓶颈问题
      	- 本地批量预取
      	- 以限流误差为代价

### 高可用

#### 手段

##### 故障隔离 Bulkheads

https://resilience4j.readme.io/docs/bulkhead

###### 故障隔离策略

####### 线程级隔离

######## 常用于单体应用

######## 通信

######### 共享变量

####### 进程级隔离

######## 通信

######### 信号量

######### 消息队列

######### 共享内存

######### RPC

####### 资源隔离

######## 微服务

######### 服务注册时：接口名 + 分组参数

######### 服务发现时：不同客户端的请求带的分组参数不一样，获取的服务器列表也就不一样了

######## 通过容器隔离资源

####### 用户隔离（多租户）

######## 服务共享

######## 数据隔离

###### 隔离带来的问题

####### 多板块数据的聚合：响应时间下降

####### 大数据仓库：增加数据合并复杂度

####### 故障雪崩

####### 分布式事务

##### 故障恢复 Failover

###### 故障检测

####### 心跳

######## 固定心跳检测

######## 基于历史心跳消息预测故障

###### 故障恢复

####### 对等节点

######## 随机访问另一个即可

####### 不对等节点

######## 选主：在多个备份节点上达成一致

######## Poxos，Raft

##### 超时控制

###### 目的是不让请求一直保持，释放资源给接下来的请求使用

###### 设置

####### ConnectTimeout

######## 建立连接阶段最长等待时间

######## 1~5s即可

######### 几秒连接不上，则可能永远连不上；配长了没意义

####### ReadTimeout

######## 从socket上读取数据的最长等待时间

######## 设为 TP99 RT

######### 过长：下游抖动会影响到客户端自己，线程hang

######### 过短：影响成功率

##### 重试 Retry

###### 场景

####### 认为这个故障时暂时的，而不是永久的

######## 网络抖动

####### 调用超时

####### 返回了某种可以重试的错误：繁忙中，流控中，资源不足...

####### 注意要幂等！

######## 绝对值的修改 天然幂等

######## 相对值的修改

######### 加 where 条件

######### 转换成绝对值修改（先查出来）

###### 重试策略(Spring)

```java
@Service
public interface MyService {
    @Retryable(
      value = { SQLException.class }, 
      maxAttempts = 2,
      backoff = @Backoff(delay = 5000))
    void retryService(String sql) throws SQLException;
    ...
}

```

####### NeverRetryPolicy

######## 只调一次

####### AlwaysRetryPolicy

######## 无限重试，直到成功

####### SimpleRetryPolicy

######## 固定次数重试

####### TimeoutRetryPolicy

######## 在超时时间内允许重试

####### CircuitBreakerRetryPolicy

######## 有熔断功能的重试策略

####### CompositeRetryPolicy

######## 组合

###### 退避策略(Spring)

####### NoBackOffPolicy

######## 立即重试

####### FixedBackOffPolicy

######## 固定时间退避

######## sleeper | backOffPeriod

####### UniformRandomBackOffPolicy

######## 随机时间退避

######## sleeper | minBackOffPeriod | maxBackOffPeriod

####### ExponentialBackOffPolicy

public static long getWaitTimeExp(int retryCount) {
    long waitTime = ((long) Math.pow(2, retryCount) );
    return waitTime;
}


######## 指数对比策略

####### ExponentialRandomBackOffPolicy

######## 指数对比策略，并引入随机乘数

###### 实现

```java
public static void doOperationAndWaitForResult() {

// 异步操作
long token = asyncOperation();
int retries = 0;
boolean retry = false;

do {
 // 获取异步操作结果
 Results result = getAsyncOperationResult(token);
 if (Results.SUCCESS == result) {
   retry = false;
 } else if (Results.NOT_READY == result 
  || Results.TOO_BUSY == result
  || Results.NO_RESOURCE == result
  || Results.SERVER_ERROR == result) {
    retry = true;
 } else {
    retry = false;
}

 if (retry) {
  long waitTime = Math.min(getWaitTimeExp(retries), MAX_WAIT_INTERVAL);
  // 等待下次 retry
  Thread.sleep(waitTime);
  }

} while (retry && (retries++ < MAX_RETRIES));



```

#### 指标

##### MTBF / (MTBF + MTTR)

###### MTBF: Mean Time Between Failure

###### MTTR: Mean Time To Repair

##### VALET

###### Volume - 容量

####### TPS

###### Availability - 可用性

###### Latency - 时延

###### Errors - 错误率

###### Tickets - 人工介入

#### 框架

##### Hystrix

###### HystrixCommand

####### run()

####### getFallback()

####### 执行

######## queue()

######## observe()

###### Spring Cloud Hystrix

####### @HystrixCommand

###### 请求合并、请求缓存

###### 隔离

####### 信号量隔离

######## 轻量

######## 不支持任务排队、不支持主动超时、异步调用

######## 适用高扇出，例如网关

####### 线程隔离

######## 支持排队、超时、异步调用

######## 线程调用会产生额外开销

######## 适用有限扇出

###### 监控

####### hystrix.stream

####### hystrix dashboard

####### turbine

######## 聚合

######## 输出到dashboard

##### Alibaba Sentinel

##### Resilience4j

#### 防雪崩

https://segmentfault.com/a/1190000005988895

##### 流量控制

###### 网关限流

###### 用户交互限流

###### 关闭重试

##### 改进缓存模式

###### 缓存预加载

###### 同步改为异步刷新

##### 服务自动扩容

##### 服务调用者降级

### 高性能

#### 手段

##### 池化

###### 连接池

####### 注意池中连接的维护问题

######## 用一个线程定期检测池中连接的可用性：C3P0

######## 获取到连接之后先校验可用性：DBCP testOnBorrow

###### 线程池

####### 优先放入队列暂存，而不是开启新线程，适用于CPU型任务

IO型任务更适合直接创建线程，例如tomcat线程池即是如此。


####### 要监控线程池队列的堆积量

####### 不能使用无界队列！会触发Full GC

##### 读写分离

###### 主从复制

####### binlog

###### 复制延迟

####### 使用缓存

######## 适合新增数据的场景

####### 读主库

###### 屏蔽底层访问方式

####### 植入应用程序内部

######## TDDL

####### 单独部署代理层

######## Mycat

##### 分库分表

###### 垂直拆分

####### 专库专用

###### 水平拆分

####### 按哈希值拆分

####### 按字段区间拆分

###### 引入的问题

####### 引入了分区键，所有查询都要带上这个字段

####### 跨库Join

####### 主键的全局唯一性问题

单库单表 一般用自增字段作为主键。

######## UUID

######### 不递增，不有序

######### 不具备业务含义

######### 耗费空间

######## Snowflake

######### 算法

########## 时间戳 + 机器ID + 序列号

如果独立主备部署（而不是分布在业务代码中），则机器ID可省略

######### id偏斜问题

问题：qps不高时，比如每毫秒只发一个id，id末位永远是1；则表库分配不均匀。

思路：
- 时间戳不记录毫秒，而是记录秒。
- 序列号起始号做随机，这秒是21，下秒是30

######### 时钟不准问题

可让发号器暂时拒绝发号。

######## 百度 UidGenerator

https://github.com/baidu/uid-generator/

######## 美团 Leaf

https://tech.meituan.com/2017/04/21/mt-leaf.html

######## 微信序列号生成器

https://www.infoq.cn/article/wechat-serial-number-generator-architecture

##### 缓存

#### 指标

##### 吞吐量 Throughput

##### 响应时间 Response Time

##### 完成时间 Turnaround Time

### 高并发

#### 手段

##### Scale out 横向扩展

###### 数据库主从

###### 分库分表

###### 存储分片

##### 缓存

###### CPU 多级缓存

###### 文件 Page Cache 缓存

##### 异步

### 可扩展性

#### 手段

##### 无状态

##### 拆分

###### 数据库

###### 业务

###### 核心 vs. 非核心

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

### 压测

#### 基础

##### 指标

###### TPS

####### TPS 和 RT 有相关性

######## TPS = (1000 ms / RT ms) * 压力机线程数

######## 示例

######### RT = 25 ms, 压力机线程 10

######### TPS = 1000 / 24 * 10 = 400

####### 并发和压力机线程数无关！

###### RT

#### 工具

##### JMeter

## 案例

### 计数系统设计

#### 微博点赞数、评论数存储

##### 特点

###### 数据量巨大 --> 要考虑存储成本

###### 访问量大 --> 性能

###### 可用性、准确性高

##### 方案

###### 1. MySQL

####### id, comment_count, praise_count

####### 分库分表

######## id HASH

######## id 时间

######### 会导致冷热不均衡，近期数据总是访问多

###### 2. Redis

####### 内存优化

######## 合并存储点赞数 + 评论数 +...

######## 解决多个相同 ID 的存储成本

###### 3. SSD + 内存：冷数据存磁盘

####### 热点属性：把久远数据迁移到SSD，缓解内存

####### 读冷数据时，使用异步线程从SSD加载到内存中单独的 Cold Cache

##### 非功能需求

###### 如何降低写压力？

####### 消息队列

####### 批量处理消息、合并 +N

######## 预聚合

###### 可扩展

####### Sharding

####### 冷数据迁移（对象存储），热数据缓存

###### 高可靠

####### 持久化 、Replication、Checkpoint (kafka offset)

#### 系统通知未读数（共享存储场景）

##### 特点

###### 所有用户通知列表是一样的

##### 方案

###### 1. 存储每个用户的未读数

####### 有新通知时，需要遍历所有用户、并+1

####### 大部分用户不活跃，存储浪费

###### 2. 通知列表共享，存储每个用户读过的最后一条消息的ID

####### 全部用户共享一份有限的存储，每个人只记录自己的偏移量

####### 读过的ID为空，则为新用户，未读数返回0

####### 对非活跃用户，定期清空存储

#### 微博未读数（信息流未读场景）

##### 特点

###### 每个用户的信息流不一样

###### 不可使用共享的存储

##### 方案

###### 通用计数器，存储每个用户发布的博文数

###### Redis 存储用户所有关注人的博文数快照，该快照在点击未读消息时刷新

###### 未读数 = 关注人实际的博文总数  -  快照中的博文总数

### 信息流设计

#### 推模式（写扩散机制）

##### 思路

###### 发微博时，主动写入粉丝收件箱

##### 方案

###### 表设计

####### Feed表

####### Timeline表

###### 发布微博时

####### 往自己发件箱（Feed）里写入一条记录 

####### 往所有粉丝收件箱里（Timeline）也写入一条记录

######## 一般用消息队列来消除写入的峰值

###### 查询微博时

####### 查询自己的收件箱即可

##### 优缺点

###### - 写入延迟

####### 缓解：多线程消费

###### - 存储成本高

####### 缓解：定期清理Timeline表，只保存最近数据

###### - 扩展性差

####### 例如要支持对关注人分组的话，必须新建 Timeline表？

####### 存储成本更高了

###### - 取关、删除微博逻辑复杂

##### 适用场景

###### 粉丝数小、粉丝数有上限的场景

###### 例如朋友圈

#### 拉模式

##### 思路

###### 用户主动拉取关注人的微博，进行排序、聚合

##### 方案

###### 表设计

###### 发布微博时

####### 只需写入自己的发件箱（Feed）

###### 查询微博时

####### 查询关注列表

####### 查询关注列表的发件箱

##### 优缺点

###### + 彻底解决延迟问题

###### + 存储成本低

###### + 扩展性好

###### - 数据聚合逻辑复杂

####### 缓解：缓存用户最近5天发布的微博ID

###### - 缓存节点带宽成本高

####### 缓解：增加缓存副本

#### 推拉结合

##### 思路

###### 大V 发布微博后，只推送给活跃用户

###### 什么是大V

####### 粉丝数超50w

###### 什么是活跃用户

####### 最近在系统中有过操作，则做标记

##### 方案

###### 每个大V 维护一个活跃粉丝列表

####### 固定长度

###### 用户变成活跃时

####### 查询关注了哪些大V

####### 写入到大V活跃粉丝列表

######## 若超长，则剔除最早加入的粉丝

### 数据迁移

#### 步骤

##### 数据同步

###### Canal

##### 双写

###### 结果要以旧库为准

####### 不能让新库影响到业务可用性和数据准确性

###### 对比 / 补偿

####### 选择合适的时间窗口

##### 灰度发布切到新库

#### 原则

##### 确保每一步可快速回滚

##### 确保不丢数据

###### 对比补偿

### 电商

#### 订单

##### 订单数据存储

###### 订单主表

###### 订单商品表

###### 订单支付表

###### 订单优惠表

##### 场景

###### 创建订单的幂等性

####### 预先生成订单号

####### 利用数据库唯一性约束，避免重复写入订单、避免重复下单

###### 修改订单的幂等性（ABA问题）

####### 引入版本号，避免并发更新问题

####### UPDATE orders set ..., version = version + 1
WHERE version = 8;

##### 性能

###### 存档历史订单数据

###### 分库分表

####### 分表

######## 解决数据量大的问题

####### 分库

######## 解决高并发问题

####### sharding KEY

######## 原则：让查询尽量落在一个分片中

######### 用户ID后几位， 作为订单ID的一部分

########## 按订单ID查找

########## 按用户ID查找

######## Q: 查询能力受限，例如按商家ID查询？

######### 同步到其他存储系统

####### 分片算法

######## 保证均匀，避免出现热点问题

######## 常用

######### 范围分片

########## 对查询优化，但易热点

######### 哈希分片

######### 查表法

########## 例如 Redis Cluster：查找槽位

#### 商详

##### 商品数据存储

###### 基本信息

####### 特点：属性固定

####### 方案：数据库 + 缓存

######## 注意保留历史版本：历史表，或KV存储

###### 商品参数

####### 特点：不同商品参数不一样，字段庞杂

####### 方案：MongoDB

###### 图片视频

####### 方案：对象存储，例如S3

##### 场景

###### 商品介绍静态化

####### 以空间换时间

####### nginx

#### 购物车

##### 未登录购物车

###### 客户端存储

####### Session / Cookie / LocalStorage

##### 登录购物车

###### MySQL

###### Redis

#### 账户

##### 一致性问题

###### 事务 ACID

###### 隔离级别

##### 账户余额更新逻辑

###### 账户余额表：增加字段log_id，表示最后一笔交易流水号

###### 开启事务，查询当前余额、log_id

###### 写入交易流水表

###### 更新账户余额，where log_id = xx

###### 检查更新余额的返回值，>0 则提交事务，否则回滚
