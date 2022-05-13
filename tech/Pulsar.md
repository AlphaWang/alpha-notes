[toc]

# | 基础

## || 特点

**Pulsar 要解决的问题**

- 数据规模
  - 多租户
  - 百万 Topics
  - 低延
- 存算分离
  - 解决运维痛点：替换机器、服务扩容、**数据 Rebalance**
- 减少文件系统依赖
  - **性能难保障**：持久化 fsync、一致性 ack = all、多主题
  - **IO 不隔离**：消费者读 backlog 会影响其他生产者和消费者



**特点**

- **云原生**
  - Broker 无状态
  - Bookie 可以水平扩展，新数据存入新的Bookie
- **支持多租户和海量 Topic**
  - Tenant + Namespace 逻辑隔离；便于做共享大集群
  - 最佳实践：
    - Access Control
    - Quota: 
    - Isolation: 延时、吞吐量、存储、placement policy；不同 tenant 可以是不同的 Cluster。
    - Tenant：可能对应一个 Team
    - Namespace：可能对应一个 application。用于通用的主题设置，例如 backlog quota、offload、retention policy； 
- **平衡消息可靠性与性能**
  - 得益于 Quorum 机制、条带化写入策略
- **低延迟**
  - kafka topic 增加，延迟也增加 
  - 单个发送线程 TPS 2w？ -- 拉卡拉数据：batchingEnabled, blockIfQueueFull
  - 
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



**功能：Event Streaming**

- **Connect**: pub/sub, connectors, protocol handlers. 
- **Store**: BK, Tiered Storage (HDFS, JCloud --> aws, gcs)
- **Process**: Pulsar Functionss (etl, routing), Flink, Spark, Presto



**社区发展**

- 常用功能
  - Pub/Sub
  - Multi-Tenancy
  - Functions
  - Tiered Storage
  - Connectors



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



**ZooKeeper**

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



**流程**

![image-20220513165819297](../img/pulsar/pulsar-arch.png)

![image-20220416215533390](../img/pulsar/pulsar-rw-flow.png)

（来自拉卡拉分享，Broker cache 的写入时机貌似不对）



**隔离性**

- BK 读写存储隔离

- Namespace 隔离

  - 让指定Namespace下所有 Bundle 只落在指定的 Broker 上，避免影响其他节点。

  - ```sh
    # 设置 NS 隔离策略
    pulsar-admin ns-isolation-policy set
      --primary broker-regs1
      --secondary broker-regs2
    ```

- Bookie 隔离：`set-bookie-affinity-group` 设置亲和性

  - 让指定Namespace下的主题尽量存到指定的 Bookie

  - ```sh
    # 设置bookie group信息
    pulsar-admin bookies set-bookie-rack
      -b 192.168.1.222:3181 
      -r rack1
      -g group1
    
    # 设置亲和性
    pulsar-admin namespaces set-bookie-affinity-group
      --primary-group group1
      --secondary-group group2
    # 设置反亲和性
    pulsar-admin namespaces set-anti-affinity-group
      --group group1
      tenant/namespace
    ```



## || 对比 Kafka

- Kafka 生产数据到 Leader，再同步到 follower；`ack=all`的情况下需要等待所有 ISR follower返回，性能差。Leader 宕机时如果ISR = 1，则可能导致数据丢失。
- Kafka 分区数据存储到指定的几个 Broker，利用率可能倾斜。
- Kafka 分区扩容困难，涉及到数据重平衡。
- Kafka 新增/替换 Follower 需要先fetch lag，同步整个分区数据，新的节点才能serve read/write；Fetch lag 会占用 IO，影响写入性能。
- Kafka catch-up read会**污染page cache**，影响 tailing read的性能、以及写入性能；
- Kafka 主题多了之后，也会污染 page  cache



## || 高性能

- Tailing Read：读取 Broker Cache
- Catch-up Read：读取 BK write cache，没有则读取 read cache，最后才会读取 Entry Log
- Produce：写入 Journal 盘，推荐 SSD



**Vs. RocketMQ**

![image-20220416135310696](../img/pulsar/rocketmq-write-flow.png)

- 升级同步双写流程
  - SendMessageProcessorThread 生成 CompletableFuture；随后即能继续处理下一个新请求；
  - CompletableFuture 何时完成：slave 复制位点超过消息位点后完成。完成后才响应客户端。



## || 高可用

> - https://jack-vanlightly.com/blog/2018/10/21/how-to-not-lose-messages-on-an-apache-pulsar-cluster



**读写高可用**

- **读高可用：Speculative Reads**

  - 原因：对等副本都可以提供读
  - 通过Speculative 减少长尾时延：同时发出两个读，哪个先返回用哪个
    Q：放大了读取请求数？

  > Kafka 为什么不能 Speculative Reads？
  >
  > - Kafka offset 是基于日志顺序读取、必须从 Leader读，而 BK 底层则并非连续存储，而是基于index

- **写高可用：Ensemble Change**

  - 最大化数据放置可能性



**Bookie 高可用**

某个 Bookie 宕机后如何处理。--> **Bookie Auto Recovery - Fragment Rollover**

> - https://bookkeeper.apache.org/docs/admin/autorecovery
>
> - https://www.youtube.com/watch?v=w14OoOUkyvo
>
> 注意：区别于broker宕机后的 **Ledger Recovery** --> Fencing 

- Auditor

  - 审计集群里是否有 bookie宕机；(ping bookies)
  - 审计某个Ledger是否有entry丢失；
  - Q：需要选主确定 auditor？--> YES. 

- 流程

  - 如果 bookie1宕机， Auditor 通过 ZK 感知到，扫描zk Ledger list 找出该bookie1存储的所有 ledger；
  - Auditor 在 `/underreplicated/` znode 下发布 rereplication 任务，每个任务对应一个 Ledger、等待一个 worker 认领。
  - Replication worker 监听该zk节点，如有新任务则加锁、从原ensemble复制原 ledger entries 到新 bookie2；
  - Replication worker 复制结束后，更新 Ledger metadata、修改原始的 Ensemble、剔除 bookie1 替换为 bookie2。

- 配置

  ```properties
  #bookkeeper.conf - 可并行复制的entry个数
  rereplicationEntryBatchSize=100
  ```

  

- Auto recovery 过程中并不会影响读取，因为 ensemble 中的其他 bookie 可以用于读取。

  > Q：会影响写入吗？--> 也不会，写入是写新的 Segment、往新的 Ensemble 中。

- 还可以手工 Recover

  ```sh
  #查看 Ledger metadata，BK fail之后，会产生新的 ensemble，但老的ensemble里还包含 FAILED_BK
  bookkeeper shell ledgermetadata -l LEDGER_ID 
  bookkeeper shell recover FAILED_BOOKIE_ID 
  #再次查看 ledger metadata，会发现老的ensembles bk列表剔除掉了FAILED_BOOKIE_ID，数据拷贝到了新BK.
  bookkeeper shell ledgermetadata -l LEDGER_ID 
  ```

  



**Broker 高可用**

- 当 Broker 节点宕机 / 或者与zk断联自动重启，客户端可以通过 Lookup 重新触发 Bundle 与 Broker 之间的绑定；让主题转移到新的 Broker 上。

  > - https://pulsar.apache.org/docs/administration-load-balance/ 
  >
  > - Q: 花费多少时间？

- 同时与该 Broker 关联的 Ledger 会进入恢复流程，**Fencing** 并重新找 owner Broker。见：**Ledger Recovery**。



**跨机架高可用**

- BK 客户端的跨区域感知：
  - 写入时选择bookie节点时，必定包含来自不同机架的节点。
- 注意
  - 务必保证每个机架都有足够多节点，否则可能导致找不到足够多不同机架节点。
  - 同步高可用：同步写入多机架，延迟会增加。



**跨地域高可用**

- GEO Replication：异步，非强一致







# | 客户端

## || Producer

https://pulsar.apache.org/docs/en/concepts-messaging/

**消息生产流程**

- Producer：Message Routing 选择分区
- Topic Discovery (lookup)：跟 Broker 通讯，找到分区对应的Broker
- Broker：调用bk客户端，并发写入多个副本



**配置**

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



**实现类**

- **ProducerImpl**
  - HandlerState 状态机
  - 发送流程
    - 同步发送底层也是调用异步发送，然后Future.get()
    - 压缩
    - 分块
    - 校验并设置元数据，包括 sequenceId
    - 进入批量发送队列
    - Flush：直接触发、或等待
    
    > 如果是异步发送，收到ack后跟未决队列进行对比，如果收到 msg-2 ack，但尚未收到 msg-1 ack；则重发 msg1
- **PartitionedProducerImpl**
  
  - 内有保存 ProducerImpl List，每个partition对应一个



**功能**

- **Batching**

  - 最大消息个数

  - 最大发送延迟

  - batch是个整体

    - batches are tracked and *stored* as single units rather than as individual messages.
    - Consumer unbundles a batch into individual messages. In general, a batch is acknowledged when all of its messages are acknowledged by a consumer. 
    - 消费的时候，一批消息只会被同一个consumer消费。
  - 一个消息ack失败，会导致整个batch重发。2.6.0 之后引入 batch index acknowledgement 解决重复发送问题：消费者发送 batch index ack request. 
    
    

- **Chunking**

  > PIP 37: Large message size handling in Pulsar: Chunking Vs Txn https://github.com/apache/pulsar/wiki/PIP-37%3A-Large-message-size-handling-in-Pulsar

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

  > https://github.com/apache/pulsar/wiki/PIP-6:-Guaranteed-Message-Deduplication

  - 效果：生产者多次发送同样的消息，只会被保存一次到bookie。

  - 实现：

    - 生产者每条消息会设置一个元数据 `sequenceId`，topic owner broker遇到比之前小的ID则可过滤掉。
    - 生产者重连后，会从Broker拿到当前topic最后的`sequenceId`，继续累加。
    - Broker还会定期将sequenceId快照存储到 BK，防止Broker和客户端同时宕机。

  - 配置：`brokerDeduplicationEnabled=true`

    > https://pulsar.apache.org/docs/en/cookbooks-deduplication/ 
    
  - 可用于 effectively-once  语义

    > https://www.splunk.com/en_us/blog/it/exactly-once-is-not-exactly-the-same.html 
    
  - **消息重复的场景**

    - Broker Down

      > 1. 生产 N 条消息，并成功写入 BK Ensemble；但在 ack 之前 Broker 宕机。
      > 2. 新 Broker 触发 Ledger Recovery，恢复之前的消息；
      > 3. 而客户端重连到新 Broker，又重新发送这 N 条消息；




- **消息顺序**

  - `业务线程` 的影响：多个线程同时持有一个 Producer 对象；
  - `路由模式` 的影响：SinglePartition 有序、RoundRobinPartition 无序；
  - `分区` 的影响：如果只有一个分区，则同SinglePartition模式，能保证有序；例外：异步发送失败后重试；
  - `发送方式` 的影响：同步 vs 异步
  - `批量发送` 的影响: 一批消息是原子性。但批与批之间的顺序可能乱；
  - `消息key` 的影响：相同key会被发到同一分区，能保证有序性；
- ack 的影响：
    - 单条确认时，未确认的消息在超时后会重新投递；若要保序，则必须按顺序 ack；
    - 累积确认时，在超时后会重新投递一批消息；
  
  


- **Exclusive Producer**

  - 只有一个 producer 能写入成功。 类似订阅类型。 

  - PIP-68 https://github.com/apache/pulsar/wiki/PIP-68%3A-Exclusive-Producer

    ```java
    Producer<String> producer = client.newProducer(Schema.STRING)
          .topic("my-topic")
          .accessMode(ProducerAccessMode.Exclusive)
          .create();
    ```

- **Producer Partial RoundRobin**

  - 目的：分区过多时，producer的链接可能非常多。
  - 解决：
    - Producer 懒加载：只有在用的时候才会真正创建producer实例。
    - Partial RoundRobin：每个 producer 实例可能只 roundrobin 到一部分分区。

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

  









## || Consumer

**消息读取流程**

- 消费者订阅的时候，根据订阅策略决定消费哪些分区；经过 topic discovery 长连接到 Owner Brokers
- Tailing Read：
  - Broker 存储消息到 bk后，从内存读取，dispatch 到consumer 的 **Recive Queue**.
- Catch-up Read:
  - 到 BK 读取数据，可读取任意 bookie



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
  
  - 直接从 RecieverQueue 中 take()，同步等待
- 异步接收
  - 用户业务线程
  - Netty IO 线程
  
    > Q: 如果推送来的消息超过 RecieverQueue 怎么办？
    >
    > -- 推送也是消费者触发的？

![image-20220403134548890](../img/pulsar/consume-flow.png)



功能

- **Acknowledgement**

  - **确认场景**

    - 单条消息确认：`consumer.acknowledge(msg);`
    
  - 累积消息确认：`consumer.acknowledgeCumulative(msg);`
    
    - 批量消息中的单个消息确认：

      - Broker 配置 `acknowledgementAtBatchIndexLevelEnabled=true`
      - 原理是在客户端重新收到一批后，过滤掉已确认的消息。

    - 否定应答: 表示处理失败、稍后重发给其他消费者。`consumer.negativeAcknowledge(msg);`

      > Q: 何时重新deliver、能否指定? 
    > A: 全局设置延迟时间；如有大量消息延迟消费，可调用 `reconsumerLater` 接口。
    
    
    
  - **确认流程**

    - 待确认的消息先放入`AcknowledgementsGroupingTracker`缓存，默认每100ms、或大小超过1000则发送一批确认请求；目的是避免broker收到高并发的确认请求。
    - 对于 **ack at batch index level**，存储格式为Map<Batch MessageId, `BitSet`>；
    - 对于 **累积消息确认**，Tracker 只需保存最新确认位置即可。
    - 对于**否定应答**，由 `NegativeAcksTracker`处理，其复用 Pulsar Client 时间轮，定期发送给 Broker。

    

  - **Ack timeout**: 可对unack消息自动重发。

    - 情况一：业务端调用 receive() 后：消息进入 `UnAckedMessageTracker`，其维护一个时间轮，超时后发送 redeliverUnacknowledgedMessages 命令给broker。

    - 情况二：消费者做了预拉取，但尚未调用 receive()：Broker侧将这些消息标记为 `pendingAck` 状态，除非当前消费者被关闭，才会被重新投递。

      > - Broker RedeliveryTracker 会记录每个消息的投递次数；
      > - 可知如果消费者 ReceiverQueue 设置过大 是会对Broker有影响的。

  - **Consumer Redeliver Backoff**

    - 默认否定应答、或应答超时后，会马上重发。
    - Backoff 则允许自定义重发的间隔。

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
                        .initialSubscriptionName("my-sub")
                        .build())
                  .subscribe();     
  // initialSubscriptionName 的作用:
    // 否则是懒创建，无法指定retention等参数。
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



**TableView**

- 2.10 新引入，类似 compacted topic
  `client.newTableViewBuilder()`
- 客户端内存实现，数据不能太大。





## || Subscription

 The subscriptions do not contain the data, only meta-data and a cursor.



**订阅类型**

![image-20220322112234641](../img/pulsar/subscription-modes.png)

- **Exclusive**

  - 只有一个消费者绑定到当前订阅。其他消费者需要使用不同的 subscription name，否则报错 ConsumerBusyException: Exclusive consumer is already connected。

  - 有序。

  - 支持单条消息确认、累积消息确认。

    > Q: 是partition 维度？- Y！
    >
    > ![image-20220322112520553](../img/pulsar/subscription-modes-exclusive.png)

- **Failover**

  - 多个消费者可以绑定到当前订阅（而不像 Exclusive 那样直接报错），但只有一个收到消息。

  - 有序。

    > 类似 Kafka。
    >
    > ![image-20220322112744805](../img/pulsar/subscription-modes-failover.png)

- **Shared** 

  - 多个消费者可以绑定到当前订阅，按 round-robin 模式接收消息（消费者可设置 priorityLevel 来提升自己的优先级）。

  - 限制：

    - 无序、
    
    - 无法使用 cumulative ack，类似kafka . --> 否则可能有误ack的情况
    
      用 individual ack时，如果中间有部分msg没有ack，则重启后会重新收到 
      
    - 不过可以 **[batch ack](https://pulsar.apache.org/api/client/org/apache/pulsar/client/api/ConsumerBuilder.html#acknowledgmentGroupTime-long-java.util.concurrent.TimeUnit-)**，提高吞吐量。
    
    > 类似传统消息队列模型。每个consumer 可能消费都 partition 中的**一部分**数据。
    > ![image-20220322113010001](../img/pulsar/subscription-modes-shared.png)
    
  
- **Key_Shared** 

  - 多个消费者可以绑定到当前订阅，按相同key模式接收消息。

  - In *Key_Shared* mode, multiple consumers can attach to the same subscription. Messages are delivered in a distribution across consumers and message with same key or same ordering key are delivered to only one consumer. 

  - No matter how many times the message is re-delivered, it is delivered to the same consumer. When a consumer connected or disconnected will cause served consumer change for some key of message.

  - 限制

    - 必须指定 key，或orderingKey
    - 消费者无法使用 cumulative ack
    - 生产者必须禁用 batching，或者使用 *key-based batching*

    > ![image-20220322113231815](../img/pulsar/subscription-modes-key-shared.png)
  



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

- 普通 Topic 只对应一个broker，限制了吞吐量；而 **Partitioned Topic 可被多个 broker 处理**、分担流量压力；
- 实现：N 个内部主题。
- routing mode: 决定生产到哪个分区；
  - RoundRobinPartition
  - SinglePartition：随机
  - CustomPartition
- subscription mode: 决定从哪个分区读取；



**Non-persistent Topic**

- 普通 Topic 存储消息到 bookie，而non-persistent topic则只存到内存
- 更快



**消息ID**

- `LedgerId, EntryId, BatchIndex, PartitionIndex`
- 类似 Kafka offset.





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

## || 生产消费流程

**Broker 端生产流程**

- **managedLedger**：与BookKeeper打交道。

![image-20220404110407223](../img/pulsar/broker-produce-flow.png)





**Broker 端消费流程**

- handlwFlow：收到消费者的请求
- 校验unacked消息：如果该消费者接收了很多消息但都没确认，则触发限流。

![image-20220404110452228](../img/pulsar/broker-consume-flow.png)



**Dispatcher**

Dispatcher 负责从 bk 读取数据、返回给消费者。

**流程**

- 收到消费者 flowPerm 命令后，循环调用 bk 客户端读取数据；

- 凑足后，选择一个 Consumer：

  - Key Shared的情况下

    - AUTO_SPLIT：新consumer按照一致性哈希环（TreeMap实现）方式进入。
    - STICKY：加入哈希环时，如果区间有重叠 则报错，拒绝新consumer加入。

  - 按优先级选择消费者。

  - 注意：如果是 batch msg，服务端并不知道batch里有多少条消息，可能超发。

    > 新引入 preciseDispatcherFlowControl，根据历史记录估算batch msg count，减少超发过多的情况。

- 过滤：延时消息、事务消息

- 发送给消费者：根据订阅类型不同

  - 独占：所有entry发送给一个消费者；
  - 共享：发给多个消费者；
  - KeyShared：按key选择；



## || 元数据管理

**元数据存储**

- LocalZooKeeper
  - 保存集群内部的元数据
- GlobalZooKeeper (ConfigurationStoreServers)
  - 用于不同Broker集群之间的数据互通；
  - 例如跨地域复制时集群之间的元数据需要互相感知。



**元数据缓存**

- `AbstractMetadataStore` 使用了 Caffeine 缓存
- 通过 ZK Watcher 通知缓存更新
- 缓存数据
  - existsCache: 缓存节点、路径是否存在
  - childrenCache: 缓存子节点信息
  - metadataCache: 缓存当前节点信息



**线程安全**

- 使用ZK Version 机制，通过 **CAS + CopyOnWrite** 来保证线程安全。

- 更新原数据流程，被封装到一个Function类型的 Lambda 表达式：

  - 1. 从缓存中读取 Policies 对象；
    2. 修改对象中的某个策略值；
    3. 写入 ZK，并带上 Version；

  - 如果第三步失败，则重新调用 Function，从第一步重新执行。



## || 存储管理

> `ManagedLedger` 负责消息存储，而不是直接使用 bk client。

- 每个 Topic 都有一个 managedLedger，包装 Ledger + Cursor
- managedLedger 还管理一个 cache。--> Broker 层的cache ! 

![image-20220424231100770](../img/pulsar/pulsar-managedLedger.png)





**存储模型**

- 每个非分区 Topic 都对应一个或多个 Ledger；只有一个Ledger处于OPEN状态。
- 每个 Ledger 有一个或多个 Fragment；
- 每个 Fragment 包含多个 Entry，每个 Entry 对应一条或一批消息。
  ![image-20220424231714789](../img/pulsar/pulsar-topic-ledger.png)



**存储流程**

- **创建 Ledger**
  - 创建 Topic 时，仅向 ZK 写入 Ledger 元数据；当Producer或Consumer连接到broker上的某个主题时，才会真正创建对应 Topic 的 Ledger。
  - 创建 `ManagedLedger`，获取元数据、决定是否创建新 Ledger。
- **写入 Ledger**
  - `ManagedLedger` 封装 OpAddEntry 对象。
  - 根据 EntryId 计算需要并行写入的 Bookie 节点、然后并行写入。 
  - 写入成功，则缓存到 Write Cache。
- **读取 Ledger**
  - Cursor逻辑：对比当前读的位置、以及Ledger中最后一条写入消息的 MessageId，判断是否还有消息可读。
  - 检查是否需要切换 Ledger。
  - 尝试从 `ManagedLedger` Write Cache 中读取，读不到则回源到 BK 客户端读取。



**游标**

- 作用：存储当前订阅的消费位置。存到一个 log 里，定期compact。

- 存储内容

  - `markDeletePosition`：被连续确认的最大 EntryID，这个ID之前的所有entry都被消费过。

    > 消费者重启后，只能从markDeletePosition位置开始消费，会存在重复消费的可能。

  - `readPosition`：订阅的当前读取位置。

  - `individualDeletedMessages`：用于保存消息的空洞信息。

    > 默认利用 Guava Range 对象，存储已被确认的范围列表。
    >
    > 还可选择优化版，利用 BitSet 记录每个Entry是否被确认。适用于空洞较多的情况。

  - `batchDeletedIndexes`：用于保存批量消息中单条消息确认的信息。

    > Key = Position对象，包含 LedgerId, EntryId
    >
    > Value = BitSet，记录batch中有哪些消息已被确认。

- 空洞管理的优化
  - 问题：大量订阅会让游标数量暴增、BK 单个entry最大5M 超过会导致空洞信息持久化失败。
  - 优化：LRU + 分段存储。
    //TODO
- 消息回溯 seek()
  - 根据 ID 回溯
    - 修改 Cursor 中的 markDeletetionPosition 和 readPosition、清理 individualDeletedMessages 中的空洞信息、清除 batchDeletedIndexes 中的信息。
    - 后续消费者重新订阅时，会从readPosition开始读取消息。
  - 根据时间戳回溯
    - 遍历找到 Ledger，再二分查找到对应的 EntryId



**数据清理**

- 属性值

  - `markDeletePosition`：如果一个 Ledger 中所有 entry 都在 `markDeletePosition` 之前，则这个 Ledger 可被清理。

    > 即，entry 被确认后会立即标记为可删除，但并不一定会马上被删除。需要等到Ledger中所有entry 都被确认才行。

  - `Retention`：消息**被 ACK 后**还想保留一段时间/或大小。

  - `MessageTTL`：当堆积超过此阈值，即便消息没有被消费，这个 Ledger 也会自动被确认、让Ledger 进入 Retention 状态。

    > 本质是自动将  `MarkDelete` 向前移；解决如果没有订阅时，消息的永远堆积问题。 

- **Data Retention**

  - 只要有 Cursor 存在，则其之后的数据不会被删除；除非ack后达到 `Retention` 
  - **ACK过的数据分为两部分**：超过 Retention 的可以被删除，Retention 之内的不可被删除。

  ![image-20220426224056965](../img/pulsar/pulsar-retention.png)

- **TTL**

  - 目的：如果只有 Retention，Consumer不再消费后，数据岂不一直不会被清理？
  - TTL 到期后，相当于自动 ack

    > Q: TTL 是 subscription level 的配置？！
  - Kafka 相当于 `TTL = Retention`

  ![image-20220426224548840](../img/pulsar/pulsar-ttl.png)

- 相关监控

  - `Msg Backlog`：尚未被 ack 的消息数量；

  - `Storage Size`：所有未被删除的 Segment 的空间大小；

    > 消息删除是基于 Segment 分片的，活跃 Segment 不会被删除，即便其中包含超过retention的entry；

    
  
- **数据清理的单位是 Ledger！**

  - Ledger (Segment) 标记成可以删除后，是被一个定时后台线程清理；所以有延时。
  - Ledger 标记成可删除后，并不表明对应 Entry Log 可以被删除，因为 Entry Log 可能还包含其他 Ledger 数据。
  - Retention 到期后也不一定马上删除，还需要看 TTL + 有无 ack.







## || 主题归属



- **ZK 存储**

  > 不能直接在 zk 上存储 topic - broker 归属关系：否则数据量太大

  - ZK 只保存 Bundle 与 Broker 之间的联系。
  - Topic 归属哪个 broker 是通过一致性哈希动态计算出来的。

  

- **Topic 归属的计算步骤** `ServerCnx#handleLookup`

  - 根据 **namespace** 找到其所有的 Bundle；

  - 计算 Topic 所属的 Bundle：一致性哈希算法；每个 **namespace** 有一个哈希环。

  - 确定 Bundle 归属哪个 Broker，先找到**裁判Broker**，其通过 `loadManager` 查找负载最低的 Broker 并把 Bundle 分配给他

    > 如何选择裁判 Broker: 选主
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
  - 如何判定负载高？
    - For example, the default threshold is 85% and if a broker is over quota at 95% CPU usage, then the broker unloads the percent difference plus a 5% margin: `(95% - 85%) + 5% = 15%`.



- **客户端如何找到 Topic Owner**

  - Topic owner == Namespace bundle owner，相关信息记在 zk 中。
  - 客户端执行 Topic lookup 发送到任意 Broker；Broker 找到 topic 归属哪个 namespace bundle、在从zk 找到bundle对应的 Owner Broker。

  > Q：如果某个 topic 消费者非常多（fan-out），那么 Owner Broker 压力会非常大。
  >
  > - 改进：增加 **readonly broker** 的概念，同步 cursor.
  > - https://github.com/apache/pulsar/wiki/PIP-63%3A-Readonly-Topic-Ownership-Support



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



## || 事务消息

目的：用于保证精确一次语义。

- Producer 生产到不同分区时，要么同时失败，要么同时成功。
- Consumer 消费多条消息时，要么同时确认失败，要么同时确认成功。
- Producer / Consumer 在同一个事务时，要么同时失败，要么同时成功。



用法：

```java
// 示例：保证 produce & consume ack 在同一个事务里
PulsarClient client = PulsarClient.builder()
  .serviceUrl()
  .enableTransaction(true)
  .build();

// 1. 开启事务
// 客户端向`TC`发送newTxn命令，TC生成新事务
Transaction tx = pulsarClient
  .newTransaction()
  .withTransacationTimeout(1, TimeUnit.MINUTES)
  .build()
  .get();


Message<String> msg = sourceConsumer.receive();
// 2. 发送消息
// 消息会先经过每个Broker上的`RM`，RM记录元数据；写入主题后，该消息对消费者不可见
// 原理：每个主题有一个 maxReadPosition属性
sinkProducer.newMessage(tx)
  .value("sink data")
  .sendAsync();
// 3. 发送ACK
// 不会直接修改游标中的MarkDeleted位置，而是先持久化到一个额外的日志Ledger中，此时主题中的消息并未被真正确认。
sourceConsumer.acknowledgeAsync(msg.getMessageId(), tx);
// 4. 提交事务
// TC收到请求后，向RM广播提交事务；更新元数据、让消息对消费者可见
// 消费者RM会从日志Ledger读取刚才的消息确认、执行确认操作。
tx.commit().get();

```





## || Schema

**目的**

- 否则需要代码 encode / decode byte[]，如果数据类型改变不好维护。
- schema 定义如何序列化反序列化、定义数据格式、处理兼容性。
- 与 Pulsar SQL 集成后可以映射为字段。



**Schema 定义格式**

- Type: AVRO / JSON / PROTOBUF

```json
{
  "type": "JSON",
  "properties": {
    "__alwaysAllowNull": "true",
    "__jsr310ConversionEnabled": "false"
  },
  "schema" : {
    type: record,
    name: Person,
    namespace: xx,
    fields: [
      "name": "age",
      "type": ["null", "string"]
    ]
  }
}
```



**Schema 的使用**

```java
//Static struct schema
Producer<User> producer = client.newProducer(Schema.AVRO(User.class)).create();
produer.newMessage()
  .value(user)
  .send();
Consumer<User> consumer = client.newConsumer(Schema.AVRO(User.class)).create();
consumer.receive();

//Dynamic struct schema: 常用于connector
RecordSchemaBuilder builder = SchemaBuilder.record("schemaName");
builder
  .field("intField")
  .type(SchemaType.INT32);
SchemaInfo schemaInfo = builder.build(SchemaType.AVRO);

Producer<GenericRecord> producer = client.newProducer(Schema.generic(schemaInfo)).create();
producer.newMessage()
  .value(schema.newRecordBuilder().set("intField", 32).build())
  .send();
```



 **Auto Schema**

- AUTO_PRODUCE：如果生产者发送的bytes不符合topic schema，则拒绝
- AUTO_CONSUME：反序列化为 GenericRecord，适用于提前不知道schema的情况。



**Schema 原理**

```java
Producer<User> producer = client.newProducer(Schema.AVRO(User.class)).create();
```

- 通过 `Schema.AVRO(User.class)` 创建 SchemaInfo
- `newProducer` 时连接到 broker，并发送 SchemaInfo；
- Broker 收到 SchemaInfo 后：
  - 如果主题没有 schema，则创建；
  - 如果主题已有 schema，且传入的schemaInfo是新的，且与已有的兼容，则创建新version.
  - 如果不兼容，则失败。
- 兼容性检查
  ![image-20220430205726651](../img/pulsar/pulsar-schema-compatibility.png)



**Schema 存储**

- Schema 存储在 BookKeeper 中，而不是 zk。





## || 安全机制

> https://pulsar.apache.org/docs/en/security-overview/

**认证授权**

- 支持的 auth provider
  - JWT
  - Athenz
  - Kerberos
  - OAuth2.0
- 配置
  - 客户端：`authPlugin` `authParams`
  - Broker：`authenticationProvider` `authorizationProvider` `superUserRoles` 
  - Broker 间通讯配置：`brokerClientAuthenticationPlugin` `brokerClientAuthenticationParameters = superUser token`
  - Broker-BK间通讯配置：`bookkeeperClientAuth...`
  - Proxy 配置：
    - 相当于Broker + Broker间通讯，配置项的并集
    - `forwardAuthorizationCredential = true/false`：是否转发客户端认证信息。
  - Function Worker：
    - 相当于 Broker 的配置：以便客户端/Proxy访问
    - 也要配置`clientAuthenticationPlugin` `clientAuthenticationParameters`：以便访问 Broker
  - Pulsar Function：
    - 继承 Function Worker 权限；可能有风险
    - 实现 FunctionAuthProvider，自定义



**加密**

- TLS加密
- 端到端加密





# | BookKeeper

> https://www.bilibili.com/video/BV1T741147B6?p=5 



## || 架构

### **概念**

> https://bookkeeper.apache.org/docs/latest/getting-started/concepts/ 

- **Entry**：一条日志记录。each unit of a log is an *entry*; 
  - 包含内容包含 metadata 和 data
    - Ledger Id
    - Entry Id
    - LAC, Last Add Confirmed：最后一条已确认的 entry id
    - Digest：CRC32
  - Data byte[]
  - Authentication code
- **Ledger** (Segment)：一组日志记录，类比一个文件。streams of log entries are called *ledgers*
  - 打开/关闭 Leger 只是操作`元数据`(元数据存储在zk)：
    - State: open/closed
    - Last Entry Id: -1L
    - Ensemble、WriteQuorum、ReadQuorum Size
  - Ledger 有多个 **Fragment** 组成。每当 bookie failover 时就生成一个新 Fragment。
- **Bookie**：存储 ledger的服务器。individual servers storing ledgers of entries are called *bookies*
  - 每个 bookie 存储部分 ledger *fragment*, 而非完整ledger



### 组件

**1. Metadata Store**

- Zk / etcd
- 存储 ledger 元数据、bookie 信息。
- 服务发现

> 可用性：如果zk挂了，已打开的Leger可以继续写，但create/delete ledger 会报错



**2. Bookie** 

- 而 Bookie 逻辑很轻量化
- 可当做是一个 KV 存储
  - `(Lid, Eid) --> Entry `
- 操作：Add / Read

![image-20220326130700495](../img/pulsar/bk-arch-rw-isolation.png)

![image-20220416233528333](../img/pulsar/bk-arch-rw-isolation2.png)

> Q: 先写Write Cache，再 flush 到 Journal，会不会导致读到脏数据？
>
> A: 不会，LAC 之后的都读不到





**3. Client**

> 对 Pulsar 来说，此 client 为 Broker.

- 胖客户端：外部共识；

  - 例如 EntryID 是由客户端生成 ，前提：一个 Ledger 只有一个 Writer。

- 功能

  - 指定 WQ、AQ

  - 保存 **LAP**：Last Add Push，发出的请求最大值，LAC~LAP 之间的数据正在存储中；

  - 保存 **LAC**：Last Add Confirm，收到的应答最大值，LAC 之前的数据一定已被持久化；

    - 客户端需要保证不跳跃。

      > 例如收到 3 的ack、但未收到 2 的ack：收到的LAC = 1, 3时，不能直接改成3，因为2还没收到；
      >
      > 同时有超时机制，如果长时间不能收到2的确认，则触发 ensemble change，将2/3都再次写入新bk。

    - 因为不跳跃，所以能保证 LAC 之前的 entry 一定已被存储成功。

    - LAC 保存为 entry 的元数据，而不必存到zk。并定期同步到 bookie！？

      > 示例：LAC ~ LAP 之前的 entry 是正在存储中的数据。
      > ![image-20220326130309827](../img/pulsar/bk-arch-consistency.png)

  - Ensemble Change，当某个bookie宕机（Q：写入失败即认为宕机？）：

    - 新的entry可能存到新的 bookie。
    - 对于已宕机bookie里存储的数据，如何修复：
    
      > Auto Recovery https://bookkeeper.apache.org/docs/admin/autorecovery 



> Q: 读取时如何找到entry存在哪个bookie?
>
> - 任何一台 bookie 都可以响应？
> - --> 根据 Ledger ID 从元数据里找到 ensemble list，再计算出存储的 bookies. 







### WQ/AQ

**节点对等架构**

![image-20220326124934997](../img/pulsar/bk-arch-openLedger.png)

- 胖客户端决定：openLedger(`Ensemble`,  `Write Quorum`,  `Ack Quorum`)

  - **Ensemble**：组内节点数目，用于分散写入数据到多个节点；**控制一个 Ledger 的读写带宽**

    > 注意必须 < 所有节点总数，否则一个节点宕机就会导致写入失败。
  - **Write Quorum**：数据备份数目；**控制一条记录的副本数量**；
  - **Ack Quorum**：等待刷盘节点数目；**控制写入每条记录需要等待的 ACK 数量**；

- 灵活性配置

  - `增加 Emsemble (E > WQ)`：“条带化” 读写bk
    - **增加读写带宽**，增加总吞吐量，充分发挥每块磁盘IO性能。
    
      > Q: 但是会影响读取性能？https://medium.com/splunk-maas/apache-bookkeeper-insights-part-1-external-consensus-and-dynamic-membership-c259f388da21 
    - 还可以让数据分布更平均，避免某个分区数据倾斜。
  - `WQ = AQ`，等待所有Write Quorum的ack：**提供强一致性保障**
  - `减少 AQ`：**减少长尾时延**

  

- 示例
  
  - E = 5, WQ = 3 条带化写入
  - 对于指定 entryId，对5取模，决定存到哪三个 bookie；效果是按 Round Robin选择![image-20220326125340028](../img/pulsar/bk-arch-openLedger-eg.png)



### 外部共识 

> - External Consensus: https://medium.com/splunk-maas/apache-bookkeeper-insights-part-1-external-consensus-and-dynamic-membership-c259f388da21 
> - BK LAC & 可视化 & compare with Raft: https://www.youtube.com/watch?v=7etLdsC-qbM
> - https://www.slideshare.net/hustlmsp/apache-bookkeeper-a-high-performance-and-low-latency-storage-service
> - Scaling Out Total Order Atomic Broadcast with Apache BookKeeper https://www.splunk.com/en_us/blog/it/scaling-out-total-order-atomic-broadcast-with-apache-bookkeeper.html



外部共识：

- 数据复制是由 bk 客户端（Pulsar Broker）执行的。

- 因为复制和协调逻辑由客户端控制，所以客户端可以在故障发生时自由更改 Ledger 成员！

- 条件：一个 ledger 任何时候都不会有两个 bk 客户端写入、LAP / LAC 维护在 bk客户端。

  > LAC 包含在发送到 bk 节点的每个 entry中；存储节点并不使用该 LAC、用于客户端来查询 LAC。
  >
  > 客户端 Broker 知道最新的 LAC，而存储节点里的 LAC 可能是旧版。

**实现：**

- **一个ledger只会由一个broker负责写入**、写满即关闭；

  - 当broker宕机，新的broker会写入新的ledger，而不会操作老ledger；
  - 即：一个ledger任何时候都不会有两个broker写入。

- 同时宕机后的老 Broker 会被禁止写入老 Ledger：**Fencing**

  - 场景：broker1 与zk出现了网络分区，zk把broker2设为新的topic owner，新broker发现ledger处于open状态，则在接管前需要进行 recovery操作。

  - **流程（详见 恢复Ledger）：**

    - 1. **Recover Ledger-X**
         并不是直接恢复 LedgerX并继续写入，而是恢复 LAC 之后的数据（尚未被确认的数据），然后直接关闭 LedgerX。

      - 1.1 Fence Ledger-X ==> Retrun LAC

        > 新broker2对**原来的 ensembler** 发起 fencing 操作：将bookie状态设为fenced，则往原ledger的后续的写入会失败，发生 LedgerFencedException；-- 原来的broker1 会放弃ownership。

      - 1.2 Forward reading from LAC until no entry is found.

        > 恢复 LAC之后的数据：尝试找 LAC + 1 的entry，如果其已经写入部分 WQ 但尚未达到 AQ，则进行复制以便达到AQ，修复 LAC + 1。

      - 1.3 Update the ledger metadata.

        > 关闭原 Ledger-X

    - 2. **Open a new ledger** to append. 

> 对比 Kafka ISR：类似主从
>
> - Under min ISR 会导致写入失败，客户端需要等待broker达成一致（主从复制）。还要考虑如果 unclean leader election，会有truncate，可能数据丢失。
> - 而 Pulsar 的recovery 特别容易。只需开启新的 Ledger segment 并将



**复制协议中日志的三种区域**

- **未提交区域**：尚未达到 AQ / Commit Quorum
- **已提交头部**：已达到 AQ / Commit Quorum，但尚未达到 WQ / Replication Factor。-- 此部分数据对客户可见。
- **已提交尾部**： 已达到 WQ / Replication Factor

>  <img src="../img/pulsar/consensus-kafka-3zones.png" alt="img" style="zoom:67%;" />
> *(Kafka 日志的三个区域)*
> <img src="../img/pulsar/consensus-bookkeeper-3zones.png" alt="img" style="zoom:67%;" />
> *(Bookkeeper 日志的三个区域)*



**Ensemble Change**

- 当客户端写入某个 BK 失败，会选择新的 BK 来替代；会创建新 Ledger (Segment) 并将“未提交区域” 保存到新 Segment、然后继续往后写入。--> **写入可用性很高**。

- 而“已提交头部”、已达到AQ但未达到WQ的 entry 会被保留在原始 Ledger Fragement 中；

  > 这会导致 BK Ledger 中部会包含“已提交头部 + 尾部”；而 Kafka 日志中部全部是“已完全提交数据” (已提交尾部)
  >
  > ![image-20220512100854284](../img/pulsar/consensus-bookkeeper-move-uncommttied-to-new-ledger.png)
  > *(BookKeeper Emsemble Change: moves uncommitted entries to the next fragment)*

- 同时宕机 BK 上的原有数据会被慢慢修复：

  - **TODO**

- Q：客户端访问的时候如何知道对应entry究竟存在哪些 bk？
  --> Ledger 元数据包含 Ensemble 信息，Ensemble change 发生会增加一条记录

  ```
  0: B1, B2, B3, B4, B5 // fragment-1
  7: B1, B2, B3, B6, B5 // fragment-2: entryId=7之后，写入到新 ensemble
  ```





**类比 Raft 一致性协议**

![image-20220326130523922](../img/pulsar/bk-arch-consistency-raft1.png)

日志复制过程：

- Q: Writer/Broker 相当于 Leader?

![image-20220326130610241](../img/pulsar/bk-arch-consistency-raft2.png)





## || 读写流程

> - Apache BookKeeper Internals — Part 1 — High Level 
>   https://medium.com/splunk-maas/apache-bookkeeper-internals-part-1-high-level-6dce62269125

读写概览：

![image-20211231232219900](../img/pulsar/bookkeeper-read-write-components.png)



### 写入

> - Apache BookKeeper Internals — Part 2 — Writes 
>   https://medium.com/splunk-maas/apache-bookkeeper-internals-part-2-writes-359ffc17c497

![image-20211231232945352](../img/pulsar/bookkeeper-write-overview.png)

**前提**

- 一个 Ledger 只能被一个 broker 写入，这保证了 broker 可以维护严格递增的 EntryId。



**三种文件**

> - **Journal**：相当于 WAL；建议用SSD
>- **Entry log**：同一个 Entry log 可能存储多个 Ledger 的 entry
>   - Q: 那么 Ledger 是一个逻辑概念？
>   - Q: 必须所有 Ledger 都删除才能真正删除 entry log？--> bookie 异步 compaction：移动ledger到其他entry log
> - **Index file**: rocksDB



**两个存储模块：**

> - **Journal**
>   - 数据写入 Journal 后，触发 fsync，并返回客户端
> - **Ledger**
>   - 以异步方式批量刷盘



- **Journal**

  - 作用：事务日志文件。**在修改 ledger 之前，先记录事务日志**。

    - 所有写操作，先**顺序写入**追加到 Journal，**不管来自哪个 Ledger**。
    - 写满后，打开一个新的 Journal、继续追加。 默认保存5个备份 file，老的被清理。

  - 特点：

    - 写入速度快（顺序写入一个文件，没有随机访问）、**读写存储隔离**
    - 相当于是个**循环 Buffer**. 
    - 但查询困难。--> 引入索引，但并不直接在 Journal 上做索引：读写分离。

  - 配置：

    - `JournalDirectories`: 每个目录对应一个 Thread，给多个Ledger开多个directory可提高写入SSD的吞吐。

    ![image-20220326234533697](../img/pulsar/bk-arch-comp-journal.png)

    

- **Write Cache**

  - JVM 写缓存，写入 Journal 之后将Entry放入缓存，并**按 Ledger 进行排序**（基于 Skiplist），方便读取。

    > 目前改成了先写 Write Cache，同时写 Journal。--> 因为有 LAC，保证不会读到脏数据。

  - 缓存满后，会被 Flush 到磁盘：写入 `Ledger Directory` （类比 KV 存储）

    > Checkpoint 定时刷新？

  - Flush 之后，Journal 即可被删除。

    > 那么 Journal 的作用是什么？仅相当于 循环Buffer、保证更高的可靠性。

    ![image-20220326234645526](../img/pulsar/bk-arch-comp-writecache.png)

    

- **Ledger Directory**

  - **Entry log**

    - Entries from different ledgers are aggregated and written sequentially, while their offsets are kept as pointers in a ledger cache for fast lookup.
    - **其实就是 write cache 的内容**，从 write cache flush 而来。

    > Q：为什么不直接存储 Journal 内容？
    >
    > - 因为一个 Journal 可能包含多个Ledger，可能造成随机读。
    > - 另外实现读写分离。

  - **Index file**

    - 每个 ledger 有一个 index 文件
    - entryId --> position 映射

    ![image-20220326235132492](../img/pulsar/bk-arch-comp-entrylog.png)
    
  - **数据清理**的问题：一个 Ledger 被删后，Entry Log 不会马上删除！！
  
    - 因为对应的 Entry Log 可能还包含其他 Ledger 的数据。





**写入流程 & 线程**

![image-20211231231801134](../img/pulsar/bookkeeper-write.png)

- **Netty 线程**
  - 处理所有 TCP 连接、分发到 write threadpool
- **Write ThreadPool**
  - 先写入 DbLedgerStorage 中的 `write cache`；成功之后再写入 Journal `内存队列`。
  - 默认线程数 = 1
- **1. DbLedgerStorage**
  - 实际上有两个 `write cache`，一个接受写入、一个准备flush，两者互切。
  - **Sync Thread**：定时 checkpoint 刷盘
  - **DbStorage Thread**：Write thread 写入 cache 时发现已满，则向 DbStorage Thread 提交刷盘操作。
    - 此时如果 swapped out cache 已经刷盘成功，则直接切换，write thread写入新的cache；
    - 否则 write thread 等待一段时间并拒绝写入请求。
- **2. Journal**
  - **Journal 线程**：循环读取内存队列，写入磁盘：group commit，而非每个entry都进行一次write系统调用
  - 定期向 `Force Write Queue` 中添加强制写入请求、触发 fsync；
  - **Froce Write Thread** ：循环从 froce write queue 中拿取强制写入请求（其中包含entry callback）、在 journal 文件上执行 fsync；
  - **Journal Callback Thread** ：fsync 成功后，执行 callback，给客户端返回 reesponse



**常见瓶颈**

- **Journal write / fsync 慢**：则 `Journal Thread `、 `Force Write Thread` 不能很快读取队列。

- **DbLedgerStorage 刷盘慢**：write cache 不能及时清空并互切。

- **Journal 瓶颈**：Journal 内存队列入队变慢，导致 `Write Thread Pool` 任务队列满、请求被拒绝。

  



### 读取

> - Apache BookKeeper Internals — Part 3 — Reads 
>   https://medium.com/splunk-maas/apache-bookkeeper-internals-part-3-reads-31637b118bf

读请求由 DbLedgerStorage 处理，一般会从缓存读取。

![image-20220101145034052](../img/pulsar/bookkeeper-read-overview.png)



读取流程：

- 根据 LedgerId + EntryId，先找到 ensemble BK 列表、计算出存储在哪些 BK。

- 读取 `Write Cache`

- 读取 `Read Cache` 

  > Read Cache 必须足够大，否则预读的entry会被频繁 evict

- 读取磁盘：

  - 找到位置信息：Entry Location Index (RocksDB)
  - 根据偏移量读取 entry 日志
  - 执行预读、写入 `Read Cache`



## || 背压

> - Apache BookKeeper Internals — Part 4 — Back Pressure
>   https://medium.com/splunk-maas/apache-bookkeeper-internals-part-4-back-pressure-7847bd6d1257



背压：通过一系列限制，防止内存占用过多。

> Backpressure 指的是在 Buffer 有上限的系统中，Buffer 溢出的现象；**它的应对措施只有一个：丢弃新事件。**
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







## || Ledger

> - A Guide to the BookKeeper Replication Protocol 
>   https://medium.com/splunk-maas/a-guide-to-the-bookkeeper-replication-protocol-tla-series-part-2-29f3371fe395 
>
> - Apache BookKeeper Internals — Part 1 — High Level: 读写流程 & 线程模型
>   https://medium.com/splunk-maas/apache-bookkeeper-internals-part-1-high-level-6dce62269125 
>
> - Apache BookKeeper Insights Part 2 — Closing Ledgers Safely
>   https://medium.com/splunk-maas/apache-bookkeeper-insights-part-2-closing-ledgers-safely-386a399d0524 //TODO
>
>   



Pulsar topic 由一系列数据分片（Segment）串联组成，每个 Segment 被称为 `Ledger (日志段)`、并保存在 BookKeeper 服务器 `bookie` 上。

- 每个 ledger 保存在多个 bookie 上，这组 bookie 被称为 ensemble；

- Ledger - Bookie 对应关系存储在 zk；



### Ledger 生命周期

> - 可视化：https://runway.systems/?model=github.com/salesforce/runway-model-bookkeeper# 

Pulsar broker 调用 BookKeeper 客户端，进行创建 ledger、关闭 ledger、读写 entry。

**Ledger 状态机**

![image-20220101224253890](../img/pulsar/bookkeeper-ledger-lifecycle.png)

- 创建 ledger 的客户端（Pulsar broker）即为这个 ledger 的 owner；**只有owner 可以往 ledger 写入数据**。
- 如果 owner 故障，则另一个客户端会接入并接管。修复 under-replicated entry、关闭 ledger. —— open ledger 会被关闭，并重新创建新 ledger



**Ledger Rollover**

- 目的：只有closed ledger才会被清理。

- 触发：写入的时候发现ledger已满，则打开新Ledger。

  - 问题：如果一个主题的写入停止了，则ledger长时间不被写入、也没办法rollover、空间无法释放。

  - 优化：引入 `maxLedgerRolloverTimeMinutes` ，超时后自动 rollover。

    > 优化后的条件：
    >
    > 1）maxRolloverTime 到期
    >
    > 2）或者，达到 maxEntries && 达到 minRolloverTime



**对比**

- **Ledger Rollover：什么时候会新建 Ledger?**

  - 1. 写满了；

  - 2. Owner Broker 故障、owner 转移，会关闭老 ledger；

  - 3. Topic offload：关闭主题并 reload，会触发 Ledger Rollover.

    > 注意，Bookie 故障后触发 Ensemble Change，只会新增 Fragment。

- **什么时候会新建 Fragment ？**

  - 1. 新建 Ledger 时；
  - 2. 写入 Bookie 失败时 `Ensemble Change`；





### 写入 ledger



![image-20220101225318769](../img/pulsar/bookkeeper-ledger-status.png)

- **Pulsar 一个主题只有一个 open 状态的 ledger；**
- 所有写操作都写入 open ledger；读操作可读取任意 ledger；



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



### Ledger Recovery

> https://medium.com/splunk-maas/apache-bookkeeper-insights-part-2-closing-ledgers-safely-386a399d0524 



> 对于 Pulsar，
>
> - 每个 topic 有一个 broker 作为 owner（注册于 zk）。该 broker 调用 BookKeeper 客户端来创建、写入、关闭 broker 所拥有的 topic 的 ledger。
> - 如果该 owner broker 故障，则ownership 转移给其他 broker；新 broker 负责关闭该topic最后一个ledger、创建新 ledger、负责写入该topic。
>
> ![image-20220102204423380](../img/pulsar/broker-failure-ledger-segment.png)



**何时触发  recovery?** 

- 每个 ledger 都有一个客户端作为 owner；如果这个客户端不可用，则另一个客户端会接入执行恢复、并关闭该 ledger。
- Pulsar：topic owner **broker 不可用**，则另一个broker接管该topic的所有权。



**防止脑裂**

- 恢复过程可能出现脑裂：客户端A (pulsar broker) 与zk断开连接，被认为宕机；触发恢复过程，由另一个客户端B来接管 ledger并恢复ledger；则有两个客户端同时操作一个 ledger。--> 可能导致数据丢失！
- **Fencing**: 客户端B 尝试恢复时，先将 ledger 设为 fence 状态，让 ledger 拒绝所有新的写入请求（则原客户端A写入新数据时，无法达到 AQ 设定的副本数）。一旦足够多的 bookie fence了原客户端A，恢复过程即可继续。



**恢复过程**

- **第一步：Fencing**

  > 将 Ledger 设为 fence 状态（OPEN --> IN_RECOVERY），并找到 LAC。

  - 新客户端 Broker2 发送 Fencing LAC 读请求：Ensemble Coverage 的 LAC 读取请求，请求中带有 fencing 标志位。
  - Bookie 收到这个 fencing 请求后，将 ledger 状态设为 fenced，并返回当前 bookie 上对应 ledger 的 LAC。
  - 一旦新客户端收到足够多的响应，则执行下一步。
    - 无需等待所有 bookie 响应，只需保证剩下的未返回 bookie 数 < AQ 即可。这样原客户端一定无法写入 AQ 个节点、亦即无法写入成功。
    - 即，收到的响应数目达到 **Ensemble Coverage** 即可：`EC = (E - AQ) + 1`

  > 为什么要找到 LAC？
  >
  > The LAC stored in each entry is generally trailing the real LAC and so finding out the highest LAC among all the bookies is the starting point of recovery.



- **第二步：Recovery reads & writes**

  > Broker2 takes the highest LAC response and then starts performing recovery reads from the LAC + 1. 
  >
  > It ensures that all entries from that point on (which may not have been previously acknowledged to the Pulsar broker) get replicated to QW bookies. Once B2 cannot read and replicate any more entries, the ledger is fully recovered.

  - **目的**：确保 LAC 之后的 entry，重新写入新的 ensemble QW。确保在关闭 ledger之前，任何已提交 entry 都被完整复制。
  - 客户端Broker2从 **LAC + 1** 处开发发送 `recovery 读请求`，读到之后将其重新写入 bookie ensemble（写操作是幂等的，不会造成重复）。重复这个过程，直到客户端读不到任何 entry。
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





# | 功能

## || Deleyed Message



-  作用：发送时，可以给每个消息配置不同的延迟时间。在一段时间之后消费消息，而不是立即消费。

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

  - 消息存储到 BookKeeper后，`DelayedDeliveryTracker` 在堆外内存**优先级队列**中维护索引 (time ->  LedgerId + EntryId)
  - 当消费时，如果消息为delay，则放入`DelayedDeliveryTracker` ；消费时还会查询 `DelayedDeliveryTracker` 获取到期消息。

  ![image-20220327191641313](../img/pulsar/pulsar-delayed-msg.png)

  - 注意：只能作用于 shared mode

- **限制：**

  - 内存占用；

  - Broker 宕机后需要重建索引、新 Broker 有一段时间会无响应。

    > 增加分区可缓解，让每个分区的数据尽可能小。

  - 索引是在subscription维度，可能有重复；

  

- **优化 - PIP26 Hieraychical Timing Wheels**

  > http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf 
  >
  > https://blog.acolyer.org/2015/11/23/hashed-and-hierarchical-timing-wheels/

  - 自定义延迟精度，并分片；只有最近的分片存储在内存、其他的持久化
    ![image-20220327195558877](../img/pulsar/pulsar-delayed-msg-plan.png)
  - 取到M9时，会查看时间片 time-partition-0 是否有消息到期、读到M8。

- **挑战**

  - 如何清理延时消息？目前必须ack之后才能删
  - Safe position for subscription to start read? 从哪里开始读既不丢消息、又不影响索引重建？
  - 内存里需要维护 too much individual acks, introduced much memory overhead. 



## || Geo Replication

> - https://pulsar.apache.org/docs/en/administration-geo

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

  - `--allowed-clusters`：配置 **tenant**，使其有权限使用上述clusters

    ```sh
    $ bin/pulsar-admin tenants create my-tenant \
      --admin-roles my-admin-role \
      --allowed-clusters us-west,us-east,us-cent
    ```

  - `set-clusters`：在 **namespace** level 指定在哪些 clusters 之间复制

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
  
- 如何避免循环复制：消息包含元数据 replicate-from
    - 如何保证 exact-once 复制：broker 去重 with sequence id。
    
    
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

  - 要复制ack状态：目前只复制 mark delete position（连续ack的最大id）；切换后可能有重复消费。

    > ACK：消费进度会被持久化到 ledger。
    >
    > ![image-20220330093458342](../img/pulsar/msg-ack-cursers.png)

- 实现

  - **Cursor Snapshot**：定期同步，记录message id 对应关系

    > ClusterA 向 B/C 发送 `ReplicatedSubscriptionSnapshotRequest`，收到响应包含 ledger_id / entry_id；则 ClusterA 会保存本地 ledger_id / entry_id，以及其他 cluster对应的 ledger_id / entry_id。
    >
    > Cursor mapping 并不能做到精确，因为请求响应会有延时。
    >
    > ![image-20220330094251420](../img/pulsar/subs-replicate-cursor-snapshot.png)
    >
    > ```json
    > {
    >   "snapshot_id": "444D3632-F96C-48D7-83D8-041C32164EC1",
    >   "local_message_id": {
    >     "ledger_id": 192,
    >     "entry_id": 123123
    >   },
    >   "clusters": [
    >     {
    >       "cluster": "b",
    >       "message_id": {
    >         "ledger_id": 1234,
    >     		"entry_id": 45678
    >       }
    >     }, {
    >       "cluster": "c",
    >       "message_id": {
    >         "ledger_id": 7655,
    >     		"entry_id": 13421
    >       }
    >     }
    >   ]
    > }
    > ```
    
  - **Update Remote Cursor**：当 clusterA 的 ack 进度超过之前 snapshot 位置，则发送 `ReplicatedSubscriptionUpdate` 请求通知其他集群更新 mark delete position。



- Cursor Snapshot 如何存储

  - 和正常的消息穿插存储： `Snapshot Marker`

  - 与正常的消息一样进行跨地域复制：实现跨地域请求响应

  - 副作用：影响backlog计算；

    ```json
    --示例
    {
      "subscription_name": "my-subs",
      "clusters": [
        {
          "cluster": "cluster-1",
          "message_id" : {
            "ledger_id": 11,
            "entry_id": 4567
          }
        }, {
          "cluster": "cluster-2",
          "message_id" : {
            "ledger_id": 22,
            "entry_id": 5678
          }
        }
      ]
    }
    ```

    

- 配置

  - Broker 启用：`enableReplicatedSubscriptions=true` (默认true)

  - 可配置参数：多久做一次snapshot、snapshot复制请求的timeout时间、最多缓存多少个snapshot；

  - 创建subscription时启用：
  
    ```java
    Consumer<String> consumer = pulsarClient.newConsumer(Schema.STRING)
                  .topic(topic)
                  .subscriptionName("my-subscription")
                  .replicateSubscriptionState(true)
                .subscribe();
    ```

- 限制
  - 定期snapshot
  - 只同步 mark delete position
  - 仅当所有相关cluster可用时才会snapshot
  - 影响backlog计算，另外 batch 也会影响backlog 



## || Tiered Storage

> - PIP: https://github.com/apache/pulsar/wiki/PIP-17:-Tiered-storage-for-Pulsar-topics 
> - Doc: https://pulsar.apache.org/docs/en/tiered-storage-overview/



- 目的：将较旧的数据从 BK 移动到其他廉价存储中。

- 触发存储老 segment:

  - 当 Ledger 被关闭时，Offloader 判断Ledger 堆积数是否超过阈值；ManagedLedger 调用具体offloader循环卸载数据。

    > Once a segment has been sealed it is immutable. I.e. The set of entries, the content of those entries and the order of the entries can never change. 

- 读取老 segment
  
  - 数据虽然被卸载，但元数据还保存在 ZK；
  
  - ManagedLedger 默认用 BK 作为数据源，如果元数据表明已卸载，则返回 Offloader 的数据源句柄。
  
    > For Pulsar to read back and serve messages from the object store, we will provide a `ReadHandle` implementation which reads data from the object store. 
    >
    > With this, Pulsar only needs to know whether it is reading from bookkeeper or the object store when it is constructing the `ReadHandle`.



## || Function Mesh

**Function**

https://pulsar.apache.org/docs/en/schema-get-started/

- 作用

  - 消费主题中的数据，进行处理，再写入另一个主题。
  - 输入输出都必须是 Pulsar Topic。

- Runtime

  - **线程**：`ThreadRuntime` 将 java function 包装成一个Runnable。
  - **进程**：`ProcessRuntime` 调用 Java ProcessBuilder 创建一个进程对象；新进程暴露 gRPC服务，提供健康检查接口。
  - **K8S**：`KubernetesRuntime` 创建 Headless Service，为每个Function创建一个StatefulSet，让Function在Pod中运行。

- 特点：可配置三种语义保障

  - At Most Once:：收到消息即 ack
  - At Least Once：Function处理成功才ack
  - Effectively Once：数据去重

- 原理：三个内部主题

  - **提交**

    - 构造 FunctionConfig：tenant/ns/name, inputs/output, classsName

    - 可以提交到任意 worker；

    - worker 校验：检查配置、检查代码

    - worker 拷贝代码 jar 到 BookKeeper. 

      >  Q：bk 如何存储？
      >
      > Map<FQFN, FunctionMetaData> 存储到元数据 **topic**，也可以作为 audit log。

  - **调度**

    - 只有 Leader worker 才能调度；

      > 选主：failover subscription，订阅 "empty coordination **topic**"

    - 触发调度的时机：Function CRUD 操作、Worker变化：新增、选主、宕机

    - 任务分配：Leader 写入 ”Assignment **Topic**“，是压缩主题 key = FQFN + InstanceId，所有 Worker 订阅该主题。

  - **执行**

    - Worker 监听到 ”Assignment Topic“ 后，由`RuntimeSpawner` 管理Function生命周期。

![image-20220502193156731](../img/pulsar/pulsar-function-flow.png)

**Function Mesh**

- 描述：用 yaml 描述一组 Function 之间的关系。
- 管理：
  - 用 pulsar admin 来管理：`./pulsar-admin function-mesh create -f mesh.yaml`
  - K8S 环境管理：`kubectl apply -f function-mesh.yaml`

- 原理：新增一个 `MeshTopic`，然后基于 Function 已有的三个topic.



**Pulsar IO (connector)**

https://pulsar.apache.org/docs/en/io-overview/

- 作用：数据经过转换之后存入外部数据源。



## || Transactions

> - https://pulsar.apache.org/docs/en/txn-how/
> - PIP-31 Transactional Streaming https://docs.google.com/document/d/145VYp09JKTw9jAT-7yNyFU255FptB2_B2Fye100ZXDI/edit#heading=h.bm5ainqxosrx

目的

- 在原子性事务里消费、处理、生产。

原理

- 概念
  - **Transaction Coordinator**：broker内部运行的一个模块，负责管理tx生命周期、处理tx超时。
  - **Transaction Log**：底层是 pulsar topic，存储 tx 状态。
  - **Transaction Buffer**：生产的消息先放到相应主题分区的 buffer、只有提交后才对消费者可见。
  - **Transaction ID**：高16位 = TC id，低 112 位 = 自增ID。
- 数据流程
  - 启动事务
    - Pulsar 客户端找到 TC，请求创建tx；TC 创建事务日志，返回 tx ID；
  - 生产消息
    - 先发送 topic partition 到TC；TC 写入事务日志；
  - Ack 消息
    - 先通过 TC 记录事务日志；再发送ack 到broker；
    - Broker 检查是否属于某个事务，标记为 PENDING_ACK；
  - 结束事务
    - 先通过 TC 记录事务日志：COMMITING / ABORTING
    - 负责生产的 Broker 写入 committed markers
    - 负责ack的Broker 将 PENDING_ACK 置为 ACK 或 UNACK
    - TC 最后将结果写入事务日志：COMMITTED / ABORTED



## || KoP

> - KoP 介绍 https://streamnative.io/blog/tech/2020-03-24-bring-native-kafka-protocol-support-to-apache-pulsar/ 
> - 连续偏移量 https://streamnative.io/blog/engineering/2021-12-01-offset-implementation-in-kafka-on-pulsar/ 

**如何实现连续 offset**

- bookie 存储内容新增 BrokerEntryMetadata 

  ```protobuf
  message BrokerEntryMetadata {
    optional uint64 broker_timestamp = 1;
    optional uint64 index = 2; //offset
  }
  ```

- FETCH

  - 直接读 Bookie，解析 BrokerEntryMetadata 即可；-- 二分查找

- PRODUCE

- - Managed Ledger 写入 Bookie 成功后有一个回调 ` public void addComplete(Position pos, ByteBuf entryData, Object ctx)`，可以从 entryData 中解析 BrokerEntryMetadata，返回 index 即可。

- COMMIT_OFFSET



## || SQL 

> https://pulsar.apache.org/docs/en/sql-overview/

查询办法

- CLI: `bin/pulsar sql`
- presto-jdbc
- Presto HTTP API
- 工具：Metabase，查询、图表



性能调优

-  **增加分区个数**，查询时指定分区；类似数据库分表。
  `where __partition__ = 1`
- **按发送时间查询**：底层使用二分法
  `where __publish_time__ > timestamp '2020-01-01 09:00:00'`
- **配置限流**，避免影响puslar正常读写 `/conf/presto/catalog/pulsar.properties` `pulsar.bookkeeper-throttle-value=`
- Presto 直接查询 bk 历史数据，是否对消息吞吐量产生影响？
  - 会。可以 offload 到二级存储，presto查二级存储；或replicate 到历史数据集群。 



监控

- 8081 端口提供 dashboard



# | 运维

## || 安装

https://pulsar.apache.org/docs/zh-CN/standalone/ 



Pulsar Manager

- https://pulsar.apache.org/docs/en/administration-pulsar-manager/

- 注意容器部署时cluster url：

  ```sh
  ./pulsar-admin clusters update standalone --url http://docker.for.mac.host.internal:8080 --broker-url pulsar://127.0.0.1:6650
  ```

   



**Docker Standalone 部署**

- 仓库：https://hub.docker.com/r/apachepulsar/pulsar

- 注意修改 advertisedAddress为container name，否则默认是主机名

  ```sh
  $ docker run -it 
    -p 6650:6650  
    -p 8080:8080 
    --mount source=pulsardata,target=/pulsar/data 
    --mount source=pulsarconf,target=/pulsar/conf 
    -e advertisedAddress=pulsar-standalone
    --name pulsar-standalone
    --hostname pulsar-standalone
    apachepulsar/pulsar:2.10.0 
    sh -c "bin/apply-config-from-env.py conf/standalone.conf && bin/pulsar standalone"
    
  ```



**Docker compose 部署**

> https://github.com/apache/pulsar/tree/master/docker-compose/kitchen-sink 

- 部署 zk, bk, broker, proxy, manager

- 注意修改 advertisedAddress



**K8S 部署**

> https://pulsar.apache.org/docs/en/helm-overview/
>
> https://pulsar.apache.org/docs/en/getting-started-helm/ 
>
> https://github.com/apache/pulsar-helm-chart

- 安装 helm：

  ```sh
  helm repo add apache https://pulsar.apache.org/charts
  helm install pular apache/pulsar --set initialize=true
  ```

- 可选端口映射到本地，通过本地地址访问

  ```sh
  kubectl port-forward svc/pulsar-pulsar-manager 9527:9527 -n default
  ```

- 如何自定义

  - 默认使用 https://github.com/apache/pulsar-helm-chart/blob/master/charts/pulsar/values.yaml 
  - 可修改 replicaCount, images




测试

```shell
./pulsar-admin clusters list
./pulsar-admin brokers list my-cluster
./pulsar-admin topic list public/default

# 生产消费
./pulsar-client produce my-topic --messages "hello pulsar"
./pulsar-client consume my-topic -s "my-subscription"

# perf 
./pulsar-perf produce -r 100 -n 2 -s 1024 my-tpic
./pulsar-perf consume my-tpic
```





## || 开发调试

启动入口

- bin/pulsar broker --> `PulsarBrokerStarter`
- Bin/pulsar standalone

服务端请求入口

- `ServerCnx`

客户端请求入口

- `ClientCnx`

控制面入口

- 服务端：pulsar-broker `xx.admin.v2.Brokers`
- 客户端：pulsar-client-admin





## || 监控

**查看主题统计**

```sh
./pulsar-admin topics stats TOPIC_NAME
./pulsar-admin topics stats-internal TOPIC_NAME #包含更多内部参数，例如ledger,cursor
```

- backlogSize / storageSize
- Publishers: 吞吐量，发送速率，地址，producerName
- Subscriptions: 吞吐量，消费速率，msgBacklog，type，
  - 该订阅下有哪些consumer、consumerName、lastAckTs、lastConsumeTs



**查看 BK**

```sh
./bookkeeper shell listledgers
./bookkeeper shell ledgermetadata -l LEDGER_ID
```

- ensembleSize / writeQuorunSize / ackQuorunSize
- ensembles : bk 节点列表
- lastEntryId 
- state
- managed-ledger: topic name -base64编码



**Pulsar-manager 监控 BookKeeper:**

```
http://localhost:7750/bkvm/
```

- Bookie 列表：Usage / 可用空间

- Ledger 列表：大小、age、replication、Ensemble、WQ、AQ

  



**Dashboard**

- https://github.com/streamnative/apache-pulsar-grafana-dashboard 



## || 性能调优

> - Penghui https://mp.weixin.qq.com/s/MSPVfH085qTlwqk2IWEBjA
> - Penghui https://mp.weixin.qq.com/s/R87gFB8tG92i6uvIPU3pOg 
> - BIGO 调优实战 https://mp.weixin.qq.com/s/mJViU-elhBwHMDiius2b8g



- **Batched Message** 

  - 一个entry存放一个batch，index 也变小
  - 读取 性能也更好

- **Producer Partition Switch 频率减少**

  - ```java
    client.newProducer()
      .topic("xx")
      .enableBatching(true)
      .batchingMaxBytes(128 * 1024 * 1024)
      .batchingMaxMessages(1000)
      .batchingMaxPublishDelay(2, MILLISECONDS)
      .blockIfQueueFull(true)
      .roundRobinRouterBatchingPartitionSwitchFrequency(10) //切换频率 10ms
      .batcherBuilder(BatcherBuilder.DEFAULT)
      .create();
    ```

  - batch 发送必须 sendAsync()，或者多线程 send()

- **Producer pending queue 增大**

  - 客户端在等待ack过程中有足够buffer继续接受写入
  - Vs. batched? 

- **Message Compression**

- **BK 消息持久化配置**

  - 增加 E > QW / QA，条带化写入；一个topic使用更多的 bookie
  - 减少 QA，忽略最慢的 bookie；

- **消息写入优化**

  - Broker configurations
    
  ```
    0. 增加 bk flush 间隔（memtable定期flush到 Entry log的间隔）：`dbStorage_writeCacheMaxSizeMb / flushInterval`
    1. managedLedgerDefaultEnsembleSize
    2. managedLedgerDefaultWriteQuorum
    3. managedLedgerDefaultAckQuorum
    4. managedLedgerNumWorkerThreads
    5. numIOThreads
    6. Dorg.apache.bookkeeper.conf.readsystemproperties=true -DnumIOThreads=8
  ```
  
  - Bookie configurations

    ```
    0. Journal 目录和 Entry 目录存储到不同磁盘，利用多个磁盘的 IO 并行性；
    1. Journal Directories: 指定多个 Journal 目录，否则bk使用单线程处理每个journal目录；
    2. Ledger Directories
    3. Journal sync data // 禁用同步刷盘 `journalSyncData=false`，entry 写入page cache后即返回。
    4. Journal group commit // 启用组提交机制：`journalAdaptiveGroupWrites=true`
    5. Write cache
    6. Flush interval
    7. Add worker threads and max pending add requests
    8. Journal pagecache flush interval
    ```
    
    

- **消息读取优化**

  - Consumer receiver queue 增大

  - Key_Shared 时，dispatcher 可能瓶颈

    > 计算hash、group by hash%slots。
    >
    > 一批读得越多性能越好。

  - Bookie configurations

    ```
    
    ```
  1. dbStorage_rocksDB_blockCacheSize
    2. dbStorage_readAheadCacheMaxSizeMb
  3. dbStorage_readAheadCacheBatchSize
    4. Read worker threads
  ```
  
  - Broker congifurations 
  
  ```
  1. Managed ledger cache
    2. Dispatcher max read batch size
    3. Bookkeeper sticky reads
    ```
    
    
    
    - `managedLedgerNewEntriesCheckDelayInMillis`：broker间隔多久检查一次是否有新entry要push给消费者。
    
  - 优化追尾读：设置缓存大小、缓存逐出策略
  
  - 优化追赶度：
  
    - bk 使用单线程读取同一个 Ledger，可设置 worker 线程池 `numReadWorkerThreads` `maxPendingReadRequestsPerThread`
    - 设置 RocksDB 块缓存 `dbStorage_rockDB_blockCacheSize`
    - 设置 Entry 预读缓存：`dbStorage_readAheadCacheMaxSizeMb / BatchSize`
    ```





