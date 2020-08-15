



# 系统设计案例

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



## 数据迁移





# Reference

System Design Interview (印度口音)

- www.youtube.com/watch?v=bUHFg8CZFws

System Design Primer

- https://github.com/donnemartin/system-design-primer

