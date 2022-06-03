

[toc]



TBD

- Chapter 1: Scale From Zero To Millions Of Users
- Chapter 2: Back-of-the-envelope Estimation
- Chapter 3: A Framework For System Design Interviews
- Chapter 4: Design A Rate Limiter
- Chapter 5: Design Consistent Hashing
- Chapter 6: Design A Key-value Store
- Chapter 7: Design A Unique Id Generator In Distributed Systems
- Chapter 8: Design A Url Shortener
- Chapter 9: Design A Web Crawler
- Chapter 10: Design A Notification System
- Chapter 11: Design A News Feed System
- Chapter 12: Design A Chat System
- Chapter 13: Design A Search Autocomplete System
- Chapter 14: Design Youtube
- Chapter 15: Design Google Drive
- Chapter 16: The Learning Continues





# | 计数系统设计

## || 微博点赞数、评论数存储

**特点**

- 数据量巨大 --> 要考虑存储成本
- 访问量大 --> 性能
- 可用性、准确性高



**存储方案**

**1. MySQL**

- 表设计：id, comment_count, praise_count
- 分库分表
  - id：HASH
  - id：时间
    - 会导致冷热不均衡，近期数据总是访问多

**2. Redis**

- 内存优化
  - 合并存储点赞数 + 评论数 +...
  - 解决多个相同 ID 的存储成本

**3. SSD + 内存：冷数据存磁盘**

- 热点属性：把久远数据迁移到SSD，缓解内存
- 读冷数据时，使用异步线程从SSD加载到内存中单独的 Cold Cache



**非功能需求**

**1. 如何降低写压力？**

- 消息队列
- 批量处理消息、合并 +N
  - 预聚合



**2. 可扩展**

- Sharding
- 冷数据迁移（对象存储），热数据缓存



**3. 高可靠**

- 持久化
- Replication
- Checkpoint (kafka offset)



## || 系统通知未读数（共享存储场景）

**特点**：所有用户通知列表是一样的

**方案**

**1. 存储每个用户的未读数**

- 有新通知时，需要遍历所有用户、并+1
- 大部分用户不活跃，存储浪费



**2. 通知列表共享，存储每个用户读过的最后一条消息的ID**

- 全部用户共享一份有限的存储，每个人只记录自己的偏移量
- 如果读过的 ID 为空，则为新用户，未读数返回 0
- 对非活跃用户，定期清空存储



## || 微博未读数（信息流未读场景）

特点

- 每个用户的信息流不一样，不可使用共享的存储

方案

- 通用计数器，存储每个用户发布的博文数；
- Redis 存储用户所有关注人的博文数快照，该快照在点击未读消息时刷新；
- `未读数 = 关注人实际的博文总数 - 快照中的博文总数`



# | 电商

## || 订单

**订单数据存储**

- 订单主表
- 订单商品表
- 订单支付表
- 订单优惠表



**场景**

**1. 创建订单的幂等性**

- 预先生成订单号
- 利用数据库唯一性约束，避免重复写入订单、避免重复下单



**2. 修改订单的幂等性（ABA问题）**

- 引入版本号，避免并发更新问题

  > UPDATE orders set ..., version = version + 1
  > WHERE version = 8;



**性能**

- **存档历史订单数据**

- **分库分表**

分表：解决数据量大的问题

分库：解决高并发问题

Sharding KEY：

- 原则：让查询尽量落在一个分片中
  - 用户ID后几位， 作为订单ID的一部分
  - 按订单ID查找
  - 按用户ID查找

- Q: 查询能力受限，例如按商家ID查询？
  - 同步到其他存储系统



**分片算法**

> 保证均匀，避免出现热点问题

- 范围分片
  - 对查询优化，但易热点

- 哈希分片

- 查表法
  - 例如 Redis Cluster：查找槽位



## || 商详

**商品数据存储**

- 基本信息
  - 特点：属性固定
  - 方案：数据库 + 缓存
  - 注意保留历史版本：历史表，或 KV 存储

- 商品参数
  - 特点：不同商品参数不一样，字段庞杂
  - 方案：MongoDB

- 图片视频
  - 方案：对象存储，例如S3



**商品介绍静态化**

- 以空间换时间
- nginx



## || 购物车

**未登录购物车**

- 客户端存储
  - Session / Cookie / LocalStorage

**登录购物车**

- MySQL
- Redis



## || 账户



**一致性问题**

- 事务 ACID
- 隔离级别



**账户余额更新逻辑**

- 账户余额表：增加字段 `log_id`，表示最后一笔交易流水号
- 开启事务，查询当前余额、log_id
- 写入交易流水表
- 更新账户余额，where log_id = xx
- 检查更新余额的返回值，>0 则提交事务，否则回滚







# | 信息流服务

## || 推模式（写扩散机制）

发微博时，主动写入粉丝收件箱

**表设计**

- Feed表
- Timeline表

**发布微博时**

- 往自己发件箱（Feed）里写入一条记录 
- 往所有粉丝收件箱里（Timeline）也写入一条记录。一般用消息队列来消除写入的峰值

**查询微博时**

- 查询自己的收件箱即可



**优缺点**

- **- 写入延迟**

缓解：多线程消费

- **- 存储成本高**

缓解：定期清理Timeline表，只保存最近数据

- **- 扩展性差**

例如要支持对关注人分组的话，必须新建 Timeline表？

存储成本更高了

- **- 取关、删除微博逻辑复杂**



**适用场景**

- 粉丝数小、粉丝数有上限的场景；例如朋友圈



## || 拉模式

用户主动拉取关注人的微博，进行排序、聚合

**发布微博时**

- 只需写入自己的发件箱（Feed）

**查询微博时**

- 查询关注列表
- 查询关注列表的发件箱



**优缺点**

- **+ 彻底解决延迟问题**

+ **+ 存储成本低**

- **+ 扩展性好**

- **- 数据聚合逻辑复杂**

缓解：缓存用户最近5天发布的微博ID

- **- 缓存节点带宽成本高**

缓解：增加缓存副本



## || 推拉结合

**思路**

大V 发布微博后，只推送给活跃用户

- 什么是大V？粉丝数超50w
- 什么是活跃用户？ 最近在系统中有过操作，则做标记



**方案**

- 每个大V 维护一个活跃粉丝列表：固定长度
- 用户变成活跃时
  - 查询关注了哪些大V，写入到大V活跃粉丝列表
  - 若超长，则剔除最早加入的粉丝





# | 分布式计数服务

功能需求：

- 计数

- 查询

非功能需求：

- **规模**：1w+ 次每秒观看记录；
- **性能**：写入读取延迟 ms 级别；写入到读取分钟级延时；
- **高可用**：无单点失败；



### 存储

**数据库选型 : SQL** 

- Replica 主从：解决高可用问题
- Sharding 分片：分摊负载
- 代理：ShardingSphere (ShardingJDBC) --> 路由 + Registry Center

**数据库选型 : NOSQL**

- Cassandra: gossip 让每个机器都知道集群信息；复制因子、仲裁（多数成功即可）
- 动态添加节点、



**表设计：**

- 基本信息：video_info (videoId, name, spaceId)

- 聚合数据：video_stats (videoId, ts, count)  
- NOSQL 记录每个事件：(videoId, ts1_count, ts2_cont) --> 列式数据库，记录每分钟的操作数目

设计考察点

- 老数据归档问题：DB --> 对象存储
- 热点数据：缓存 --> 三级缓存：cache / db / S3



### 计数服务

**设计思路**

- 可扩展：分区，sharding
- 高性能：内存计算，批处理 （预聚合）
- 高可靠：持久化、Replication、checkpointing（kafka offset）

**MQ 解耦**

- 引入MQ：请求直接丢人MQ，计数服务消费、进行本地聚合、写入数据库。
- 好处：
  - 1、解耦，削峰；
  - 2、即便计数器服务下线维护，MQ仍然能保证请求不丢，后续仍能补充计数。
- 统计粒度：按分钟的队列、按小时的队列、按天的队列... --> 按分钟统计完后，扔进按小时队列，依次类推；类似 Kafka Streaming。



**数据接收层设计**

- API Gateway: 转发用户请求。
  --> 问题：负载均衡，服务发现，限流容错，防爬虫
- Counting Service：分发到 kafka 分区 
  --> 问题：同步生产？异步生产？MQ 高可靠？



**计数消费者设计**

- MQ消费者：拉取分区消息，放入本地内存队列1。
- Aggregator：消费本地内存队列数据、聚合，写入内部队列2 （磁盘、db、kafka）。
- DB Writer：将计算结果写入DB。如果无法写入，放入死信队列。

代码设计：

```
- countViewEvent(videoId) 
- countEvent(videoId, eventType)
- processEvent(video, eventType, count|sum|avg) 
- processEvent(eventList)
```



### 查询服务

**数据获取层设计**

- API Gateway: 接受并转发用户请求；
- Query Service: 查询数据库；

代码设计：

```
- getViewsCount(videoId, startTime, endTime)
- getCount(videoId, eventType, s, e)
- getStat(videoId, eventType, count|sum|avg, s, e)
```



### 考察点

1. 监控
   - 压力测试
   - 调用量、错误数
   - 队列消息堆积

2. 如何保证功能正确
   - 线下功能测试
   - 模拟用户事件、进行校验 
   - 实时流计算系统、线下批处理系统同时计算，校验对比。--> Lambda Architecture
3. 热分区问题 （热点数据导致某些队列分区访问频繁）
   - 视频ID key添加时间戳
4. 如何监控慢消费者
   - see MQ design



# | MQ 系统设计



## || Rebalancer: Partition assigner

需求

Think Kafka consumer groups. With a Kafka consumer group you have P partitions and C consumers and you want to balance consumption of the partitions over the consumers such that:

- Allocation of partitions to consumers is balanced. With 7 partitions and 3 consumers, you’ll end up with 3, 2, 2.
- No partition can be allocated to more than one consumer
- No partition can remain unallocated

Also, when a new partition is added, or a consumer is added or removed or fails, then the partitions need to be rebalanced again.



https://jack-vanlightly.com/blog/2019/1/25/building-a-simple-distributed-system-the-what 

- 基于ZK分配任务

  ```
  /rebalanser
     /{group name} (permanent)
        /clients (permanent)
           /c_000000000 (ephemeral sequential)
           /c_000000001 
           /c_000000002
           /c_n
        /resources (permanent - data: stores allocations map)
           /{res id 1} (permanent)
           /{res id 2}
           /{res id n}
        /barriers (permanent)
           /{res id 1} (ephemeral)
           /{res id 2}
           /{res id n}
        /term (permanent)
  ```

  

- `/clients/`客户端启动时注册 zk：

  - 临时节点、递增节点

  - 如果 ID 最小，则成为 Leader；并监听 **term** 节点；

    > 避免脑裂多个节点成为 Leader，当 term 节点版本更新，上一个 Leader 监听到并退位(abdicate)

  - 如果 ID 不是最小，则成为 Follower；并监听**前一个 ID** 节点；

  - 







# | Chat 

https://systeminterview.com/design-a-chat-system.php  



需求

- Receive messages from other clients.
- Find the right recipients for each message and relay the message to the recipients.
- If a recipient is not online, hold the messages for that recipient on the server until she is online.



沟通方式

- **Polling**

  - 客户端每隔一段时间发请求，询问是否有新消息。
  - 缺点：
    - 浪费服务端资源

- **Long Polling**

  - 客户端发起链接后不会马上关闭，而是等待一段时间服务器响应。
  - 缺点：
    - 发送者和接收者可能连到不同服务器
    - 服务器无法感知客户端是否断链；
    - 效率低，需要为低频客户端保持链接。

- **WebSocket** 

  - 建立持久化双向通道

    > It starts its life as a HTTP connection and could be “upgraded” via some well-defined handshake to a WebSocket connection.



组件

- **Chat Server**

  - 处理消息收发

- **Presence Server**

  - 管理上下线状态

- **API Server**

  - 处理用户登录、注册、修改 profile

- **Notification Server**

  - 推送提醒消息

- **KV Store**

  - 保存消息历史 

  - MessageId: 要求有序

    > 三种思路
    >
    > - 自增逐渐：nosql 不支持；
    > - 全局ID生成器，Snowflake：重量级
    > - 本地ID生成器：只要保证在一对一聊天、或一个群聊内唯一且有序即可。



消息流程

- UserA 发送消息给 Chat Server 1；

- Chat Server1 调用 ID 生成器生成 message id；

- Chat Server1 将消息发送到队列；

- 消息存储到 KV store；

- 如果 UserB在线，则消息转发给UserB对应的Chat Server2；由Chat Server2通过websocket发送给UserB。

  > Q: 如果转发到 Chat Server2? 
  >
  > - A: 增加 User Mapping Service, 存储 user·-server对应关系。
  >
  > Q: 找到server后，如何通信？
  >
  > - A1：为每个 server创建 dedicated queue。缺点：队列太多、server宕机后 队列里的消息怎么办。
  > - A2：HTTP 会导致消息无序、且请求量巨大。
  > - A3：改进HTTP：添加 prevMsgId 保序、使用 buffer 降低请求量。

- 如果 UserB 不在线，则发送调用Notification Server发送 Push notification



群聊

- 群组较小时：PUSH
- 群组较大时：PULL



上下线状态

- 心跳保活
- 存储到redis （同时也是 user-server mapping）



# | 会话缓存服务

需求

- 消除粘性会话 Sticky Session：单点问题，发布问题，难以水平扩展。

**设计思路**

- 四种常用会话技术：粘性会话、纯客户端会话、服务器共享会话，集中式会话。
- sessionId 与 SessionServer的映射关系问题：一致性哈希不适用，因为服务器挂了会导致数据丢失；cookie 携带 server IP，如果读失败，则从DB加载数据、重新分配server IP。
- SessionServer挂了怎么办：缓存写后（异步刷盘到DB）
- 升级扩容 / 服务发现：



**SessionServer 内部设计**

- 两级缓存，平衡性能和容量：
  - LRU内存缓存；
  - 磁盘存储：存储不活跃的session，cache eviction后放入二级磁盘存储。value = 存储块ID + 位移；存储块整理线程，清理存储块中被删除的段。
    --> 参考 Yahoo HaloDB
- DB持久化（缓存写后异步刷盘，也可刷到集中式缓存，参考携程 x-pipe）
  - 异步操作 如何保证一定成功？
- LRU 缓存实现：
  - Guava Cache --> Caffeine
  - 线程安全：get/put - 写锁；size - 读锁
  - 线程安全 + 高并发：分段锁 LruCache[] 

**集群跨数据中心HA**

- 摇摆策略：各数据中心互通、互相注册。
- 双写策略：异步写入另一个数据中心。



# | 延迟任务队列

**思路**

- 周期性扫描任务表：`select * from tasks where alert_after < now`。同时计算下一次执行时间。

开源：

- `db-scheduler`  
- `killbill notification queue` 
- `quartz` 重量
- `Xxl-job`
- `cron-utils` 工具类



**如何保证一个任务只被一个worker执行：**

- 1) 乐观锁：所有worker都能获取任务，但只有第一个写入的才能写入成功。`update version = version + 1 where version = ? ` 
-  2) 每个节点只处理当前节点写入的任务（sticky polling）





# | 轻量级锁

- 乐观锁：基于版本号
- 排它锁：
  - 基于数据库：
  - 如果获取锁后线程挂掉
    - 超时机制：但有可能任务执行本来就很慢。
    - 栅栏令牌（fencing token）：返回锁 + 令牌，令牌类似乐观锁version？
  - 开源产品：`ShedLock`



# | 分布式限流系统

常用算法
- 令牌桶：支持突发流量
- 漏桶算法
- 固定窗口计数器
- 滑动窗口计数器

考点

- 限流规则配置化、功能开关
- 网关限流



# | TopK 反爬虫

功能要求：返回访问最频繁的 topK 客户 `topK(k, startTime, endTime)`

非功能需求：高性能，高可用，扩展，准确性

**单机方案**

- 保存 Hash: key = IP, value = count
- topK
  - 全排序 O(NLogN)
  - 堆 O(NlogK)

**分区方案**

- 分区服务：将请求IP Sharding到不同的”分区聚合主机“；后续 ”归并主机“ 进行归并、存入DB

**总体架构**

- API 网关：
  - 访问日志组件：记录访问日志；
  - 防爬虫组件：读取爬虫列表、拒绝请求；
- 访问日志采集服务：将日志写入 MQ
- 消费聚合服务：消费MQ，计算每分钟的TopK，存入DB
- 爬虫计算任务：根据DB数据 + 爬虫规则，计算出爬虫IP，存入DB
- 查询服务：查询当前爬虫



# | 数据迁移

**思路** 

- 双写 、读老库、异步对比
- 双写、读新库、异步对比
- 读写新库



**步骤**

**1. 数据同步**

Canal



**2. 双写**

- **结果要以旧库为准；**不能让新库影响到业务可用性和数据准确性
- **对比 / 补偿；**选择合适的时间窗口



**3. 灰度发布切到新库**



**原则**

- 确保每一步可快速回滚

- 确保不丢数据
  - 对比补偿



# | 促销数据及时更新

- 本地缓存，如何保持和库存数据同步
- 缓存更新：加锁？--> 如何实现限制最多N个请求回表？
- 数据库锁？
- 





# Reference

System Design Interview (印度口音)

- www.youtube.com/watch?v=bUHFg8CZFws

System Design Primer

- https://github.com/donnemartin/system-design-primer
- https://systeminterview.com/

