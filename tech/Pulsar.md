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

## || Producer

https://pulsar.apache.org/docs/en/concepts-messaging/

配置

- **Synd Mode**
  - **sync send**: waits for an acknowledgement from the broker after sending every message.
  - **async send**: puts a message in a blocking queue and returns immediately.

- **Access Mode**

  - **Shared**: Multiple producers can publish on a topic.

  - **Exclusive**: Only one producer can publish on a topic.

  - **WaitForExclusive**: If there is already a producer connected, the producer creation is pending (rather than timing out) until the producer gets the `Exclusive` access. 

    > 类似选主。if you want to implement the leader election scheme for your application, you can use this access mode.



功能

- **Batching**

  - 最大消息个数

  - 最大发送延迟

  - batch是个整体

    - batches are tracked and *stored* as single units rather than as individual messages.
    - Consumer unbundles a batch into individual messages. In general, a batch is acknowledged when all of its messages are acknowledged by a consumer. 
    - 一个消息ack失败，会导致整个batch重发。2.6.0 之后引入 batch index acknowledgement 解决重复发送问题：消费者发送 batch index ack request. 

    

- **Chunking**

  - 作用：生产者将大payload拆分、消费者组装

  - 1. The producer splits the original message into chunked messages and publishes them with chunked metadata to the broker separately and in order.
    2. The broker stores the chunked messages in one managed-ledger in the same way as that of ordinary messages, and it uses the `chunkedMessageRate` parameter to record chunked message rate on the topic.

    3. The consumer buffers the chunked messages and aggregates them into the receiver queue when it receives all the chunks of a message.

    4. The client consumes the aggregated message from the receiver queue.

  - 限制

    - Chunking is only available for persisted topics.  
    - Chunking is only available for the exclusive and failover subscription types. 
    - Chunking cannot be enabled simultaneously with batching.



- **Deduplication**

  - 生产者多次发送同样的消息，只会被保存一次到bookie。

  - 配置：https://pulsar.apache.org/docs/en/cookbooks-deduplication/ 

  - 可用于 effectively-once  语义

    > https://www.splunk.com/en_us/blog/it/exactly-once-is-not-exactly-the-same.html 

  

- **Deleyed Message Delivery**

  - 作用：在一段时间之后消费消息，而不是立即消费

    ```java
    // message to be delivered at the configured delay interval
    producer.newMessage()
      .deliverAfter(3L, TimeUnit.Minute)
      .value("Hello Pulsar!")
      .send();
    ```

    

  - 原理：

    - 消息存储到 BookKeeper后，`DelayedDeliveryTracker` 在内存中维护索引 (time -> messageId)
    - 当消费时，如果消息为delay，则放入`DelayedDeliveryTracker`

  - 注意：只能作用于 shared mode



## || Consumer



配置

- **Receive Mode**
  - **Sync Receive**: blocked until a message is available.
  - **Async Receive**: returns immediately with a future value.



功能

- **Acknowledgement**

  - Being acknowledged **individually**. With individual acknowledgement, the consumer acknowledges each message and sends an acknowledgement request to the broker.
    `consumer.acknowledge(msg);`

  - Being acknowledged **cumulatively**. With cumulative acknowledgement, the consumer only acknowledges the last message it received. All messages in the stream up to (and including) the provided message are not redelivered to that consumer.
    `consumer.acknowledgeCumulative(msg);`

  - **Negative Ack**: 表示处理失败、稍后重发。
    `consumer.negativeAcknowledge(msg);`

    > Q: 何时重新deliver、能否指定? 

  - **Ack timeout**: 可配置对unack消息自动重发。the client tracks the unacknowledged messages within the entire `acktimeout` time range, and sends a `redeliver unacknowledged messages` request to the broker automatically when the acknowledgement timeout is specified.

    > Q: 由消费者请求重发，而不是broker主动推送？

    

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

  

  

- 

  

## || Topic

命名：`{persistent|non-persistent}://tenant/namespace/topic`



类型

- Persistent
- Non-persistent



**Namespace**

-  The administrative unit of the topic, which acts as a grouping mechanism for related topics. 
- Most topic configuration is performed at the [namespace](https://pulsar.apache.org/docs/en/concepts-messaging/#namespaces) level. 
- Each tenant has one or multiple namespaces.



**Partitioned Topic**

- 普通 Topic 只对应一个broker，限制了吞吐量；而 Partitioned Topic 则可被多个 broker 处理；
- 实现：N 个内部主题。
- routing mode: 决定生产到哪个分区；
  - RoundRobinPartition
  - SinglePartition：随机
  - CustomPartition
- subscription mode: 决定从哪个分区读取；



**Non-persistent Topic**

- 普通 Topic 存储消息到 bookie，而non-persistent topic则只存到内存
- 更快









## || Subscription

**订阅模式**

![image-20220322112234641](../img/pulsar/subscription-modes.png)

- **Exclusive**

  - 只有一个消费者绑定到当前订阅。

  - In *exclusive* mode, only a single consumer is allowed to attach to the subscription. If multiple consumers subscribe to a topic using the same subscription, an error occurs.

  - > ![image-20220322112520553](../img/pulsar/subscription-modes-exclusive.png)

- **Failover**

  - 多个消费者可以绑定到当前订阅，但只有一个收到消息。

  - In *failover* mode, multiple consumers can attach to the same subscription. A master consumer is picked for non-partitioned topic or each partition of partitioned topic and receives messages. 

  - When the master consumer disconnects, all (non-acknowledged and subsequent) messages are delivered to the next consumer in line.

    > ![image-20220322112744805](/Users/alpha/dev/git/alpha/alpha-notes/img/pulsar/subscription-modes-failover.png)

- **Shared** 

  - 多个消费者可以绑定到当前订阅，按 round-robin 模式接收消息。

  - In *shared* or *round robin* mode, multiple consumers can attach to the same subscription. Messages are delivered in a round robin distribution across consumers, and any given message is delivered to only one consumer. 

  - When a consumer disconnects, all the messages that were sent to it and not acknowledged will be rescheduled for sending to the remaining consumers.

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
    - 无法使用 cumulative ack
    - 必须禁用 batching，或者使用 *key-based batching*

  - > ![image-20220322113231815](/Users/alpha/dev/git/alpha/alpha-notes/img/pulsar/subscription-modes-key-shared.png)











# | Broker

- 



# | BookKeeper



## || 概念

> https://bookkeeper.apache.org/docs/latest/getting-started/concepts/ 

- `entry`：一条日志记录。each unit of a log is an *entry*; Each entry has the following fields:
  - Ledger Id
  - Entry Id
  - LC, Last Confirmed：上次记录的 entry id
  - Data
  - Authentication code
- `ledger`：一组日志记录。streams of log entries are called *ledgers*
- `bookie`：存储 ledger的服务器。individual servers storing ledgers of entries are called *bookies*
  - 每个 bookie 存储部分 ledger *fragment*, 而非完整ledger



**三种文件类型**

- `Journal`
  - 事务日志。在修改 ledger 之前，先记录事务日志。
- `Entry log`
  - An entry log file manages the written entries received from BookKeeper clients. 
  - Entries from different ledgers are aggregated and written sequentially, while their offsets are kept as pointers in a ledger cache for fast lookup.
- `Index file`
  - 每个 ledger 有一个 index 文件



## || Ledger

> https://medium.com/splunk-maas/a-guide-to-the-bookkeeper-replication-protocol-tla-series-part-2-29f3371fe395 



Pulsar topic 由一系列数据分片（Segment）串联组成，每个 Segment 被称为 `Ledger`、并保存在 BookKeeper 服务器 `bookie` 上。

- 每个 ledger 保存在多个 bookie 上，这组 bookie 被称为 ensemble (?)；

- Ledger - bookie 对应关系存储在 zk；



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

- Pulsar 一个主题只有一个 open 状态的 ledger；
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
- Pulsar：topic owner broker 不可用，则另一个broker接管该topic的所有权。



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



