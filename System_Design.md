





## 信息流服务

TBD





## 分布式计数服务

功能需求：

- 计数

- 查询

非功能需求：

- 规模：1w+ 次每秒观看记录；
- 性能：写入读取延迟 ms 级别；写入到读取分钟级延时；
- 高可用：无单点失败；

### 存储

数据库选型 : SQL 

- Replica 主从：解决高可用问题
- Sharding 分片：分摊负载
- 代理：ShardingSphere (ShardingJDBC) --> 路由 + Registry Center

数据库选型 : NOSQL

- Cassandra: gossip 让每个机器都知道集群信息；复制因子、仲裁（多数成功即可）
- 动态添加节点、

表设计：

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
- 好处：1、解耦，削峰；2、即便计数器服务下线维护，MQ仍然能保证请求不丢，后续仍能补充计数。
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



## MQ 系统设计

TBD





## 会话缓存服务

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



## 延迟任务队列

- 周期性扫描任务表：select * from tasks where alert_after < now。同时计算下一次执行时间。

- 开源：

  - `db-scheduler`  
  - `killbill notification queue` 
  - `quartz` 重量
  - `Xxl-job`
  - `cron-utils` 工具类

- 如何保证一个任务只被一个worker执行：

  - 1) 乐观锁：所有worker都能获取任务，但只有第一个写入的才能写入成功。`update version = version + 1 where version = ? ` 
  -  2) 每个节点只处理当前节点写入的任务（sticky polling）

  



## 轻量级锁

- 乐观锁：基于版本号
- 排它锁：
  - 基于数据库：
  - 如果获取锁后线程挂掉
    - 超时机制：但有可能任务执行本来就很慢。
    - 栅栏令牌（fencing token）：返回锁 + 令牌，令牌类似乐观锁version？
  - 开源产品：`ShedLock`



## 分布式限流系统

- 常用算法
  - 令牌桶：支持突发流量
  - 漏桶算法
  - 固定窗口计数器
  - 滑动窗口计数器
- 限流规则配置化、功能开关
- 网关限流



## TopK 反爬虫

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



## 数据迁移





# Reference

System Design Interview (印度口音)

- www.youtube.com/watch?v=bUHFg8CZFws

System Design Primer

- https://github.com/donnemartin/system-design-primer

