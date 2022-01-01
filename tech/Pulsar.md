[toc]

# | 基础

## || 特点

- **云原生**
  - Broker 无状态
  - Bookie 可以水平扩展，新数据存入新的Bookie
- **支持多租户和海量Topic**
  - 便于做共享大集群
- **平衡可靠消息与性能**
  - 得益于 Quorum 机制、条带化写入策略
- **低延迟**
  - kafka topic增加，延迟也增加
- **高可靠、分布式**
  - Geo replication
- **轻量级函数式计算**
- **批流一体**
- **多协议**
  - KOP, AMQP, MQTT
- **功能丰富**
  - 延迟队列
  - 死信队列
  - 顺序消息
  - 主题压缩
  - 多租户
  - 认证授权
  - 分层存储
  - 跨地域复制



## || 架构

**代理层**

- 作用：请求转发
- 对外隐藏 Broker IP



**Broker 层**

- 负责业务逻辑



**Bookie 层**

- 负责数据存储
- BookKeeper: 只可追加数据的存储服务



https://pulsar.apache.org/docs/zh-CN/concepts-architecture-overview/



# | 客户端





# | Broker



# | BookKeeper



## || Ledger

> https://medium.com/splunk-maas/a-guide-to-the-bookkeeper-replication-protocol-tla-series-part-2-29f3371fe395 



Pulsar topic 由一系列数据分片（Segment）串联组成，每个 Segment 被称为 `Ledger`、并保存在 BookKeeper 服务器 `bookie` 上。

- 每个 ledger 保存在多个 bookie 上，这组 bookie 被称为 ensemble (?)；

- Ledger - bookie 对应关系存储在 zk；



### Ledger 生命周期

Pulsar broker 调用 BookKeeper 客户端，进行创建 ledger、关闭 ledger、读写 entry。

![image-20220101224253890](../img/pulsar/bookkeeper-ledger-lifecycle.png)

- 创建 ledger 的客户端（Pulsar broker）即为这个 ledger 的 owner；只有owner 可以往 ledger 写入数据。
- 如果 owner 故障，则另一个客户端会接入并接管。修复 under-replicated entry、关闭 ledger. —— open ledger 会被关闭，并重新创建新 ledger



### Ledger 状态

![image-20220101225318769](../img/pulsar/bookkeeper-ledger-status.png)

- Pulsar 一个主题只有一个 open 状态的 ledger；
- 所有写操作都写入 open ledger；读操作可读取任意 ledger；



### 写入 ledger

参数

- `Write Quorum (WQ)`：每份 entry 数据需要写入多少个 bookie，类似 replicas；
- `Ack Quorum (AQ)`：需要多少个 bookie 确认，entry 才被认为提交成功，类似 min-ISR；
- `Ensemble Size (E)`：可用于存储 ledger 数据的 bookie 数量；
- `Last Add Confirmed (LAC)`：水位线，达到 AQ 的最大 entry id. 
  - 高于此值：entry 未被提交；
  - 低于或等于此值：entry 已提交；



**Ledger Fragment**





### 读取 ledger

















读写概览：

![image-20211231232219900](../img/pulsar/bookkeeper-read-write-components.png)



## || 写入

![image-20211231232945352](../img/pulsar/bookkeeper-write-overview.png)

写入两个存储模块：

- **Journal**
  - 数据写入 Journal 后，触发 fsync，并返回客户端
- **Ledger**
  - 以异步方式批量刷盘





![image-20211231231801134](../img/pulsar/bookkeeper-write.png)

**写入流程**

- **Netty 线程**
  - 处理所有 TCP 连接、分发到 write thread pool
- **Write ThreadPool**
  - 写入 DbLedgerStorage 中的 `write cache`；成功之后再写入 Journal `内存队列`。
  - 默认线程数 = 1
- **Ledger**
  - 实际上有两个 `write cache`，一个接受写入、一个准备flush，两者互切。
  - `Sync Thread`：定时 checkpoint 刷盘
  - `DbStorage Thread`：Write thread 写入 cache 时发现已满，则向 DbStorage Thread 提交刷盘操作。
    - 此时如果 swapped out cache 已经刷盘成功，则直接切换，write thread写入新的cache；
    - 否则 write thread 等待一段时间并拒绝写入请求。
- **Journal**
  - `Journal 线程`循环读取内存队列，写入磁盘：group commit，而非每个entry都进行一次write系统调用
  - 定期向 `Force write queue` 中添加强制写入请求、触发 fsync；
  - `Froce Write Thread` ：循环从 froce write queue 中拿取强制写入请求（其中包含entry callback）、在 journal 文件上执行 fsync；
  - `Journal Callback Thread` ：fsync 成功后，执行 callback，给客户端返回 reesponse



**常见瓶颈**

- **Journal write / fsync 慢**：则 `Journal Thread `、 `Force Write Thread` 不能很快读取队列。

- **DbLedgerStorage 刷盘慢**：write cache 不能及时清空并互切。

- **Journal 瓶颈**：Journal 内存队列入队变慢，导致 `Write Thread Pool` 任务队列满、请求被拒绝。

  



## || 读取

读请求由 DbLedgerStorage 处理，一般会从缓存读取。

![image-20220101145034052](../img/pulsar/bookkeeper-read-overview.png)



读取流程：

- 读取 `Write Cache`

- 读取 `Read Cache` 

  > Read Cache 必须足够大，否则预读的entry会被频繁 evict

- 读取磁盘：

  - 找到位置信息：Entry Location Index (RocksDB)
  - 根据偏移量读取 entry 日志
  - 执行预读、写入 `Read Cache`



## || 背压

背压：通过一系列限制，防止内存占用过多。

> **Backpressure 指的是在 Buffer 有上限的系统中，Buffer 溢出的现象；它的应对措施只有一个：丢弃新事件。**
>
> 在数据流从上游生产者向下游消费者传输的过程中，上游生产速度大于下游消费速度，导致下游的 Buffer 溢出，这种现象就叫做 Backpressure 出现。
>
> Backpressure 和 Buffer 是一对相生共存的概念，只有设置了 Buffer，才有 Backpressure 出现；只要设置了 Buffer，一定存在出现 Backpressure 的风险。



![image-20220101153046466](../img/pulsar/bookkeeper-backpressure.png)



**1. In-Progress 写入总数**

- 配置 `maxAddsInProgressLimit`
- 超过后，Netty 线程会被阻塞



**2. In-Progress 读取总数**

- 配置 `maxReadsInProgressLimit`



**3. 每个 write thread 待处理的写入请求数**

- 配置线程池任务队列大小 `maxPendingAddRequestsPerThread`
- 超过后，客户端收到 TOO_MANY_REQUESTS，客户端选择另一个 bookie 写入



**4. 每个 read thread 待处理的读取请求数**

- 同上



**5. Journal 队列**

- 队列满之后，写入线程被阻塞



**6. DbLedgerStorage 拒绝的写入**

- write cache 已满，同时 swapped out write cache还未完成刷盘；则等待一段时间`dbStorage_maxThrottleTimeMs`，写入请求被拒绝



**7. Netty 不可写通道**

- 配置 `waitTimeoutOnResponseBackpressureMs`
- 当 channel 缓冲区满导致通道不可写入，写入响应会延迟等待 `waitTimeoutOnResponseBackpressureMs`，超时后不会发送响应、而只发出错误 metric；
- 而如果不配置，则仍然发送响应，这可能到时 OOM （如果通过channel发送的字节过大）





# | 运维

## || 安装

https://pulsar.apache.org/docs/zh-CN/standalone/ 



