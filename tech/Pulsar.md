[toc]

# | 基础

## || 特点

- **云原生**
  - Broker 无状态
  - Bookie 可以水平扩展，新数据存入新的Bookie
- **支持多租户和海量 Topic**
  - Tenant + Namespace 逻辑隔离
  - 便于做共享大集群
- **平衡消息可靠性与性能**
  - 得益于 Quorum 机制、条带化写入策略
- **低延迟**
  - kafka topic 增加，延迟也增加
- **高可靠、分布式**
  - Geo replication
- **轻量级函数式计算**
  - Pulsar Function: source --> sink
- **批流一体**
  - Segment 分片存储：方便支持流计算
  - 分层存储：方便支持批处理
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

> https://pulsar.apache.org/docs/zh-CN/concepts-architecture-overview/



**代理层**

- 作用：请求转发
- 对外隐藏 Broker IP



**Broker 层**

- 负责业务逻辑
- 提供管理接口



**Bookie 层**

- 负责数据存储
- BookKeeper: 只可追加数据的存储服务



ZK

- 存储元数据

  > Web-service-url 管理流地址
  >
  > Broker-service-url 数据流地址
  >
  > Ledger信息
  >
  > Topic 信息 （非持久化 NonPartitionedTopic 无需存储到zk）

- Broker选主

- 分布式锁

  





# | 客户端

## || Producer

https://pulsar.apache.org/docs/en/concepts-messaging/

配置

- **Synd Mode**
  - `sync send`: waits for an acknowledgement from the broker after sending every message.
  - `async send`: puts a message in a blocking queue and returns immediately.

- **ProducerAccessMode**

  - `Shared`: Multiple producers can publish on a topic.

  - `Exclusive`: Only one producer can publish on a topic.

  - `WaitForExclusive`: If there is already a producer connected, the producer creation is pending (rather than timing out) until the producer gets the `Exclusive` access. 

    > 类似选主。if you want to implement the leader election scheme for your application, you can use this access mode.
  
- **MessageRoutingMode**: 对于 PartitionedTopic，决定消息被发送到哪个分区/Broker

  - `SinglePartition`: 如果消息有key，则对key哈希找分区；无key，则发送到某个特定分区（可以实现全局有序）；
  - `RoundRobinPartition`: 如果无key，则轮询发到各个分区；
  - `CustomPartition`: 自定义路由。



实现类

- **ProducerImpl**
  - HandlerState 状态机
  - 发送流程
    - 同步发送底层也是调用异步发送，然后Future.get()
    - 压缩
    - 分块
    - 校验并设置元数据，包括 sequenceId
    - 进入批量发送队列
    - Flush：直接触发、或等待
- **PartitionedProducerImpl**
  - 内有保存 ProducerImpl List，每个partition对应一个



功能

- **Batching**

  - 最大消息个数

  - 最大发送延迟

  - batch是个整体

    - batches are tracked and *stored* as single units rather than as individual messages.
    - Consumer unbundles a batch into individual messages. In general, a batch is acknowledged when all of its messages are acknowledged by a consumer. 
    - 消费的时候，一批消息只会被同一个consumer消费。
  - 一个消息ack失败，会导致整个batch重发。2.6.0 之后引入 batch index acknowledgement 解决重复发送问题：消费者发送 batch index ack request. 
    
    

- **Chunking**

  - 作用：生产者将大payload拆分、消费者组装

  - 1. The producer splits the original message into chunked messages and publishes them with chunked metadata to the broker separately and in order.
    2. The broker stores the chunked messages in one managed-ledger in the same way as that of ordinary messages, and it uses the `chunkedMessageRate` parameter to record chunked message rate on the topic.

    3. The consumer buffers the chunked messages and aggregates them into the receiver queue when it receives all the chunks of a message.

    4. The client consumes the aggregated message from the receiver queue.

  - 限制

    - 不能同时使用  batching；
    - 仅支持持久化主题；
    - 仅支持 exclusive / failover 订阅模式，要保证消息被同一个消费者消费。



- **重试机制**

  - 不会重试的场景

    - Topic 已关闭：producer 收到此异常后会把自己关闭；
    - 服务端返回 NotAllowedError：客户端直接抛出异常；
    - 超时：TCP 连接级别的超时抛出异常；

  - 重试的场景

    - 消息校验和不正确：缺失或被篡改；
    - 逻辑重连：Bundle 卸载、分裂等导致主题归属转移；
    - 无法获得连接：

    

- **Deduplication**

  - 效果：生产者多次发送同样的消息，只会被保存一次到bookie。

  - 实现：

    - 生产者每条消息会设置一个元数据 `sequenceId`，broker遇到比之前小的ID则可过滤掉。
    - 生产者重连后，会从Broker拿到当前topic最后的`sequenceId`，继续累加。

  - 配置：`brokerDeduplicationEnabled=true`

    > https://pulsar.apache.org/docs/en/cookbooks-deduplication/ 
    
  - 可用于 effectively-once  语义

    > https://www.splunk.com/en_us/blog/it/exactly-once-is-not-exactly-the-same.html 
    
    

- **消息顺序**

  - `业务线程` 的影响：多个线程同时持有一个 Producer 对象；
  - `路由模式` 的影响：SinglePartition 有序、RoundRobinPartition 无序；
  - `分区` 的影响：如果只有一个分区，则同SinglePartition模式，能保证有序；例外：异步发送失败后重试；
  - `发送方式` 的影响：同步 vs 异步
  - `批量发送` 的影响: 一批消息是原子性。但批与批之间的顺序可能乱；
  - `消息key` 的影响：相同key会被发到同一分区，能保证有序性；

  

- **Deleyed Message Delivery**

  - 作用：发送时，可以给每个消息配置不同的延迟时间。在一段时间之后消费消息，而不是立即消费。

    ```java
    // message to be delivered at the configured delay interval
    producer.newMessage()
      .deliverAfter(3L, TimeUnit.Minute)
      .value("Hello Pulsar!")
      .sendAsync();
    
    // message to be delivered at future timestamp
    producer.newMessage()
      .deliverAt(long timestamp)
      .value("Hello Pulsar!")
      .sendAsync();
    ```

    

  - **原理：**

    - 消息存储到 BookKeeper后，`DelayedDeliveryTracker` 在堆外内存**优先级队列**中维护索引 (time -> messageId (LedgerId + EntryId))
    - 当消费时，如果消息为delay，则放入`DelayedDeliveryTracker` 

    ![image-20220327191641313](../img/pulsar/pulsar-delayed-msg.png)

  - 注意：只能作用于 shared mode

  - **限制：**

    - 内存占用：delayed index memory limitation.

    - Broker宕机后需要重建索引、新Broker有一段时间会无响应：rebuilding delayed index. 

      > 增加分区可缓解，让每个分区的数据尽可能小。

    - 索引是在subscription维度，可能有重复：the index only available for a subscription.
      

  - **优化 - PIP26 Hieraychical Timing Wheels**

    > http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf 
    >
    > https://blog.acolyer.org/2015/11/23/hashed-and-hierarchical-timing-wheels/

    - 自定义延迟精度，并分片；只有最近的分片存储在内存、其他的持久化
      ![image-20220327195558877](../img/pulsar/pulsar-delayed-msg-plan.png)
    - 取到M9时，会查看时间片 time-partition-0 是否有消息到期。

  - **挑战**

    - 如何清理延时消息？
    - Safe position for subscription to start read?
    - 内存里需要维护 too much individual acks, introduced much memory overhead. 



## || Consumer



配置

- **Receive Mode**
  - `Sync Receive`: blocked until a message is available.
  - `Async Receive`: returns immediately with a future value.
- **subscriptionMode**
  - `durable`：在Broker生成持久化的cursor
  - `nonDurable`
- **subscriptionType**
  - Exclusive, Shared, Failover, Key_Shared



实现类

- **ConsumerImpl** 

  - 初始化

    - ConsumerStatsRecorder：记录消费者metrics信息。
    - Trackers：
      `UnAckedMessageTracker` 记录接收未确认的消息，用于管理后续重投递；
      `AcknowledgementsGroupingTracker` 批量确认管理；
      `NegativeAcksTracker` negative ack 管理；

  - 异步发送 Lookup 请求

    - 确认主题归属的 Broker

  - 获取连接

    - 复用 PulsarClient 连接池；

  - 发送订阅命令

  - 发送 `FlowPermits` 命令

    - 通知Broker推送数据，默认推送的消息个数是 ReceiverQueue 大小；
    - 然后消费者**缓存**预拉取消息到 ReceiverQueue；

    > 是一种“经过优化的 Poll 模式”：
    >
    > - 当 ReceiverQueue 中的消息少于一半时，消费者重新触发 FlowPermits 命令，要求Broker推送消息；
    > - 既能避免 Push 模式下消费能力不足的问题，又能提升消息消费的及时性。

- **MultiTopicsConsumerImpl**

  - 用于消费多分区 PartitionedTopic
  - 内部引用多个 ConsumerImpl

- **PatternMultiTopicsConsumer**

  - 用于用正则表达式订阅主题时；类似 MultiTopicsConsumerImpl，并实时更新订阅主题的变化。
  - 定时任务通过 Lookup 查到namespace下的所有主题，再通过正则筛选。

- **ZeroQueueConsuemrImpl**

  - 不会预拉取消息；适用于消息不多但单条消息要消费很久的场景。
  - 不可用来消费batch消息。

- **RawConsumerImpl**

  - 直接通过 bk client 读取和写入消息；
  - 仅用于pulsar内部，例如实现压缩主题。



消费流程

- 同步接收
  - 直接从 ReciverQueue 中 take()，同步等待
- 异步接收
  - 用户业务线程
  - Netty IO 线程

![image-20220403134548890](../img/pulsar/consume-flow.png)



功能

- **Acknowledgement**

  - **确认场景**

    - 单条消息确认：`consumer.acknowledge(msg);`
    
  - 累积消息确认：`consumer.acknowledgeCumulative(msg);`
    
    - 批量消息中的单个消息确认：Broker 配置 `acknowledgementAtBatchIndexLevelEnabled=true`

    - 否定应答: 表示处理失败、稍后重发给其他消费者。`consumer.negativeAcknowledge(msg);`
    
    > Q: 何时重新deliver、能否指定? 
      > A: 全局设置延迟时间；如有大量消息延迟消费，可调用 `reconsumerLater` 接口。

      

  - **确认流程**

    - 待确认的消息先放入`AcknowledgementsGroupingTracker`缓存，默认每100ms、或大小超过1000则发送一批确认请求；目的是避免broker收到高并发的确认请求。
    - 对于 ack at batch index level，存储格式为Map<Batch MessageId, `BitSet`>；
    - 对于 累积消息确认，Tracker 只需保存最新确认位置即可。
    - 对于否定应答，由 `NegativeAcksTracker`处理，其复用 Pulsar Client 时间轮，定期发送给 Broker。

    

  - **Ack timeout**: 可对unack消息自动重发。

    - 情况一：业务端调用 receive() 后：消息进入 `UnAckedMessageTracker`，其维护一个时间轮，超时后发送 redeliverUnacknowledgedMessages 命令给broker。

    - 情况二：消费者做了预拉取，但尚未调用 receive()：Broker侧将这些消息标记为 `pendingAck` 状态，除非当前消费者被关闭，才会被重新投递。

      > - Broker RedeliveryTracker 会记录每个消息的投递次数；
      > - 可知如果消费者 ReceiverQueue 设置过大 是会对Broker有影响的。

  

- **reconsumeLater**

  - 作用：把消息发送到重试队列、后续再消费。
  - 原理：
    - 消费者创建时，指定 DeadLetterPolicy，包含 `maxRedeliverCount` `retryLetterTopic` `deadLetterTopic`
    - 消费者自动创建对应重试队列、死信队列的内部 Producer；并同时偷偷地订阅“重试队列”。
    - 当未超过最大重试次数，发送到重试队列；否则发送到死信队列。
  - 注意：重试队列不是延迟队列，会立即被消费。如果设置了延迟多久reconsumeLater，则会被投递到延迟队列。



- **Dead Letter Topic**

  - 将消费失败的消息存到 dead letter `<topicname>-<subscriptionname>-DLQ`，可以自定义如何处理死信消息。

  - 配置处理策略：

    ```java
    Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
                  .topic(topic)
                  .subscriptionName("my-subscription")
                  .subscriptionType(SubscriptionType.Shared)
                  .deadLetterPolicy(DeadLetterPolicy.builder()
                        .maxRedeliverCount(maxRedeliveryCount)
                        .deadLetterTopic("your-topic-name")
                        .build())
                  .subscribe();      
    ```

  - 在 negative ack 和 ack timeout 时放入？



- **Retry Letter Topic** 

  - When **automatic retry** is enabled on the consumer, a message is stored in the retry letter topic if the messages are not consumed, and therefore the consumer automatically consumes the failed messages from the retry letter topic after a specified delay time.
    `consumer.reconsumeLater(msg,3,TimeUnit.SECONDS);`

  - 配置消费 retry letter:

    ```java
    Consumer<byte[]> consumer = pulsarClient.newConsumer(Schema.BYTES)
                    .topic(topic)
                    .subscriptionName("my-subscription")
                    .subscriptionType(SubscriptionType.Shared)
                    .enableRetry(true)
                    .receiverQueueSize(100)
                    .deadLetterPolicy(DeadLetterPolicy.builder()
                            .maxRedeliverCount(maxRedeliveryCount)
                            .retryLetterTopic("persistent://my-property/my-ns/my-subscription-custom-Retry")
                            .build())
                    .subscriptionInitialPosition(SubscriptionInitialPosition.Earliest)
                    .subscribe();
    ```

  

## || Reader

Reader 包装了 Consumer，拥有Consumer的所有功能。

- 特点
  - Reader 强制使用 Exclusive 订阅类型；
  - Reader 订阅模式是 NonDurable，即没有持久化的游标。

- 示例

  ```java
  Reader<byte[]> reader = pulsarClient
    .newReader()
    .topic("my-topic")
    .startMessageId(MessageId.earliest)
    .create();
  
  while (reader.hasMessageAvailable()) {
    reader.readNext();
  }
  ```



## || Subscription

**订阅类型**

![image-20220322112234641](../img/pulsar/subscription-modes.png)

- **Exclusive**

  - 只有一个消费者绑定到当前订阅。其他消费者需要使用不同的 subscription name，否则报错。

  - 支持单条消息确认、累积消息确认。

  - > ![image-20220322112520553](../img/pulsar/subscription-modes-exclusive.png)

- **Failover**

  - 多个消费者可以绑定到当前订阅（而不像 Exclusive 那样直接报错），但只有一个收到消息。

    > ![image-20220322112744805](/Users/alpha/dev/git/alpha/alpha-notes/img/pulsar/subscription-modes-failover.png)

- **Shared** 

  - 多个消费者可以绑定到当前订阅，按 round-robin 模式接收消息（消费者可设置 priorityLevel 来提升自己的优先级）。

  - 限制：

    - 不保序、
  - 无法使用 cumulative ack. 
  
- > ![image-20220322113010001](/Users/alpha/dev/git/alpha/alpha-notes/img/pulsar/subscription-modes-shared.png)
  
- **Key_Shared** 

  - 多个消费者可以绑定到当前订阅，按相同key模式接收消息。

  - In *Key_Shared* mode, multiple consumers can attach to the same subscription. Messages are delivered in a distribution across consumers and message with same key or same ordering key are delivered to only one consumer. 

  - No matter how many times the message is re-delivered, it is delivered to the same consumer. When a consumer connected or disconnected will cause served consumer change for some key of message.

  - 限制

    - 必须指定 key，或orderingKey
    - 消费者无法使用 cumulative ack
    - 生产者必须禁用 batching，或者使用 *key-based batching*

  - > ![image-20220322113231815](/Users/alpha/dev/git/alpha/alpha-notes/img/pulsar/subscription-modes-key-shared.png)





## || Topic

命名：`{persistent|non-persistent}://tenant/namespace/topic`



类型

- Persistent
- Non-persistent



**Namespace**

-  The administrative unit of the topic, which acts as a grouping mechanism for related topics. 
-  Most topic configuration is performed at the [namespace](https://pulsar.apache.org/docs/en/concepts-messaging/#namespaces) level. 
-  Each tenant has one or multiple namespaces.



**Partitioned Topic**

- 普通 Topic 只对应一个broker，限制了吞吐量；而 Partitioned Topic 则可被多个 broker 处理、分担流量压力；
- 实现：N 个内部主题。
- routing mode: 决定生产到哪个分区；
  - RoundRobinPartition
  - SinglePartition：随机
  - CustomPartition
- subscription mode: 决定从哪个分区读取；



**Non-persistent Topic**

- 普通 Topic 存储消息到 bookie，而non-persistent topic则只存到内存
- 更快



## || 客户端通用能力

**连接管理**

- 客户端与每个broker只建立一个连接，但可配置。
- ConnectionPool 利用 ConcurrentHashMap 保存连接，key = Broker IP. 



**线程池管理**

- Pulsar 优化了原生线程池：ExecutorProvider 内部创建了多个原生 `ScheduledExecutorService`，每个线程池max thread = 1

- 优点：

  - 实现线程池隔离；
  - 保证线程安全、实现去锁。getExecutor(Object obj) 只要传入对象一样，则每次拿到的线程都是同一个。 

  

**LookupService**

- 目的
  - 动态服务发现：获取主题的归属broker。
  - 元数据查询：分区信息、Schema信息、某个 NS 下的所有主题。



**内存限制**

MemoryLimitController

- 目的：控制客户端内存使用量，避免例如 ReceiverQueueSize 不合理、大量 ConsumerImpl 带来的过度内存占用。

  





# | Broker

Broker 是 Bookie 的客户端。

## || 主题归属



- **ZK 存储**

  > 不能直接在 zk 上存储 topic - broker 归属关系：否则数据量太大

  - ZK 只保存 Bundle 与 Broker 之间的联系。
  - Topic 归属哪个 broker 是通过一致性哈希动态计算出来的。

- **Topic 归属的计算步骤** `ServerCnx#handleLookup`

  - 根据 namespace 找到其所有的 Bundle；

  - 计算 Topic 所属的 Bundle：一致性哈希算法；

  - 确定 Bundle 归属哪个 Broker，先找到`裁判Broker`，其通过 loadManager 查找负载最低的 Broker 并把Bundle 分配给他

    > 裁判 Broker:
    >
    > - 优先选择 Heartbeat、SLAMonitor 所在的broker；
    > - 如果 loadManager 使用的是中心化策略，则需要 Leader 裁判；
    > - 如果 loadManager 使用的是非中心化策略，则当前Broker即可裁判；

  - 客户端请求到归属 Broker，该Broker会尝试在 zk 写入一个节点，如果写入失败则说明 Bundle 被别的 Broker 抢到了。

  

- **Topic 的迁移**

  - 作用：
    - 当 Broker 扩容后，将现有的topic迁移到新 Broker上。
    - 或者当 Broker负载不均衡时，把高负载broker上的部分bundle转移到低负载broker。
  - Q: 如果无论迁移到哪个Broker都无法承载topic的负载？
    - 支持 split bundle （线上建议关闭）
    - Bundle分裂，重新进行一致性哈希，将**部分** topic 转移到新的 Broker上。



## || 压缩主题



- 作用：相同key的消息，只保存最新的值。
- 触发
  - CLI: `pulsar-admin topics compact {name}`
  - 自动：定时线程检查积压的消息是否达到阈值。
- 实现
  - 接收到消息后，在原始topic创建Ledger时，同时在内部创建一个 `CompactedTopic` 对象；
  - 每个 Broker 上有一个单线程调度器 `compactionMonitor`，定时触发当前 Broker上所有 topic的 `checkCompaction`方法；每个Topic单独创建 `RawReader`直接从 bk 读取原始主题的数据；RawReader 会创建一个 `CompactorSubscription`。
  - 压缩阶段一：遍历原始主题，在内存中保存每个消息的 key + MessageId，只保留最新记录。
  - 压缩阶段二：根据阶段一的MessageId 再读一次数据，写入新的压缩 Ledger。
- Q: 压缩是异步进行的，那如何保证读取时总是读到最新值呢？
  - 如果要读取的 MessageId 位于游标位置之前，则从压缩主题读取；
  - 如果位于游标位置之后，则证明尚未被压缩，则从原始主题读取；







# | BookKeeper

> https://www.bilibili.com/video/BV1T741147B6?p=5 



## || 架构

### **概念**

> https://bookkeeper.apache.org/docs/latest/getting-started/concepts/ 

- **Entry**：一条日志记录。each unit of a log is an *entry*; 
  - 包含内容包含 metadata 和 data
    - Ledger Id
    - Entry Id
    - LC, Last Confirmed：上次记录的 entry id
    - Digest：CRC32
  - Data byte[]
  - Authentication code
- **Ledger**：一组日志记录，类比一个文件。streams of log entries are called *ledgers*
  - 打开/关闭 Leger 只是操作`元数据`(元数据存储在zk)：
    - State: open/closed
    - Last Entry Id: -1L
    - Ensemble 
    - WriteQuorum
    - ReadQuorum Size
- **Bookie**：存储 ledger的服务器。individual servers storing ledgers of entries are called *bookies*
  - 每个 bookie 存储部分 ledger *fragment*, 而非完整ledger



### 组件

**Metadata Store**

- Zk / etcd
- 存储 ledger 元数据
- 服务发现

> 可用性：如果zk挂了，已打开的Leger可以继续写，但create/delete ledger 会报错



#### **Bookie**

- 而 Bookie 逻辑很轻量化
- 可当做是一个 KV 存储
  - `(Lid, Eid) --> Entry `
- 操作：Add / Read

![image-20220326130700495](../img/pulsar/bk-arch-rw-isolation.png)

> Q: 先写Write Cache，再 flush 到 Journal，会不会导致读到脏数据？
>
> A: 不会，LAC 之后的都读不到



**Bookie 组件**

- **Journal**
  - 事务日志文件。在修改 ledger 之前，先记录事务日志。
    - 所有写操作，先**顺序写入**追加到 Journal，*不管来自哪个 Ledger*。
    - 写满后，打开一个新的 Journal 
  - 作用：
    - 写入速度快（顺序写入一个文件，没有随机访问）、读写存储隔离
    - 相当于是个**循环 Buffer**. 
  - 配置：
    - JournalDirectories: 每个目录对应一个Thread，给多个Ledger开多个directory可提高写入SSD的吞吐。
  - ![image-20220326234533697](../img/pulsar/bk-arch-comp-journal.png)
- **Write Cache**
  - JVM 写缓存，写入 Journal 之后将Entry放入缓存，并按 Ledger 进行**排序**（基于 Skiplist），方便读取。
  - 缓存满后，会被 Flush 到磁盘：写入 Ledger Directory （类比 KV 存储）
  - Flush 之后，Journal 即可被删除
  - ![image-20220326234645526](../img/pulsar/bk-arch-comp-writecache.png)
- **Ledger Directory**
  - **Entry log**
    - An entry log file manages the written entries received from BookKeeper clients. 
    - Entries from different ledgers are aggregated and written sequentially, while their offsets are kept as pointers in a ledger cache for fast lookup.
    - 其实就是 write cache的内容
  - **Index file**
    - 每个 ledger 有一个 index 文件
    - entryId --> position 映射
  - ![image-20220326235132492](../img/pulsar/bk-arch-comp-entrylog.png)



**三种文件类型**

- Journal：建议用SSD

- Entry log

- Index file




#### **Client**

> 对 Pulsar 来说，此 client 为 Broker.

- 胖客户端：外部共识；

  - 例如 EntryID 是由客户端生成 ，前提：一个 Ledger 只有一个 Writer

- 功能

  - 指定 WQ、AQ

  - 保存 LAP：Last Add Push，发出的请求最大值

  - 保存 LAC：Last Add Confirm，收到的应答最大值

    - 客户端需要保证不跳跃，例如收到 3 的ack、但未收到 2 的ack

      > 如果一直收不到2: timeout，执行 ensemble change、重发2 & 3

    - LAC 之前的 entry 一定已被存储成功。

    - LAC 保存未 entry 的元数据，而不必存到zk 

  - Ensemble Change

    - 处理当某个bookie宕机：
      - 新的entry可能存到新的 bookie
      - 对于已宕机bookie里存储的数据，如何修复：TBD



> Q: 读取时如何找到entry存在哪个bookie?
>
> - 任何一台 bookie 都可以响应？







### WQ/AQ

**节点对等架构**

![image-20220326124934997](../img/pulsar/bk-arch-openLedger.png)

- 胖客户端决定：openLedger(`Ensemble`, `Write Quorum`, `Ack Quorum`)

  - Ensemble：组内节点数目，用于分散写入数据到多个节点；**控制一个 Ledger 的读写带宽**
  - Write Quorum：数据备份数目；**控制一条记录的副本数量**；
  - Ack Quorum：等待刷盘节点数目；**控制写入每条记录需要等待的 ACK 数量**；

- 灵活性配置

  - 增加 Emsemble：**增加读写带宽**
  - WQ = AQ，等待所有Write Quorum的ack：**提供强一致性保障**
  - 减少 QA：**减少长尾时延**

  

- 示例
  
  - E = 5, WQ = 3
  - 对于指定 entryId，对5取模，决定存到哪三个 bookie；效果是按 Round Robin选择
  - ![image-20220326125340028](../img/pulsar/bk-arch-openLedger-eg.png)



### 高可用



**读写高可用**

- **读高可用：Speculative Reads**
  - 原因：对等副本都可以提供读
  - 通过Speculative 减少长尾时延：同时发出两个读，哪个先返回用哪个
    Q：放大了读取请求数？
- **写高可用：Ensemble Change**
  - 最大化数据放置可能性

![image-20220326130000834](../img/pulsar/bk-arch-rw-ha.png)



**Bookie 高可用**

> 某个 Bookie 宕机后如何处理。--> Auto Recovery
>
> 注意：区别于broker宕机 --> Fencing



- Auditor

  - 审计集群里是否有 bookie宕机；(ping bookies)
  - 审计某个Ledger是否有entry丢失；

- 流程

  - 如果 bookie1宕机， auditor 找出该bookie存储的所有 ledger；

  - 新的 bookie 替换原有的 ensembler，复制原ledger entries；

    





### 外部共识 

Consensus：一个ledger任何时候都不会有两个broker写入。



**要点：**

- LastAddPushed
- LastAddConfirmed
- Fencing 避免脑裂

![image-20220326130309827](../img/pulsar/bk-arch-consistency.png)

**实现：**

- 一个ledger只会由一个broker负责写入、写满即关闭；

  - 当broker宕机，新的broker会写入新的ledger，而不会操作老ledger；
  - 即：一个ledger任何时候都不会有两个broker写入。

- **Fencing**

  - 场景：broker1 与zk出现了网络分区，zk把broker2设为新的topic owner，新broker发现ledger处于open状态，则在接管前需要进行 recovery操作。

  - **流程：**

    - 1. Recover Ledger-X

      - 1.1 Fence Ledger-X ==> Retrun LAC

        > 新broker对原来的 ensembler 发起 fencing 操作：将bookie状态设为fenced，则往原ledger的新的写入会失败，发生 LedgerFencedException；-- 防止原来的broker1还在继续写。

      - 1.2 Forward reading from LAC until no entry is found.

        > 尝试找 LAC + 1 的entry，如果其已经写入部分 WQ 但尚未达到 AQ，则进行复制以便达到AQ，修复 LAC + 1。

      - 1.3 Update the ledger metadata.

        > 关闭原 Ledger-X

    - 2. Open a new ledger to append. 

  - **对比 Kafka ISR**

    - Under min ISR 会导致写入失败，客户端需要等待broker达成一致（主从复制）。
    - 还要考虑如果 unclean leader election，会有truncate，可能数据丢失。
    - 而 Pulsar 的recovery 特别容易。



**类似 Raft 一致性协议**

![image-20220326130523922](../img/pulsar/bk-arch-consistency-raft1.png)

日志复制过程：

- Q: Writer 相当于 Leader?

![image-20220326130610241](../img/pulsar/bk-arch-consistency-raft2.png)











## || Ledger

> - https://medium.com/splunk-maas/a-guide-to-the-bookkeeper-replication-protocol-tla-series-part-2-29f3371fe395 
>
> - https://medium.com/splunk-maas/apache-bookkeeper-insights-part-2-closing-ledgers-safely-386a399d0524 //TODO



Pulsar topic 由一系列数据分片（Segment）串联组成，每个 Segment 被称为 `Ledger`、并保存在 BookKeeper 服务器 `bookie` 上。

- 每个 ledger 保存在多个 bookie 上，这组 bookie 被称为 ensemble；

- Ledger - Bookie 对应关系存储在 zk；



### Ledger 生命周期

Pulsar broker 调用 BookKeeper 客户端，进行创建 ledger、关闭 ledger、读写 entry。

![image-20220101224253890](../img/pulsar/bookkeeper-ledger-lifecycle.png)

- 创建 ledger 的客户端（Pulsar broker）即为这个 ledger 的 owner；**只有owner 可以往 ledger 写入数据**。
- 如果 owner 故障，则另一个客户端会接入并接管。修复 under-replicated entry、关闭 ledger. —— open ledger 会被关闭，并重新创建新 ledger



> 对于 Pulsar，
>
> - 每个 topic 有一个 broker 作为 owner（注册于 zk）。该 broker 调用 BookKeeper 客户端来创建、写入、关闭 broker 所拥有的 topic 的 ledger。
> - 如果该 owner broker 故障，则ownership 转移给其他 broker；新 broker 负责关闭该topic最后一个ledger、创建新 ledger、负责写入该topic。
>
> ![image-20220102204423380](../img/pulsar/broker-failure-ledger-segment.png)



**Ledger 状态**

![image-20220101225318769](../img/pulsar/bookkeeper-ledger-status.png)

- **Pulsar 一个主题只有一个 open 状态的 ledger；**
- 所有写操作都写入 open ledger；读操作可读取任意 ledger；



### 写入 ledger

**参数**

- `Write Quorum (WQ)`：每份 entry 数据需要写入多少个 bookie，类似 *replicas*；

- `Ack Quorum (AQ)`：需要多少个 bookie 确认，entry 才被认为提交成功，类似 *min-ISR*；

- `Ensemble Size (E)`：可用于存储 ledger 数据的 bookie 数量；

- `Last Add Confirmed (LAC)`：水位线，达到 AQ 的最大 entry id.  --> 类似 Kafka 高水位。

  > Bookie 本身并不存储 LAC，而是请求数据中包含最新 LAC

  - 高于此值：entry 未被提交；
  - 低于或等于此值：entry 已提交；



**Ledger Fragment**

- Leger 本身可以分成一个或多个 Fragment。
- 创建 Ledger 时，包含一个 Fragment，由一组bookie存储（it consists of a single fragment with an ensemble of bookies）
- 当写入某个 bookie 失败时，客户端用一个新的 bookie 来替代，创建新 Fragment（with a new ensemble）、重新发送未提交 entry 以及后续 entry
- Fragments 又称为 Ensembles ?



### 读取 ledger

四种读取类型

- **Regular entry read**
  - 从任意 bookie 节点读取；如果读取失败，从 ensemble 中换个bookie 继续读取。
- **Long poll LAC read**
  - 读取到 LAC 位置后，即停止读取、并发起 long pool LAC read，等待有新的 entry 被提交。
- **Quorum LAC read**
  - 用于恢复
- **Recovery read**
  - 用于恢复





使用的 Quorum：

- **Ack Quorum (AQ)**

  - 主要用于写入

- **Write Quorum (WQ)**

  - 主要用于写入

- **Quorum Coverage (QC)** = `(WQ - AQ) + 1`

  - 主要用于恢复过程
  - QC cohort 是单个 entry 的写入集合，QC 当需要保证单个 entry 时有用。
  - A given property is satisfied by at least one bookie from every possible ack quorum within the cohort.
  - There exists no ack quorum of bookies that do not satisfy the property within the cohort. 
    

- **Ensemble Coverage (EC)** = `(E - AQ) + 1`

  - 主要用于恢复过程
  - EC cohort 是当前fragment的bookie集合，EC 当需要保证整个 fragment 时有用。

  

> *Bookies that satisfy property = (Cohort size — Ack quorum) + 1*



### 恢复 ledger

> https://medium.com/splunk-maas/apache-bookkeeper-insights-part-2-closing-ledgers-safely-386a399d0524 



**何时触发  recovery?** 

- 每个 ledger 都有一个客户端作为 owner；如果这个客户端不可用，则另一个客户端会接入执行恢复、并关闭该 ledger。
- Pulsar：topic owner **broker 不可用**，则另一个broker接管该topic的所有权。



**防止脑裂**

- 恢复过程可能出现脑裂：客户端A (pulsar broker) 与zk断开连接，被认为宕机；触发恢复过程，由另一个客户端B来接管 ledger并恢复ledger；则有两个客户端同时操作一个 ledger。--> 可能导致数据丢失！
- **Fencing**: 客户端B 尝试恢复时，先将 ledger 设为 fence 状态，让 ledger 拒绝所有新的写入请求（则原客户端A写入新数据时，无法达到 AQ 设定的副本数）。一旦足够多的 bookie fence了原客户端A，恢复过程即可继续。



**恢复过程**

- **第一步：Fencing**

  > 将 Ledger 设为 fence 状态，并找到 LAC。

  - 新客户端发送 Fencing 请求：Ensemble Coverage 的 LAC 读取请求，请求中带有 fencing 标志位。
  - Bookie 收到这个 fencing 请求后，将 ledger 状态设为 fenced，并返回当前 bookie 上对应 ledger 的 LAC。
  - 一旦新客户端收到足够多的响应，则执行下一步。
    - 无需等待所有 bookie 响应，只需保证剩下的未返回 bookie 数 < AQ 即可。这样原客户端一定无法写入 AQ 个节点、亦即无法写入成功。
    -  即，收到的响应数目达到 **Ensemble Coverage** 即可：`EC = (E - AQ) + 1`

> 为什么要找到 LAC？
>
> The LAC stored in each entry is generally trailing the real LAC and so finding out the highest LAC among all the bookies is the starting point of recovery.



- **第二步：Recovery reads & writes**

  > Learning the highest LAC is only the first step, now the client must find out if there are more entries that exist beyond this point.
  >
  > 确保在关闭 ledger之前，任何已提交 entry 都被完整复制。

  - 客户端从 LAC + 1 处开发发送 `recovery 读请求`，读到之后将其重新写入 bookie ensemble（写操作是幂等的，不会造成重复）。重复这个过程，直到客户端读不到任何 entry。
  - `recovery 读请求`：与regular读不同，需要 **quorum**；每个 recovery 读请求决定 entry 是否已提交、是否可恢复：
    - 已提交 = Ack Quorum 返回存在响应
    - 未提交 = **Quorum Coverage** 返回不存在响应：`QC = (WQ - AQ) + 1`
    - 如果所有响应都已收到，但以上两个阈值都未达到，则无法判断是否已提交；这时会重复执行恢复过程，直至明确状态。

  > 1. 可否完全不等待 bookie 响应？
  >
  > NO，否则会导致 ledger truncation：Last Entry Id 设置得过低，导致已提交的 entry 无法被读取。
  >
  > 2. AQ = 1 带来的问题
  >
  > - 存储 entry 时没有冗余；
  > - 导致 recovery 过程卡住：必须等待所有 bookie 返回

  

  ![image-20220102184102357](../img/pulsar/bookkeeper-recovery-readwrite.png)



- **第三步：关闭 Ledger**

  > 一旦所有已提交 entry 都被识别并被修复，客户端会关闭 ledger；

  - 更新 zk 上的 ledger 元数据，将状态设为 CLOSED、将 `Last Entry Id` 设为最高的已提交 entry id。

  - 在 bookie ensemble 中找到一起交的最高 entry id，确保每个 entry 已被复制到 Write Quorum。

  - 新客户端关闭 ledger，将状态置为 CLOSED，将 Last Entry ID 设置为最高的已提交 entry（即 `LAC，Last Added Confirmed`）；`Last Entry ID` 表示ledger的结尾，其他客户端来读取时，永远不会超过此 Last Entry Id。



## || 读写流程

读写概览：

![image-20211231232219900](../img/pulsar/bookkeeper-read-write-components.png)



### 写入

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
  - 先写入 DbLedgerStorage 中的 `write cache`；成功之后再写入 Journal `内存队列`。
  - 默认线程数 = 1
- **Ledger**
  - 实际上有两个 `write cache`，一个接受写入、一个准备flush，两者互切。
  - `Sync Thread`：定时 checkpoint 刷盘
  - `DbStorage Thread`：Write thread 写入 cache 时发现已满，则向 DbStorage Thread 提交刷盘操作。
    - 此时如果 swapped out cache 已经刷盘成功，则直接切换，write thread写入新的cache；
    - 否则 write thread 等待一段时间并拒绝写入请求。
- **Journal**
  - `Journal 线程` 循环读取内存队列，写入磁盘：group commit，而非每个entry都进行一次write系统调用
  - 定期向 `Force Write Queue` 中添加强制写入请求、触发 fsync；
  - `Froce Write Thread` ：循环从 froce write queue 中拿取强制写入请求（其中包含entry callback）、在 journal 文件上执行 fsync；
  - `Journal Callback Thread` ：fsync 成功后，执行 callback，给客户端返回 reesponse



**常见瓶颈**

- **Journal write / fsync 慢**：则 `Journal Thread `、 `Force Write Thread` 不能很快读取队列。

- **DbLedgerStorage 刷盘慢**：write cache 不能及时清空并互切。

- **Journal 瓶颈**：Journal 内存队列入队变慢，导致 `Write Thread Pool` 任务队列满、请求被拒绝。

  



### 读取

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
> - 在数据流从上游生产者向下游消费者传输的过程中，上游生产速度大于下游消费速度，导致下游的 Buffer 溢出，这种现象就叫做 Backpressure 出现。
>
> - Backpressure 和 Buffer 是一对相生共存的概念，只有设置了 Buffer，才有 Backpressure 出现；只要设置了 Buffer，一定存在出现 Backpressure 的风险。



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



# | 功能

## || Geo Replication

https://pulsar.apache.org/docs/en/administration-geo

*Geo-replication* is the replication of persistently stored message data across multiple clusters of a Pulsar instance.

![image-20220331000551070](../img/pulsar/geo-replication-clusters.png)

- **配置**

  - 配置 clusters 互通性

    ```sh
    # Configure the connection from us-west to us-east.
    # Run the following command on us-west.
    $ bin/pulsar-admin clusters create \
      --broker-url pulsar://<DNS-OF-US-EAST>:<PORT> \
      --url http://<DNS-OF-US-EAST>:<PORT> \
      us-east
    ```

  - `--allowed-clusters`：配置 tenant，使其有权限使用上述clusters

    ```sh
    $ bin/pulsar-admin tenants create my-tenant \
      --admin-roles my-admin-role \
      --allowed-clusters us-west,us-east,us-cent
    ```

  - `set-clusters`：在 namespace level 指定 clusters 

    ```sh
    # 该namespace下的消息会被复制到所有指定cluster
    $ bin/pulsar-admin namespaces set-clusters my-tenant/my-namespace \
      --clusters us-west,us-east,us-cent
      
    # 不过发送消息时可以指定只复制到部分cluster:
    producer.newMessage()
            .value("my-payload".getBytes())
            .setReplicationClusters(Arrays.asList("us-west", "us-east"))
            .send();
    ```

    或者在 topic level 指定 `set-replication-clusters`：

    ```sh
    $ bin/pulsar-admin topics 
      set-replication-clusters --clusters us-west,us-east,us-cent 
      my-tenant/my-namespace/my-topic
    ```

    

- **原理**

  - **异步复制**
    ![image-20220331162951691](../img/pulsar/geo-replication-underline.png)

  - **Global Config Store**

    - replication_clusters

    ![image-20220331163143997](../img/pulsar/geo-replication-globalconfigstore.png)

  - **Geo-replication without global zk**

    - 可以实现自定义的复制策略，例如 Aggregation、Failover冷备
    - TODO







**Subscription Replication**

- 要解决的问题：

  - 复制后 LedgerId / EntryId 可能会变

    > Message ID = Ledger ID | Entry ID | Partition Index | Batch Index

  - 要复制ack状态：目前只复制 mark delete position（连续ack的最大id）

    > ACK：消费进度会被持久化到 ledger。
    >
    > ![image-20220330093458342](../img/pulsar/msg-ack-cursers.png)

- 实现

  - **Cursor Snapshot** 定期同步，记录message id 对应关系

    > ClusterA 向B/C发送 `ReplicatedSubscriptionSnapshotRequest`后，会收到响应，包含 ledger_id / entry_id；则ClusterA 会保存本地ledger_id / entry_id，以及其他cluster对应的ledger_id / entry_id
    >
    > ![image-20220330094251420](../img/pulsar/subs-replicate-cursor-snapshot.png)



- Cursor Snapshot 如何存储

  - 和正常的消息穿插存储： `Snapshot Marker`
  - 与正常的消息一样进行跨地域复制：副作用 - 影响backlog计算 

- 配置

  - Broker 启用：enableReplicatedSubscriptions=true (默认true)

  - 创建subscription时启用：

    ```java
    Consumer<String> consumer = pulsarClient.newConsumer(Schema.STRING)
                  .topic(topic)
                  .subscriptionName("my-subscription")
                  .replicateSubscriptionState(true)
                  .subscribe();
    ```

  - 可配置参数：多久做一次snapshot、snapshot复制请求的timeout时间、最多缓存多少个snapshot、





## || Tiered Storage

> - PIP: https://github.com/apache/pulsar/wiki/PIP-17:-Tiered-storage-for-Pulsar-topics 
> - Doc: https://pulsar.apache.org/docs/en/tiered-storage-overview/



- 存储老 segment:
  - Once a segment has been sealed it is immutable. I.e. The set of entries, the content of those entries and the order of the entries can never change. 即可不必存储在SSD上了、可存储到廉价介质。

- 读取老 segment
  - For Pulsar to read back and serve messages from the object store, we will provide a `ReadHandle` implementation which reads data from the object store. 
  - With this, Pulsar only needs to know whether it is reading from bookkeeper or the object store when it is constructing the `ReadHandle`.





## || 新功能

### Producer Partial RoundRobin

目的：分区过多时，producer的链接可能非常多。

![image-20220330202155910](/Users/alpha/dev/git/alpha/alpha-notes/img/pulsar/producer-partial-round-robin.png)

```java
PartitionedProducerImpl<byte[] producer = (PartitionedProducerImpl<byte[]>) pulsarClient.newProducer()
  .topic("")
  .enableLazyStartPartitionedProducers(true) //设置1
  .messageRouter(new PartialRoundRobinMessageRouterImpl(3)) //设置2
  .messageRouting(MessageRoutingMode.CustomPartition)
  .enableBatching(false)
  .create();
```



### Consumer Redeliver Backoff

```java
// 反向签收
client.newConsumer()
  .negativeAckRedeliveryBackoff(MultiplierRedeliveryBackoff.builder()
                                .minDelayMs(1000)
                                .maxDelayMs(60 * 1000)
                                .build())
  .subscribe();

client.newConsumer()
  .ackTimeout(10, TimeUnit.SECOND)
  .ackTimeoutRedeliveryBackoff(MultiplierRedeliveryBackoff.builder()
                                .minDelayMs(1000)
                                .maxDelayMs(60 * 1000)
                                .build())
  .subscribe();
```







# | 运维

## || 安装

https://pulsar.apache.org/docs/zh-CN/standalone/ 


