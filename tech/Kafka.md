# 消息队列

## 作用

### 异步处理

- 提升总体性能：

- 更快返回结果；

- 减少等待，各步骤实现并发

### 削峰填谷

- 消息堆积，消费端按自己的能力处理

代价

- 增加调用链，总体时延变长

- 上下游都要改成异步消息，增加复杂度



### 服务解耦



## 消息模型

- 队列

- 发布订阅

#### 含义

##### Streaming Platform

###### 发布订阅数据流

###### 存储数据流

###### 处理数据流

##### 对比

###### vs. 消息系统

####### 集群，自由伸缩

####### 数据存储

####### 流式处理

###### vs. hadoop

####### 批处理 --> 流处理

###### vs. ETL

#### 特点

##### 支持多生产者

###### 生产者以相同格式 写入主题，消费者即可获得单一流

##### 支持多消费者

###### 消费者组之间互不影响

##### 基于磁盘的数据存储

Disk-Based Retention


###### 消费者可暂时离线

##### 伸缩性

##### 高性能

#### 使用场景

##### Activity tracking

###### 用户行为日志

##### Messaging

###### 用户通知：格式化、聚合、Preference

##### Metrics & Logging

##### Commit log

###### Change Log

###### 可设置为log-compacted，为每个key只保留一个变更数据

##### Stream processing

###### 实时 Map Reduce

## 组件

### 主题 / 分区

#### 主题 Topic

##### 作用

###### 类似数据库里的表

#### 分区 Partition

##### 作用

###### 实现数据冗余 Redundancy

####### 每个 Partition 可以分布在不同机器上：Replica

###### 实现伸缩性 Scalability

####### 一个 Topic 包含多个 Partition

####### 类比 Sharding

##### 特点

###### 分区内保证顺序

###### 每个分区是一组有序的消息日志

####### 消息日志的定期回收机制

######## Log Segment

##### 自定义分区策略

###### 实现Partitioner接口

int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);


####### 根据key选择partition

###### 策略

####### 轮询策略 Round Robin

####### 随机策略 Randomness

```
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);

return ThreadLocalRandom.current().nextInt(partitions.size());

```

####### 按消息键保序策略 Key-Ordering

```
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);

return Math.abs(key.hashCode()) % partitions.size();

```

######## 同一个key的所有消息都进入相同分区

#### 分区副本 Replica

##### 作用

###### Availability

####### 数据冗余，提升可用性、容灾能力

###### 备份

####### 同一分区下所有副本保存“相同的”消息序列

##### 副本机制

###### 角色

####### Leader Replica

######## 负责读写

######## 每个 partition只有一个 leader replica

######### 保证 Consistency

######## vs. Preferred Replica

######### 最先创建的副本。

######### 如果 auto.leader.rebalance.enable=true，如果Preferred Replica不是当前Leader，而它又在ISR中；则触发leader选举、将Preferred Replica选为Leader

######### 如果手工reassign replica，注意第一个replica均匀分布在各Broker上

####### Follower Replica

######## 只负责与leader同步（异步拉取）

为什么Follower不提供读服务？

消息写入主节点后，等待ISR个副本都复制成功后才会返回。

######### 异步！

######### 拉取！

########## 发送 Fetch 请求 == Consumer 消费消息

######## 不对外提供服务

######### 好处

########## 实现 Read Your Writes

########### 写入后马上读取，没有lag

########## 实现单调读 Monotonic Reads

########### 不会看到某条消息一会儿存在一会儿不存在

######## 会滞后

###### 分类

####### AR: Assigned Replicas

######## 所有副本集合

####### ISR: In-Sync Replicas

######## 只有ISR集合中的副本才可能选为新leader

######## 判断是否 In-Sync

Kafka在启动的时候会开启两个任务，一个任务用来定期地检查是否需要缩减或者扩大ISR集合，这个周期是replica.lag.time.max.ms的一半，默认5000ms。当检测到ISR集合中有失效副本时，就会收缩ISR集合，当检查到有Follower的HighWatermark追赶上Leader时，就会扩充ISR。

除此之外，当ISR集合发生变更的时候还会将变更后的记录缓存到isrChangeSet中，另外一个任务会周期性地检查这个Set,如果发现这个Set中有ISR集合的变更记录，那么它会在zk中持久化一个节点。然后因为Controllr在这个节点的路径上注册了一个Watcher，所以它就能够感知到ISR的变化，并向它所管理的broker发送更新元数据的请求。最后删除该路径下已经处理过的节点。

######### replica.lag.time.max.ms

########## 默认 10s

######### 是判断落后的时间间隔，而非落后的消息数！

####### OSR: Out-of-Sync Replicas

##### Lead Election

###### 正常领导者选举

####### 当Leader挂了，zk感知，Controller watch，并从ISR中选出新Leader

####### 其实并不是选出来的，而是Controller指定的

###### Unclean 领导者选举

####### 当ISR全空，如果unclean.leader.election.enable=true

######## 从OSR中选出领导者

####### 问题

######## 提高可用性的代价：消息丢失！

######## CP --> AP

######### 通过配置参数，实现选择 C vs. A

##### 可优化的点

###### 提高伸缩性

####### 让follower副本提供读功能？

######## 没必要

######### Leader分区已经均匀分布在各Broker上，已有负载均衡；不像MySQL压力都在主上

######### 而且位移管理会更复杂

###### 改善数据局部性

####### 把数据放在与用户地理位置近的地方

##### 副本同步

###### 高水位 High Watermark

####### 取值

######## = ISR集合中最小的 Log End Offset (LEO)

LEO:
表示副本写入下一条消息的位移值。


######### 是一个特定的偏移量

######### 分区的高水位  == Leader 副本的高水位

######## 日志末端位移 LEO (Log End Offset)

######### 表示副本写入写一条消息的位移值

####### 作用

######## 定义消息可见性

######### 消费者只能拉取高水位之前的消息

########## 高水位以上的消息属于未提交消息

########## 即：消息写入后，消费者并不能马上消费到！

######## 实现异步的副本同步

https://time.geekbang.org/column/article/112118



####### 更新机制

######## Leader副本

######## Follower副本

####### 思想

######## Kafka 复制机制既不是完全的同步复制，也不是单纯的异步复制

同步复制要求所有能工作的follower都复制完，这条消息才会被commit，这种复制方式极大的影响了吞吐率。

而异步复制方式下，follower异步的从leader复制数据，数据只要被leader写入log就被认为已经commit，这种情况下如果follower都还没有复制完，落后于leader时，突然leader宕机，则会丢失数据。

而Kafka的这种使用ISR的方式则很好的均衡了确保数据不丢失以及吞吐率。



######## Redis 属于同步复制？

###### 副本同步机制

####### 流程

######## ISR

######### 同步更新Follower？

######### 1. Leader收到消息后，写入本地，并复制到所有Follower

########## 靠 Follower 异步拉取

######### 2. 如果某个Follower落后太多，则从ISR中删除

######## Leader 副本

######### 处理生产者写入消息

########## 1. 写入本地磁盘

########## 2. 更新分区高水位值

########### HW = max{HW, min(所有远程副本 LEO)}

######### 处理Follower副本拉取消息

########## 1. 读取磁盘/页缓存中的消息数据

########## 2. 更新远程副本LEO值 = 请求中的位移值

########## 3. 更新分区高水位置

########### HW = max{HW, min(所有远程副本 LEO)}

######## Follower 副本

######### 异步拉取

########## 1. Leader副本收到一条消息，更新状态

########### Leader

############ HW = 0, LEO = 1, RemoteLEO = 0

########### Follower

############ HW = 0, LEO = 0

########## 2. Follower拉取消息（featchOffset = 0）

###########  Follower 本地操作

############ 写入磁盘

############ 更新 LEO

############ 更新 HW

############# HW = min{Leader HW, 本地LEO}

########### Follwer

############ HW = 0, LEO=1

########### 至此 Leader/Flower的高水位仍然是0

############ 需要在下一轮拉取中被更新

########## 3. Follower再次拉取消息（featchOffset = 1）

########### Leader

############ HW = 1, LEO = 1, RemoteLEO = 1

########### Follower

############ HW = 1, LEO = 1

############ Follower收到回复后更新 HW

####### 高水位的问题

######## 可能出现数据丢失

######### 场景1

########## 1. Follower高水位尚未更新时，发生重启

########## 2. 其重启后，会根据之前的HW来更新LEO，造成数据截断 

########## 3. Follower再次拉取消息，理应会把截断的数据重新拉过来；但是假如此时Leader宕机，Follower则成为新的Leader

########## 只会发生在 min.insync.replicas = 1 时？？？

######### 场景2

########## 同时重启

假设集群中有两台Broker，Leader为A，Follower为B。A中有两条消息m1和m2，他的HW为1，LEO为2；B中有一条消息m1，LEO和HW都为1.假设A和B同时挂掉，然后B先醒来，成为了Leader（假设此时的min.insync.replicas参数配置为1）。然后B中写入一条消息m3，并且将LEO和HW都更新为2.然后A醒过来了，向B发送FetchrRequest，B发现A的LEO和自己的一样，都是2，就让A也更新自己的HW为2。但是其实，虽然大家的消息都是2条，可是消息的内容是不一致的。一个是(m1,m2),一个是(m1,m3)。

这个问题也是通过引入leader epoch机制来解决的。

现在是引入了leader epoch之后的情况：B恢复过来，成为了Leader，之后B中写入消息m3，并且将自己的LEO和HW更新为2，注意这个时候LeaderEpoch已经从0增加到1了。
紧接着A也恢复过来成为Follower并向B发送一个OffsetForLeaderEpochRequest请求，这个时候A的LeaderEpoch为0。B根据0这个LeaderEpoch查询到对应的offset为1并返回给A，那么A就要对日志进行截断，删除m2这条消息。然后用FetchRequest从B中同步m3这条消息。这样就解决了数据不一致的问题。

######## 原因

######### Leader/Follower的高水位更新存在时间错配

######### 因为Follower的高水位要额外一次拉取才能更新

######## 解决

######### Leader Epoch

########## 取值

########### 大版本：单调递增的版本号

############ 领导变更时会增加大版本

########### 小版本：起始位移，leader副本在该Epoch值上写入的首条消息的位移

########## 目的

########### 规避高水位可能带来的数据丢失

########### 流程

############ 2. Follower重启后，先向Leader请求Leader的LEO

############# 会发现缓存中没有 > Leader LEO的Epoch；所以不做日志截断

############ 3. Leader重启后，新的Leader会更新Leader Epoch

### Broker

#### 控制器 Controller

##### 定义

###### 集群中的某一个Broker会作为控制器

####### 唯一。在zk的帮助下，管理和协调“整个”kafka集群

###### 重度依赖 ZK 

####### 写入临时节点 /controller

####### 选出新 controller后，会分配更高的 “controller epoch”

######## 如果broker收到了来自老epoch的消息，则会忽略

##### 作用

###### Partition 领导者选举

####### 当有 Broker离开

######## 1. 失去leader的分区需要一个新leader

######## 2. 控制器会遍历这些分区，确定谁将成为新leader

######## 3. 控制器然后向包含新leader、原有follower的broker发送请求，告知新的leader/follower信息

######## 4. 新 leader处理生产者消费者请求、follower则从新leader复制消息

####### 当有Broker加入

######## 1. 控制器根据 Broker ID 检查新broker是否已有repilcas

######## 2. 如果已有，则通知 brokers，新broker则会从已有的leader处复制消息

###### 主题管理

####### 创建、删除、增加分区

######## kafka-topics 执行时，后台大部分工作是控制器完成

###### 分区重分配

####### kafka-reassign-partitions

###### Preferred 领导者选举

####### 为了避免部分Broker负载过重而提供的一种换Leader方案

###### 集群成员管理

####### 新增Broker、Broker主动关闭、Broker宕机

####### 控制器watch "zk /brokers/ids 下的临时节点" 变化

###### 数据服务

####### 向其他 Broker 提供元数据信息

######## 元数据

######### 主题信息：分区、Leader Replica、ISR

######### Broker信息：运行中、关闭中

####### 是 zk 数据的缓存？

##### 选举 / 故障转移

###### 抢占！

每个Broker启动后，都尝试创建 /controller 临时节点。

没抢到的，会Watch这个节点

####### zk /controller节点创建

###### 技巧

####### 当控制器出问题时，可手工删除节点 触发重选举

#### 协调者 GroupCoordinator

##### 作用

###### 为consumer group 执行 Rebalance

####### 协调者对应一个 Group！

###### 位移管理

####### 消费者提交位移时，是向Coordinator所在的Broker提交

###### 组成员管理

####### 消费者启动时，向Coordinator所在Broker发送请求，由Coordinator进行消费者组的注册、成员管理

##### 消费者如何找到Coordinator？

###### 找到由位移主题的哪个分区来保存该Group数据

partitionId = Math.abs(groupId.hashCode % offsetsTopicPartitionCoun)

###### 找到该分区的Leader副本所在的Broker

### 生产者

#### 发送流程

##### 准备

###### 0. 创建 KafkaProducer

###### 1. 创建 ProducerRecord

####### 必选：Topic / Value

####### 可选：Key / Partition

##### send()

###### 2. Seriazlier

####### 将 key/value 序列化为字节数组

####### 实现方式

######## 自定义：继承 Serializer

######### 不推荐：兼容性

######## 使用已有的：JSON、Avro、Thrift、Protobuf

######### 引入 Schema Registry

######### schema.registry.url

########## Q: 如何关联到 Schema ID / version ??

####### 注意

######## 不是序列化整个 ProducerRecord对象？

######### NO，序列化的是 value“对象”

###### 3. Partitioner

####### 分区机制

######## 如果已指定 Partition：不做处理

######## 如果已指定 Key: 按Key分区

```
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return Math.abs(key.hashCode()) % partitions.size();
```

######### 使用Kafka自己特定的散列算法

######### 如果增加 Partition，相同key可能会分到另一个Partition

########## 建议提前规划，不要增加分区。。。

######## 自定义分区器

######### 实现Partitioner接口

int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);


######### 场景：某个key数据量超大，需要写入专门的分区

######## key == null && 没有分区器，则 Round-robin 平衡

###### 4. 记录到batch

####### 同一个batch的记录，会发送到相同的Topic / Partition

###### 5. 单独线程 发送batch 到相应的 Broker

Sender线程：
new KafkaProducer时会创建“并启动”Sender线程，该线程在开始运行时会创建与bootstrap.servers的连接

####### 三种发送方式

######## Fire-and-forget

######### send(record)

######## 同步发送

######### send(record).get()

########## 获取RecordMetadata

######## 异步发送

######### send(record, Callback)

##### response

###### 6. Broker 回复 Response

####### 成功

######## 返回 RecordMetadata，包含topic / partition / offset

####### 失败

######## 返回 Error，生产者决定是否重试

###### 7. close

####### 可用 try-with-resource

#### 消息

##### Batch 批量发送

###### 权衡吞吐量 vs. 延时

##### 元数据

###### key

####### 可选，消息分区的依据

###### offset

####### 分区内唯一

##### Schemas

###### JSON, XML

###### Avro

#### 拦截器

##### interceptor.classes

```
Properties props = new Properties();

List<String> interceptors = new ArrayList<>();
interceptors.add("com.yourcompany.kafkaproject.interceptors.AddTimestampInterceptor"); // 拦截器1
interceptors.add("com.yourcompany.kafkaproject.interceptors.UpdateCounterInterceptor"); // 拦截器2

props.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, interceptors);
```

##### 实现接口 ProducerInterceptor

###### onSend()

####### 发送之前被调用

###### onAcknowlegement()

####### 早于callback调用

####### 与onSend()是不同线程，注意共享变量的同步！

##### 应用场景

###### 监控、审计

###### 一个消息从生产到消费的总延时

### 消费者

#### 消费者组

##### 定义

###### 组内所有消费者协调在一起来消费订阅主题的所有分区

###### 每个分区只能由组内的一个Consume实例消费

####### 增加组内消费者：scale up

##### 模型

###### 消息队列模型

####### 所有实例都属于同一个group

###### 发布订阅模型

####### 所有实例分别属于不同group

##### 组员个数

###### = 分区数

####### 每个实例消费一个分区

###### > 分区数

####### 有实例空闲

###### < 分区数

####### 有实例消费多个分区

##### 要点

###### 每个Consumer属于一个Consumer Group

###### 每个Consumer独占一个或多个Partition

###### 每个Consumer Group 都有一个 Coordinator 

####### 负责分配 消费者与分区的对应关系

####### 当分区或者消费者发生变更，则触发 rebalance

###### Consumer 维护与 Coordinator 之间的心跳，以便让Coordinator 感受到Consumer的状态

#### 消费流程

##### 准备

###### 0. 创建 kafkaConsumer

###### 1. subscribe(topics)

####### 参数

######## 参数：Topic 列表

######## 参数可以是正则表达式

######### 同时注册多个主题，如果有新主题匹配，会触发 Rebalance

####### 或者作为 standalone 消费者

######## consumer.assign(partitions)

##### 轮询

###### 2. poll()

####### 第一次 poll 时

######## 找到 GroupCoordinator

######## 加入 Consumer Group

######## 收到 Partition 指派

####### 返回 ConsumerRecords

######## 是 ConsuemrRecord的集合

######## 包含 topic / partition / offset / key / value

####### Deserializer

######## 不推荐自定义

######## Avro

######### 引入 Schema Registry

######### Q: 如何关联到 Schema ID / version ??

##### 关闭

###### 3. consuemr.wakeup()

####### 从另一个线程调用，例如 addShutdownHook() 中调用

####### 然后 poll() 会抛出 WakeupException

######## 该异常无需处理

###### 4. commit / close

####### 会马上触发 Rebalance，而无需等待 GroupCoordinator  被动发现

####### 在 finally 中执行

#### 注意

##### 编码

###### 消费者组

####### KafkaConsumer.subscribe()

###### 独立消费者

####### KafkaConsumer.assign()

###### 拦截器

####### interceptor.classes

####### 实现接口 ConsumerrInterceptor

######## onConsume()

######### 消费之前被调用

######## onCommit()

######### 提交位移之后执行

##### 多线程消费

###### 粗粒度：每个线程启一个KafkaConsumer

```java
public class KafkaConsumerRunner implements Runnable {

private final AtomicBoolean closed = new AtomicBoolean(false);

private final KafkaConsumer consumer;


public void run() {
  try {
  consumer.subscribe(Arrays.asList("topic"));
  
  while (!closed.get()) {
	ConsumerRecords records = 
consumer.poll(Duration.ofMillis(10000));
     //  执行消息处理逻辑
   }
   } catch (WakeupException e) {
     // Ignore exception if closing
     if (!closed.get()) throw e;
   } finally {
      consumer.close();
   }
}


// Shutdown hook which can be called from a separate thread
public void shutdown() {
   closed.set(true);
   consumer.wakeup();
}

````

####### 优点

######## 实现简单

######## 线程之间独立，没有交互

######## 可保证分区内消息的顺序性

####### 缺点

######## 占用资源：内存、TCP连接

######## 线程数受限：不能多于分区数

######## 无法解决消息处理慢的情况

###### 细粒度：多线程执行消息处理逻辑

https://www.cnblogs.com/huxi2b/p/7089854.html

```java
private final KafkaConsumer<String, String> consumer;
private ExecutorService executors;

executors = new ThreadPoolExecutor(
	workerNum, workerNum, 0L, TimeUnit.MILLISECONDS,
	new ArrayBlockingQueue<>(1000), 
	new ThreadPoolExecutor.CallerRunsPolicy());


...
while (true)  {
  ConsumerRecords<String, String> records = 
  consumer.poll(Duration.ofSeconds(1));
  
  for (ConsumerRecord record : records) {
	executors.submit(new Worker(record));
  }
}


```

####### 优点

######## 消息获取、消息消费独立

######## 伸缩性好

####### 缺点

######## 实现难度大：有两组线程

######## 无法保证分区内消息的顺序性

######## 较难正确提交正确的位移

######### 因为消息消费链路被拉长

######### 可能导致消息重复消费

### zookeeper

#### 存储信息

https://www.jianshu.com/p/da62e853c1ea

##### broker

###### /brokers/ids/0

Schema:
{
"jmx_port": jmx端口号,
"timestamp": kafka broker初始启动时的时间戳,
"host": 主机名或ip地址,
"version": 版本编号默认为1,
"port": kafka broker的服务端端口号,由server.properties中参数port确定
}

Example:
{
"jmx_port": 6061,
"timestamp":"1403061899859"
"version": 1,
"host": "192.168.1.148",
"port": 9092
}

####### 临时节点

######## 如果 broker 宕机，节点删除

######### 新启动的Broker如果有相同ID，则无缝继承原Broker的partition和topic

######## 对应每个在线的Broker

######### 包含：地址、版本号、启动时间

###### /brokers/topics/x/partitions/0/state

Schema:
{
  "controller_epoch": 表示kafka集群中的中央控制器选举次数,
  "leader": 表示该partition选举leader的brokerId,
  "version": 版本编号默认为1,
  "leader_epoch": 该partition leader选举次数,
  "isr": [同步副本组brokerId列表]
}

Example:
{
  "controller_epoch": 1,
  "leader": 2,
  "version": 1,
  "leader_epoch": 0,
  "isr": [2, 1]
}

####### 保存分区信息

######## 分区的Leader

######## 所有ISR的BrokerID

####### 如果该分区的 leader broker 宕机，节点删除

##### consumer

###### /consumers/consumer-group/ids/consumer_id

Schema:
{
"version": 版本编号默认为1,
"subscription": { //订阅topic列表
"topic名称": consumer中topic消费者线程数
},
"pattern": "static",
"timestamp": "consumer启动时的时间戳"
}

Example:
{
"version": 1,
"subscription": {
"open_platform_opt_push_plus1": 5
},
"pattern": "static",
"timestamp": "1411294187842"
}

###### /consumers/consumer-group/offsets/topic-x/p-x

####### 问题

######## offset 写入频繁，并不适合 zk

######## 所以新版本 offset 保存在 内部 Topic 中

######### __consumer_offsets

###### /consumer/consumer-group/owners/topic-x/p-x

标识被哪个consumer消费


##### controller

##### config

#### 用例

##### 客户端如何找到对应Broker地址

###### 先从 /state 中找到分区对应的brokerID

###### 再从 /brokers/ids 中找到对应地址

###### 注意

####### 客户端并不直接和 zk 交互，而是和broker交互

####### 每个broker 都维护了和 zk 数据一样的元数据缓存，通过Watcher 机制更新

## 位移

### 概念

#### 目的

##### 重平衡后，消费者得以从最新的已提交offset处开始读取

##### 当已提交offset < 当前消费者已处理消息：重复消费

##### 当已提交offset > 当前消费者已处理消息：lost

###### 但实际lost部分的消息肯定已被其他消费者处理过，所以没问题

#### 位移存储

##### 老版本：zk

###### 用zookeeper存储位移

###### 好处：broker无状态，方便扩展

###### 坏处：zk不适合频繁写入

##### 新版本：位移主题

######  __consumer_offsets

###### 消息结构

####### K

######## groupId + topic + partition

######### 分区粒度

######### groupId ==> 消费者信息

####### V

######## 位移

######## 时间戳

######### 为了删除过期位移消息

######## 用户自定义数据

###### tombstone消息

####### 墓碑消息，delete mark

######## 表示要彻底删除这个group信息

####### 当consumer group下所有实例都停止，并且位移数据都被删除时，会写入该消息

###### 创建

####### 第一个consumer启动时，自动创建位移主题

####### offset.topic.num.partitions=50

######## 默认分区50

####### offset.topic.replication.factor=3

######## 默认副本3

### 编码

#### 位移提交

##### 自动提交

###### 原理

####### 每隔 N 秒自动提交一次位移

######## Q: how? 单独线程计时？

####### 开始调用poll()时，提交上次poll 返回的所有消息

######## Q: 和 auto.commit.interval.ms有关系吗？

自动提交逻辑是在poll方法中，如果间隔大于最小提交间隔，就会运行逻辑进行offset提交，如果小于最小间隔，则忽略offset提交逻辑？也就是说上次poll 的数据即便处理结束，没有调用下一次poll，那么offset也不会提交？

###### 配置

####### enable.auto.commit=true

####### auto.commit.interval.ms

######## 提交间隔

######### 默认 5s

######## vs. poll 间隔？

######### interval.ms 表示最小间隔，实际提交间隔可能大于该值

###### 缺点

####### consumer不关闭 就会一直写入位移消息

######## 导致位移主题越来越大

######## 需要自动整理消息

######### Log Cleaner 后台线程

########## Compact 整理策略

########## 扫描所有消息，删除过期消息

####### 可能导致重复消费

######## 在提交之前发生 Rebalance

##### 手动提交

###### consumer.commitSync()

```
while (true) {
  ConsumerRecords<String, String> records =
  consumer.poll(Duration.ofSeconds(1));
  
  process(records); // 处理消息
  
  try {
     consumer.commitSync();
   } catch (CommitFailedException e) {
   handle(e); // 处理提交失败异常
   }
}

```

####### commitSync()时会阻塞

####### 自动重试

###### consumer.commitAsync()

```java
while (true) {
  ConsumerRecords<String, String> records = 
  consumer.poll(Duration.ofSeconds(1));
  
  process(records); // 处理消息
  
  consumer.commitAsync((offsets, exception) -> {
	if (exception != null)
	  handle(exception);
	});
}

```

####### 回调

####### 不会重试

######## 也不要在回调中尝试重试

######### 如果重试，则提交的位移值可能早已过期

######### 除非：设置单调递增的ID，回调中检查ID判断是否可重试

###### 组合 同步+非同步

```java
try {
  while (true) {
    ConsumerRecords<String, String> records = 
consumer.poll(Duration.ofSeconds(1));
  // 处理消息
  process(records); 

  // 使用异步提交规避阻塞
  commitAysnc(); 
  }
} catch (Exception e) {
   handle(e); 
} finally {
   try {
     // 最后一次提交使用同步阻塞式提交
     consumer.commitSync();     
	} finally {
	 consumer.close();
    }
}

```

####### 轮询中：commitAsync()

######## 避免阻塞

####### 消费者关闭前：commitSync()

######## 确保关闭前保存正确的位移

######## 因为这是rebalance前的最后一次提交机会

###### 精细化提交

问题：
如果一次poll过来5000条消息，默认要全部消费完后一次提交

####### 目的：每消费xx条数据即提交一次位移，增加提交频率

####### commitAsync(Map<TopicPartition, OffsetAndMetadata>)

```
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
int count = 0;
……
while (true) {

  ConsumerRecords<String, String> records = 
  consumer.poll(Duration.ofSeconds(1));

  for (ConsumerRecord<String, String> record: records) {
   // 处理消息
   process(record);  

   // 记录已处理的 offsets + 
   offsets.put(
   new TopicPartition(record.topic(), record.partition()),
   new OffsetAndMetadata(record.offset() + 1);
  
  if（count % 100 == 0）
    // 提交已记录的offset
    consumer.commitAsync(offsets, null); // 回调处理逻辑是 null
    count++;
    }
}

```

######## 为什么+1：位移表示要处理的“下一条消息”

###### 注意

####### 提交之前确保该批消息已被处理结束，否则会丢失消息

##### 问题

###### 消费者重启后，如何获取offset？

####### 去 位移主题 里查询？

####### Coordinator 会缓存

#### CommitFailedException

##### 含义

###### 提交位移时发生不可恢复的错误

###### 对于可恢复异常

####### 很多api都是支持自动错误重试

####### 例如commitSync()

##### 原因

###### 消费者组开启rebalance，并将要提交位移的分区分配给了另一个消费者

当超过max.poll.interval.ms配置的时间Kafka server认为kafka consumer掉线了，于是就执行分区再均衡将这个consumer踢出消费者组。但是consumer又不知道服务端把自己给踢出了，下次在执行poll()拉取消息的时候（在poll()拉取消息之前有个自动提交offset的操作），就会触发该问题。 

###### 深层原因：连续两次调用poll的间隔 超过了max.poll.interval.ms

####### 因为触发了重平衡？

###### 冷门原因：Standalone 消费者的groupId与其他消费者组重复

##### 规避

###### 缩短单条消息处理的时间

###### 增大消费一批消息的最大时长

####### 增大 max.poll.interval.ms

###### 减少poll方法一次性返回的消息数量

####### 减小 max.poll.records

###### 多线程消费

#### 消费者组位移管理

##### 查看消费者组位移

查看消费者组提交的位移

kafka-console-consumer.sh 
  --bootstrap-server host:port 
  --topic __consumer_offsets 
  --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" 
  --from-beginning


读取位移主题消息

kafka-console-consumer.sh 
  --bootstrap-server host:port 
  --topic __consumer_offsets 
  --formatter "kafka.coordinator.group.GroupMetadataManager\$GroupMetadataMessageFormatter" 
  --from-beginning


###### kafka-console-consumer.sh 

##### 重设消费者组位移

###### 重设策略

####### 位移维度

######## Earliest

重新消费所有消息


######## Latest

从最新消息处开始消费


######## Current

重设到当前最新提交位移处。

场景：修改了消费者代码，并重启消费者后。

consumer.partitionsFor(topic).stream().map(info -> 
	new TopicPartition(topic, info.partition()))
	.forEach(tp -> {
	long committedOffset = consumer.committed(tp).offset();
    
	consumer.seek(tp, committedOffset);
});


######## Specified-Offset

手动跳过错误消息的处理。

long targetOffset = 1234L;

for (PartitionInfo info : consumer.partitionsFor(topic)) {
	TopicPartition tp = new TopicPartition(topic, info.partition());
	consumer.seek(tp, targetOffset);
}


######## Shift-By-N



for (PartitionInfo info : consumer.partitionsFor(topic)) {
  TopicPartition tp = new TopicPartition(topic, info.partition());

  long targetOffset = consumer.committed(tp).offset() + 123L; 

  consumer.seek(tp, targetOffset);
}


####### 时间维度

######## DateTime

绝对时间: consumer.offsetsForTimes

long ts = LocalDateTime.of(
	2019, 6, 20, 20, 0).toInstant(ZoneOffset.ofHours(8)).toEpochMilli();
    
Map<TopicPartition, Long> timeToSearch = 
 consumer.partitionsFor(topic).stream().map(info -> 
	new TopicPartition(topic, info.partition()))
.collect(Collectors.toMap(Function.identity(), tp -> ts));

for (Map.Entry<TopicPartition, OffsetAndTimestamp> entry : 
  consumer.offsetsForTimes(timeToSearch).entrySet()) {

consumer.seek(entry.getKey(), entry.getValue().offset());
}


######## Duration

相对时间 

Map<TopicPartition, Long> timeToSearch = consumer.partitionsFor(topic).stream()
 .map(info -> new TopicPartition(topic, info.partition()))
 .collect(Collectors.toMap(Function.identity(), tp -> System.currentTimeMillis() - 30 * 1000  * 60));

for (Map.Entry<TopicPartition, OffsetAndTimestamp> entry : 
         consumer.offsetsForTimes(timeToSearch).entrySet()) {
         consumer.seek(entry.getKey(), entry.getValue().offset());
}


###### 重设方式

####### kafka-consumer-groups.sh

bin/kafka-consumer-groups.sh 
  --bootstrap-server host:port 
  --group test-group 
  --reset-offsets 
  --all-topics 

  --to-earliest 
  --to-latest
  --to-current
  --to-offset xx
  --shift-by XX
  --to-datetime 2019-08-11T20:00:00.000
  --by-duration PT0H30M0S

  –execute


####### API

```java

class HandleRebalance implements ConsumerRebalanceListener {

  public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
  // Assigned: 找到偏移量
  for (TopicPartition p : partitions) 
    consumer.seek(p, getOffsetFromDB(p));
  
  }
  
  public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
  // Revoked: 提交偏移量
    commitDbTrx();
  }

}

try {
  // subscribe 时传入监听器
  consumer.subscribe(topics, new HandleRebalance());
  consumer.poll(0);
  
  for (TopicPartition p : consumer.assignement()) {
    consumer.seek(p, getOffsetFromDB(p));
  }
  
  while (true) {
    // 轮询
    ConsumerRecords<String, String> records =  consumer.poll(100);
  for (ConsumerRecord record : records) {
    // 同一事务里：存储记录、offset
     storeRecordInDB();
     storeOffsetInDB();
  }
  commitDbTrx();
  

  // 使用异步提交规避阻塞
  commitAysnc(currOffsets, null);
  }
} catch (WakeupException e) {
  // ignore.
} catch (Exception e) {
   handle(e); 
} finally {
   try {
     // 最后一次提交使用同步阻塞式提交
     consumer.commitSync(currOffsets);     
	} finally {
	 consumer.close();
    }
}

```


######## seek()

```java

void seek(TopicPartition partition, long offset);

void seek(TopicPartition partition, OffsetAndMetadata offsetAndMetadata);

```

######### 调用时机

########## 1. 消费者启动时

########### subscribe / poll 之后，通过 consumer.assignment() 获取分配到的分区，对每个分区执行 consumer.seek(partition, offset) 

########### 其中offset 自己管理，从存储中读取

########## 2. onPartitionAssigned()

########### 对每个新分配的 partition，执行consumer.seek(partition, offset)

########### 其中offset 自己管理，从存储中读取

######## seekToBeginning() / seekToEnd()

```java

void seekToBeginning(Collection<TopicPartition> partitions);
  
void seekToEnd(Collection<TopicPartition> partitions);

```

## rebalance

### 触发条件

#### 消费者组成员数目变更

##### 增加、离开、崩溃

##### 避免不必要的重平衡：被协调者错误地认为消费者已离开

#### 主题数变更

##### 例如订阅模式 consumer.subsribe(Pattern.compile("t.*c"))

##### 当创建的新主题满足此模式，则发生rebalance

#### 分区数变更

##### 增加

### 避免不必要的rebalance

#### rebalance的问题

##### rebalance过程中，所有consumer停止消费

###### STW

###### 消费者部署过程中如何处理？

####### 发布系统：只有在 switch 的时候才启动消费者？

##### rebalance是所有消费者实例共同参与，全部重新分配；而不是最小变动

###### 未考虑局部性原理

##### rebalance过程很慢

#### 什么是不必要的重平衡？

##### 被 coordinator “错误地”认为已停止

##### 组员减少导致的rebalance

#### 手段

##### 避免“未及时发心跳”而导致消费者被踢出

###### session.timeout.ms = 6s

####### 存活性时间间隔

####### 越小则越容易触发重平衡

###### heartbeat.interval.ms = 2s

####### 心跳间隔，用于控制重平衡通知的频率

####### 保证在timeout判定前，至少发送3轮心跳

##### 避免“消费时间过长”而导致被踢出

###### max.poll.interval.ms

####### 两次poll() 的最大间隔

######## 如果超过，则consumer会发起“离开组”请求

####### 如果消费比较耗时，应该设大

######## 否则会被Coordinator剔除出组

###### Worker Thread Pool 异步并行处理

####### 注意不能阻塞poll，否则心跳无法上报

####### 异步处理开始后，pause() 使得继续 poll 但不返回新数据；等异步处理结束，再 resume()

##### GC调优

###### full GC导致长时间停顿，会引发rebalance

### 重平衡监听器

#### ConsumerRebalanceListener

##### 作用

###### 当将要失去分区所有权时，处理未完成的事情（例如提交 offset）

###### 当被分配到一个新分区时，seek到指定的offset处

##### 代码

```java
Map<TopicPartition, OffsetAndMetadata> currOffsets = new HashMap();

class HandleRebalance implements ConsumerRebalanceListener {
  public void onPartitionsAssigned(Collection<TopicPartition> partitions) {}
  
  public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
  // Revoked: 提交偏移量
    consumer.commitSync(currOffsets);
  }

}

try {
  // subscribe 时传入监听器
  consumer.subscribe(topics, new HandleRebalance());
  
  while (true) {
    // 轮询
    ConsumerRecords<String, String> records =  consumer.poll(100);
  for (ConsumerRecord record : records) {
     process(record); 
     // 记录当前处理过的偏移量
     currOffsets.put();
  }
 
  

  // 使用异步提交规避阻塞
  commitAysnc(currOffsets, null);
  }
} catch (WakeupException e) {
  // ignore.
} catch (Exception e) {
   handle(e); 
} finally {
   try {
     // 最后一次提交使用同步阻塞式提交
     consumer.commitSync(currOffsets);     
	} finally {
	 consumer.close();
    }
}

```

###### onPartitionsAssigned(partitions)

####### 触发时间：消费者停止消费之后、rebalance开始之前

####### 常见操作：清理状态、seek()

###### onPartitionsRevoked(partitions)

####### 触发时间：rebalance 之后、消费者开始消费之前

####### 常见操作：提交offset

######## 要用 commitSync()，确保在rebalance开始之前提交完成

###### subscribe(topics, listener)

### rebalance流程

如何通知消费者进行重平衡？

#### 原理

##### 心跳线程

###### Consumer 定期发送心跳请求到Broker

####### Broker通过心跳线程通知其他消费者进行重平衡

###### heartbeat.interval.ms

####### 控制重平衡通知的频率

##### Group Coordinator

###### 当分组协调者决定开启新一轮重平衡，则在“心跳响应”中加入 REBALANCE_IN_PROGRESS

#### 消费者组状态流转

##### 状态

###### Empty

####### 组内没有任何成员，但可能存在已提交的位移

###### Dead

####### 组内没有任何成员，且组的元数据信息已在协调者端移除

###### PreparingRebalance

####### 消费者组准备开启重平衡

###### CompletingRebalance

####### 消费者组所有成员已加入，正在等待分配方案

######## = AwaitingSync ?

###### Stable

####### 分配完成

##### 流转

###### 初始化

####### Empty --> PreparingRebalance --> CompletingRebalance --> Stable

###### 成员加入/退出

####### Stable --> PreparingRebalance

####### 所有成员重新申请加入组

###### 所有成员退出

####### Stable --> PreparingRebalance --> Empty

####### kafka定期自动删除empty状态的过期位移

#### 流程

##### Consumer 端重平衡流程

###### JoinGroup请求

####### 加入组时，向分组协调者发送JoinGroup请求

######## 上报自己订阅的主题

####### 选出Leader Consumer

######## 协调者收集到全部成员的JoinGroup请求后，"选择"一个作为领导者

######### == 第一个加入组的消费者？

######## 协调者将订阅信息放入JoinGroup响应中，发给Leader Consumer

######## Leader Consumer负责收集所有成员的订阅信息，据此制定具体的分区消费分配方案

######### PartitionAssignor

######## 最后 Leader Consumer 发送 SyncGroup 请求

####### Q

######## 为什么引入 Leader Consumer？可否协调者来分配？

######### 客户端自己确定分配方案有很多好处。比如可以独立演进和上线，不依赖于服务器端

###### SyncGroup请求

####### Leader Consumer 将分配方案通过SyncGroup请求发给协调者

######## 此时其他消费者也会发送空的SyncGroup请求

####### 协调者将分配方案放入SyncGroup响应中，下发给所有成员

######## 通过协调者中转

####### 消费者进入Stable状态

##### Broker 端重平衡场景

###### 新成员入组

####### JoinGroup

####### 回复“心跳请求”响应给所有成员，强制开启新一轮重平衡

###### 组成员主动离组: close()

####### LeaveGroup

####### 回复“心跳请求”响应给所有成员，强制开启新一轮重平衡

###### 组成员崩溃离组

####### session.timeout.ms 后，Broker感知有成员超时

######## 在此期间，老组员对应的分区消息不会被消费

####### 回复“心跳请求”响应给所有成员时，强制开启新一轮重平衡

## 原理

### 请求处理

https://time.geekbang.org/column/article/110482

#### 种类

##### 数据类型请求

###### PRODUCE

####### 来自生产者

####### 必须发往 leader replica

######## 否则会报错 “Not a Leader for Partition”

######## 如果ack=all，leader 会将其暂存在 Purgatory，等到 ISR 复制完该消息

###### FETCH

####### 来自消费者、Follower Replica

####### 必须发往 leader replica

######## 否则会报错 “Not a Leader for Partition”

######## 返回response使用 zero-copy !

###### METADATA request

####### 可以发往任意 broker

####### 客户端据此才能知道 那个是Leader Replica，才能正确发送PRODUCE / FETCH 请求

##### 控制类请求

###### LeaderAndIsr

####### Controller --> Replicas

######## 通知新的Leader，开始接收客户端请求

######## 通知其他Follower，向leader复制消息

###### StopReplica

###### OffsetCommitRequest

###### OffsetFetchRequest

#### 原理

##### Reactor 模式

##### 隔离 控制类请求 vs. 数据类请求

###### 两套组件：网络线程池、IO线程池 都有两套

#### 组件

##### SocketServer

###### 接收请求

##### Acceptor 线程

###### 请求分发

###### 轮询，将入站请求公平地分发到所有网络线程

##### Processor Thread 网络线程池

###### num.network.threads = 3

###### 负责与客户端通过网络读写数据

####### 网络线程拿到请求后，并不自己处理，而是放入一个共享请求队列（Request Queue）中

####### 最后从 Response Queue 中找到response并返回给客户端

##### Request Queue 共享请求队列

###### 所有网络线程共享

##### IO 线程池

###### num.io.threads = 8

###### 从 Request Queue 共享请求队列中取出请求，进行处理

####### 包括写入磁盘、读取页缓存等

##### Response Queue 请求响应队列

###### 每个网络线程专属，不共享

###### 因为没必要共享了！！！

##### Purgatory 炼狱

###### 用来缓存延时请求

###### 例如acks=all时

acks=all, 需要所有ISR副本都接收消息后才能返回。处理该请求的IO线程就必须等待其他Broker的写入结果。

当请求不能立刻处理时，就会暂存在Purgatory中。

等条件满足，IO线程会继续处理该请求，将Response放入对应网络线程的响应队列中

#### TCP 连接管理

##### 生产者TCP连接管理

###### TCP

####### 多路复用请求

将多个数据流合并到一条物理连接。

####### 同时轮询多个连接

###### 建立连接

####### 时机

######## 创建Producer时与与bootstrap.servers建链

Sender线程：
new KafkaProducer时会创建“并启动”Sender线程，该线程在开始运行时会创建与bootstrap.servers的连接

######## 可选

######### 更新元数据后，如果发现与某些Broker没有连接，则建链

更新元数据的时机：
1. 给不存在的主题发送消息时，Broker返回主题不存在，Producer会发送METADATA;
2. metadata.max.age.ms定期更新元数据；

########## 会连接所有 Broker

########## 浪费！

######### 发送消息时，会连接到目标 Broker

####### 流程

######## 1. 创建KafkaProducer实例

######## 2. 后台启动Sender线程，与Broker建立连接：连接到bootstrap.servers

######## 3. 向某一台broker发送METADATA请求，获取集群信息

###### 关闭连接

####### 主动关闭

######## producer.close()

####### 自动清理

######## connections.max.idle.ms

######### broker端发起

###### 更新集群元数据

####### 当给一个不存在的主题发消息：回复主题不存在，producer会发送metadata请求刷新元数据

####### metadata.max.age.ms到期

##### 消费者TCP连接管理

###### 创建连接

####### 执行 poll() 时建链，三个时机

######## 首次执行poll()，发送 FindCoordinator 请求时

######### 目的：询问Broker谁是当前消费者的协调者；

######### 策略：向集群中当前负载最小的Broker发送请求

######## 连接协调者，执行组成员管理操作时

只有连接协调者，才能：

加入组、等待组分配方案、心跳请求处理、位移获取、位移提交

######### 只有连接协调者，才能处理加入组、分配、心跳、位移提交等

######## 消费数据时

######### 与分区领导者副本所在Broker建立连接

####### 注意：new KafkaConsumer()时并不创建连接

###### 关闭连接

####### 主动关闭

######## KafkaConsumer.close()

######## kill

####### 自动关闭

######## connection.max.idel.ms 到期

######## 默认9分钟

### 存储

#### Partition 分配

##### 选择 Broker

###### Round-Robin

####### 先随机选一个 broker-1作为 partition-1 leader；依次选 broker-2 作为 partition-2 leader

####### 然后针对每个分区，从leader broker开始往后设置 follower

######## 例如partition-2 : leader = broker-2, follower1 = broker-3, ...

###### 配置 broker.rack

####### 更高的可用性

##### 选择 目录

###### 使用 已有partition个数最少的那个目录

####### 考虑个数，而不考虑大小！

#### 文件管理

##### Segment

###### log.segment.ms | bytes

###### 一个segment对应一个数据文件

####### 当达到segment限制时，会关闭当前文件，并创建一个新的

###### active segment

####### 当前正在写入的segment，不会被删除

####### 可能导致 log.retention.ms 不生效：超出很多后才能被过期

##### 文件格式

###### DumpLogSegment 工具：查看segment文件内容

###### 对于压缩过的消息，Broker不会解压，而是直接存储为 Wrapper Message

#### Indexes

##### 将 offset 映射到 segment 文件 + 文件内的位置

### 高性能

#### 原理

##### 批量消息

批消息构建：生产端
批消息解开：消费端

Broker处理的永远是批消息

##### 压缩

##### 顺序读写

##### PageCache

##### 零拷贝 ZeroCopy

AS-IS
PageCache 
--> 内存 
--> Socket缓冲区

TO-BE
PageCache 
--> Socket缓冲区



AS-IS
--> 磁盘 
--> 内核缓冲区 
--> 用户缓冲区
--> Socket缓冲区
--> 网卡缓冲区

TO-BE
--> 磁盘 
--> 内核缓冲区 
--> Socket缓冲区
--> 网卡缓冲区


###### 发生在传输给Consumer时

###### 操作系统 sendfile 函数

####### JavaNIO: FileChannel.transferTo()

#### 调优

##### 链接调优

###### 复用 Producer / Consumer 对象

###### 及时关闭 Socket连接、ByteBuffer缓冲区

##### 吞吐量调优

###### Producer端

####### batch.zise 增大

####### linger.ms 增大

######## 批次缓存时间

####### compress.type=lz4 / zstd

####### acks=0/1, 不要是all

####### retries=0

###### Broker端

####### num.replica.fetchers 增大

######## Follower副本用多少线程来拉取消息

###### Consumer端

####### 客户端fetch.min.bytes

Broker 积攒了N字节数据，就可以返回给consumer


####### 多线程方案

###### 场景

####### 允许延时、允许丢消息

##### 延时调优

###### Producer端

希望消息尽快发出，不要停留

####### linger.ms=0

####### compress.type=none

####### acks=1

###### Broker端

####### num.replica.fetchers 增大

###### Consumer端

####### fetch.min.bytes=1

######## 只要Broker有数据就立即返回给 Consumer

## 设计思路

### 事务消息

#### RocketMQ 事务消息

##### 作用

###### 解决本地事务和发消息的数据一致性问题

##### 例子

###### 下单后清理购物车

###### 购物车模块：清理后再提交确认

###### 订单模块：创建订单+发送消息=事务

##### 流程

###### 1. Producer --> Broker: 开启事务

###### 2. Producer --> Broker: 发送半消息

半消息：消费者看不到

RocketMQ: 添加属性 PROPERTY_TRANSACTION_PREPARED=true

###### 3. Producer: 执行本地事务

###### 4. Producer --> Broker: 提交或回滚

####### 若失败，则抛异常；业务代码可重试

####### RocketMQ则实现了事务反查

需要Produer提供反查接口，Broker定期去查询本地事务状态，根据结果决定提交或回滚。

作用：Producer发送提交或回滚失败，Broker一直没收到后续命令，兜底策略

###### 5. Broker: 投递消息

#### Kafka 事务消息

##### 作用

###### 实现 Exactly Once 机制：读数据 + 计算 + 保存结果过程中数据“不重不丢”

###### Topic A --> 流计算 --> Topic B 过程中每个消息都被计算一次

##### 实现

为了实现事务，也就是保证一组消息可以原子性生产和消费，Kafka引入了如下概念；

- 引入了 `事务协调者（Transaction Coordinator）`的概念。与消费者的组协调者类似，每个生产者会有对应的事务协调者，赋予PID和管理事务的逻辑都由事务协调者来完成。

- 引入了 `事务日志（Transaction Log）` 的内部主题。与消费者位移主题类似，事务日志是每个事务的持久化多副本存储。事务协调者使用事务日志来保存当前活跃事务的最新状态快照。

- 引入了`控制消息（Control Message）` 的概念。这些消息是客户端产生的并写入到主题的特殊消息，但对于使用者来说不可见。它们是用来让broker告知消费者之前拉取的消息是否被原子性提交。控制消息之前在这里被提到过。

- 引入了 `TransactionalId` 的概念，TransactionalId可以让使用者唯一标识一个生产者。一个生产者被设置了相同的TransactionalId的话，那么该生产者的不同实例会恢复或回滚之前实例的未完成事务。

- 引入了 `生产者epoch` 的概念。生产者epoch可以保证对于一个指定的TransactionalId只会有一个合法的生产者实例，从而保证事务性即便出现故障的情况下。

##### 非半消息机制？！

### 可靠性交付

#### 最多一次 at most once

##### 需要Producer禁止重试

#### 至少一次 at least once

##### 默认提供

##### 何时会多于一次？

###### Broker返回应答时 网络抖动，Producer此时选择重试

##### 可能消息重复

###### At Least Once + 幂等消费 = Exactly Once

###### 幂等性生产

####### 每条消息赋予唯一ID

####### 服务端收到消息，对比存储的最后一条ID，如果一样，则认为是重复消息丢弃

###### 幂等性消费

####### 利用数据库的唯一约束实现幂等

// 判断 ID 是否存在
boolean isIDExisted = selectByID(ID); 
if(isIDExisted) {
  // 存在则直接返回
  return; 
} else {
  // 不存在，则处理消息
  process(message); 
  // 存储 ID
  saveID(ID);
}

######## 或者Redis SETNX

####### 为更新的数据设置前置条件

######## version

####### 记录并检查操作

######## Token / GUID

#### 精确一次 exactly once

http://www.dengshenyu.com/kafka-exactly-once-transaction-interface/ 

##### 生产者 -幂等性 Idempotence

###### 实现

####### props.put("enable.idempotence", true)

###### 原理

####### Broker此时会多保存一些字段，用于判断消息是否重复，自动去重

####### Kafka 发送时自动去重

###### 限制

####### 无法实现跨分区的幂等

####### 无法实现跨会话的幂等

######## Producer重启后会丧失幂等性

##### 生产者 -事务 Transaction

###### 实现

####### props.put("enable.idempotence", true)

####### 同时设置 transactional.id

######## 如果配置了transaction.id，则此时enable.idempotence会被设置为true

######## 在使用相同TransactionalId的情况下，老的事务必须完成才能开启新的事务

####### 代码中显式地提交事务

```java
producer.initTransactions();

try {
  producer.beginTransaction();
  producer.send(record1);
  producer.send(record2);
  producer.commitTransaction();
  
} catch (KafkaException e) {
  producer.abortTransaction();
}

```

###### consumer改动

####### 设置isolation.level

######## read_uncommitted

######## read_committed

只读取成功提交了的事务

######### 如果不设置，消费者会读到提交失败的消息

###### 原理

####### 保证消息原子性地写入到多个分区，要么全部成功，要么全部失败

######## 但即便失败，也会写入日志；因为没法回滚

######## 解决幂等性生产者的限制：不能跨分区、跨会话

####### 区别 RocketMQ 的半消息！

####### 2PC 两阶段提交

https://www.jianshu.com/p/f77ade3f41fd

###### 限制

####### 性能更差

##### 消费者 幂等性

###### 1. 通过“Unique Key”存储，保证消费幂等性

####### RDB / ES

###### 2. 通过RDB事务性，原子性写入记录、存储offset

####### 当重启时，通过 consumer.seek() 找到上次处理位置

### 消息丢失

#### 消息可靠性

##### 生产阶段

###### 丢失原因

####### 网络抖动

####### Leader replica 所在broker异常

###### 措施

####### 请求确认机制

####### 超时重传

###### 注意

####### 生产者异步发送消息：fire and forget

######## 慎用 producer.send(msg)

######## 推荐 producer.send(msg, callback)

或者producer.send(msg).get()

######### 在callback中重试

########## 直接配置 retires不就可以了？？

######### 在callback中正确处理异常？

####### 重传可能导致重复消费

##### 存储阶段

###### 丢失原因

####### 异步刷盘，不是实时存储到磁盘

###### 措施

####### 单机：写入磁盘后再给Producer返回确认

####### 集群：至少写入2个以上节点，再给Producer返回确认

######## ISR

######## acks=all

发送给Leader + 所有ISR之后，才发送确认

######## unclean.leader.election.enable = false

##### 消费阶段

###### 丢失原因

####### 接收时网络抖动，导致没收到

####### 处理时失败，但更新消费进度成功

####### 异步处理消息 + 自动提交

###### 措施

####### 确认机制

######## 推荐先消费、再更新位移

######## 异步处理消息时，不要开启自动提交位移

####### 如果broker没收到消费确认，则下次拉取时还会返回同一条消息

#### 消息丢失检测

##### 序号连续递增

发送端：拦截器注入序号；
接收端：拦截器检测连续性；

#### 什么情况下消息不丢失

##### 已提交的消息

###### 当若干个broker成功接收到消息，并写入到日志文件。

##### 有限度的持久化保证

###### 至少有一个broker存活 

#### 最佳实践

##### Producer设置

###### acks = all

####### 要求所有 ISR 副本Broker都要接收到消息，才算已提交

####### “所有 ISR 副本”！而非“所有副本”！

###### retries = big_value

####### 自动重试消息

###### producer.send(msg, callback)

####### callback 记录错误、或存储错误、或调用其他应用....

##### Broker设置

###### unclean.leader.election.enable=false

####### 禁止落后太多的broker成为leader

###### replication.factor >= 3

####### 将消息多保存几份

###### min.insync.replicas > 1

####### 写入多少个副本才算已提交

####### min.insync.replicas保证的写入副本的下限。

###### replication.factor > min.insync.replicas

####### 否则，只要一个副本挂了，整个分区就挂了

####### 还要考虑可用性

##### Consumer设置

###### enable.auto.commit = false

###### group.id

####### 如果某个消费者想要读取所有分区消息，则使用独立的 group.id

###### auto.offset.reset = earliest

####### 当没有位移可提交时（例如刚启动）从最小位置开始读取；可能导致重复

###### 位移提交

####### 并确保消息“处理”完成再提交

####### 提交频率：性能 vs. 重复

###### 重试

####### 1. 将待重试记录存入 buffer，下次poll时重试该buffer

######## 还可以调用 pause() 让下一次poll不返回新数据，更方便重试

####### 2. 将待重试记录写入单独主题

######## 类似 死信队列

######## 同一消费者组订阅 main topic + retry topic；或由另一个消费者组订阅 retry topic

### 消息积压

#### 问题

##### 马太效应

##### 消息堆积后，移出OS页缓存，失去zero copy，越堆越多

#### 优化

##### 发送端性能优化

###### 并发发送

####### RPC可直接在当前线程发送消息，因为rpc多线程

###### 批量发送

####### 例如离线分析系统

##### 消费端性能优化

###### 优化消费代码

####### 并行

###### 扩容

####### 注意如果大于分区数量则无效果

####### 注意 rebalance 的影响

###### 降级

#### 监控

##### 消费进度监控

###### kafka-consumer-groups 脚本

kafka-consumer-groups.sh 
  --bootstrap-server <Kafka broker 连接信息 > 
  --describe 
  --group <group 名称 >


####### CURRENT-OFFSET

######## 当前消费进度

####### LOG-END-OFFSET

######## 消息总数

####### LAG

######## 堆积数

###### API

```

public static Map<TopicPartition, Long> lagOf(String groupID, String bootstrapServers) {
  
  Properties props = new Properties();
  props.put(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
  
  try (AdminClient client = AdminClient.create(props)) {
  
  // 1.获取消费者组最新位移
  ListConsumerGroupOffsetsResult result = client.listConsumerGroupOffsets(groupID);
  
  Map<TopicPartition, OffsetAndMetadata> consumedOffsets = result.partitionsToOffsetAndMetadata().get(10, TimeUnit.SECONDS);
  
  props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);    props.put(ConsumerConfig.GROUP_ID_CONFIG, groupID);
  props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
  props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
  
  try (final KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
  
    // 2.获取订阅分区的最新位移
    Map<TopicPartition, Long> endOffsets = consumer.endOffsets(consumedOffsets.keySet());
    
    return endOffsets.entrySet().stream().collect(Collectors.toMap(
    entry -> entry.getKey(),
    // 3.做减法
    entry -> entry.getValue() - consumedOffsets.get(entry.getKey()).offset()));
   


```

####### AdminClient#listConsumerGroupOffsets()

######## 列出消费者组的最新offset

####### KafkaConsumer#endOffset()

######## 获取订阅分区的最新offset

###### JMX

kafka.consumer:type=consumer-fetch-manager-metrics,client-id=xx

####### records-lag-max

######## 堆积数

####### records-lead-min

######## = 消费位移 - 分区当前第一条消息位移

######### 监控快要被删除但还未被消费的数目

######## 原因：消息留存时间达到，之前的老消息被删除，导致lead变小

## 配置参数

### 操作系统配置

#### ulimit -n 1000000

##### 文件描述符限制

###### Topic

#### swappiness

##### 建议设一个小值，例如1

##### 但不要设成0，否则物理内存耗尽可能直接被OOM Killer

#### 提交时间 / Flush落盘时间

Kafka收到数据并不马上写入磁盘，而是写入操作系统Page Cache上。
随后操作系统根据LRU算法，定期将页缓存上的脏数据落盘到物理磁盘。
默认5秒，可适当调大。

### Broker配置

#### Broker连接相关配置

##### listeners

PLAINTEXT://localhost:9092

###### 告诉外部连接者通过什么协议访问主机名和端口开放的Kafka服务

##### advertised.listeners

###### 对外发布的

##### host.name/port

###### 过期参数

##### borkerlid

###### 默认为0，可以取任意值

##### zookeeper.connect

zk1:2181,zk2:2181,zk3:2181/kafka1

zk记录了元数据信息：
- 有哪些Broker在运行
- 有哪些Topic
- 每个Topic有多少分区
- 每个分区的Leader副本都在哪些机器上

###### 建议配置 chroot

###### 建议配置多个zk地址

#### 内存环境变量

$> export KAFKA_HEAP_OPTS=--Xms6g  --Xmx6g

$> export  KAFKA_JVM_PERFORMANCE_OPTS= -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true

$> bin/kafka-server-start.sh config/server.properties


##### KAFKA_HEAP_OPTS

###### 堆大小

##### KAFKA_JVM_PERFORMANCE_OPTS

###### GC参数

#### Retention 数据留存配置

##### log.dirs

###### 指定Broker使用的文件目录列表

###### 可配置多个

####### broker会按照“least-used”原则选择目录

####### least-used == 存储的分区最少，而非容量！

###### 建议挂在到不同物理磁盘

####### 提升读写性能

####### 实现故障转移

##### log.retention.hour|minutes|ms

###### 一条消息被保存多久

##### log.retention.bytes

###### Broker为消息保存的总磁盘大小

##### message.max.bytes

###### 一条消息最大大小

###### 默认值1000012，偏小！

##### log.cleaner.enabled

###### log compacted

####### 对每一个key，只保留最新的一条消息

####### 适合 change log 类型的消息

####### 原理

######## segment 分为两部分

######### Clean

########## 上次 compact 过的消息；clean部分每个key只对应一个value

######### Dirty

########## 上次compact过后新增的消息

######## 多个 compaction 线程

######### 遍历 Dirty 部分，构造 offset map

######### 遍历 Clean 部分，对 offset map进行补充

######### 替换 segment ?

#### 主题相关

##### auto.create.topics.enable=false

###### 是否允许自动创建主题

###### 应该设为false，把控主题名称

##### unclean.leader.election.enable=false

###### 是否允许unclean leader选举：允许落后很多的副本参与选举

###### 应该设为false，否则数据可能丢失

##### auto.leader.rebalance.enable=false

###### 允许定期进行Leader选举

###### 应该设为false

### Topic配置

#### 分区配置

##### num.partitions

###### 注意分区数只能增加，不能减少

####### 设得大一些，方便扩展消费者数目

###### 如何定值？

####### 期望吞吐量 / 消费者吞吐量

#### 消息配置

##### retention.ms | bytes

###### 默认7天，无限大

##### message.max.bytes

###### 定义一条消息最大大小

###### 大消息的缺点

####### 会增加磁盘写入块的大小，影响 IO 吞吐量

####### 处理网络连接和请求的线程要花费更多时间来处理

###### 联动参数

####### fetch.message.max.bytes

######## 消费者能读取的最大消息大小，如果 < max.message.bytes，则消费会被阻塞

####### replica.fetch.max.bytes

######## 集群同步的最大消息大小，原则同上

##### log.segment.ms | bytes

###### 单个 log segment的大小；达到该值时，当前日志片段会关闭 并创建新的日志片段

###### 如何定值？

####### 若太小，会频繁关闭和分配文件

####### 若太大，而写入数据量又很小，则可能等到超过 log.retention.ms 很多后才能被过期

#### 配置操作

##### kafka-topics.sh --config 创建topic时

bin/kafka-topics.sh
--bootstrap-server localhost:9092
--create
--topic transaction
--partitions 1
--replication-factor 1
--config retention.ms=15552000000
--config max.message.bytes=5242880


##### kafka-configs.sh

bin/kafka-configs.sh
--zookeeper localhost:2181
--entity-type topics
--entity-name transaction
--alter
--add-config
max.message.bytes=10485760


### 副本相关

#### Producer 端

##### acks

https://medium.com/better-programming/kafka-acks-explained-c0515b3b707e

###### 表示多少 Replica 写入成功才算真的写入成功

###### 取值

####### 0

######## 发完即认为成功

######### 保证性能

####### 1

######## 只要 Leader 写入成功即可

####### all

######## 需要所有 ISR 副本写入成功

######### 保证可靠性

#### Broker 端

##### min.insync.replica

###### 含义

####### 表示 acks=all 时最少需要多少个 ISR 存在

###### 注意

####### 可设置 broker or topic level

####### 配合acks

######## 如果acks = all，但ISR中只有 Leader 一个，则有问题：写入这一个即认为成功

####### 如果 ISR 少于min，则返回 NotEnoughReplicasException

######## 值越小，性能影响越小，但丢失数据的风险越大

##### replica.lag.time.max.ms

###### follower的延迟如果超过该值，则被踢出 ISR

##### replication.factor

###### 副本个数，消息被复制的份数，默认 3

###### 越大，则可用性可靠性越大；同时需要的 Broker 数目也越大

##### unclean.leader.election.enable

###### 默认=true，允许 OSR replica 被选为 leader

####### 提高可用性

######## 适用于用户行为跟踪

####### 缺点

######## 消息丢失

######## 数据不一致

###### 设为false，可保证 committed data 不丢失

####### 适用于银行支付信息

### 生产者配置

```
Properties p = new Properties();
p.put("bootstrap.servers", "b1:p1,b1:p2");
p.put("key.serializer", "");
p.put("value.serializer", "");

producer = new KafkaProduer<String, String>(p);
```

#### 必选配置

##### bootstrap.servers

##### key.serializer / value.serializer

#### 可选配置

##### 常用

###### acks

https://medium.com/better-programming/kafka-acks-explained-c0515b3b707e

####### 含义

######## 表示多少 Replica 写入成功才算真的写入成功

####### 取值

######## 0

######### 发完即认为成功

########## 保证性能、吞吐量，但会丢消息

######## 1

######### 只要 Leader 写入成功即可

########## 写入分区数据文件，但不一定同步到磁盘

########## Leader 选举阶段会返回 LeaderNotAvailableException

######## all

######### 需要所有 ISR 副本写入成功

########## 保证可靠性

###### retries

####### 含义

######## 遇到临时性错误时的重试次数

######### 可重试错误

########## e.g. A lack of leader for a partition

########## LEADER_NOT_AVAILABLE

######### 不可重试错误

########## e.g. message too large

########## INVALID_CONFIG

######## 配合 retry.backoff.ms

######### 默认退避 100ms

####### 注意

######## 无需在业务代码里处理重试！

######## 业务代码只需关注 不可重试的错误，或者重试失败的情况

######## MirrorMaker 会设置 retries = MAX

###### batch.size

####### 含义

######## 每个batch的字节数

######### 注意不是size!

####### 注意

######## 不宜过小

######### 过小会导致频繁发送消息

######### 生产者并不是非要等到 达到 batch.size才会发送出去

########## 配合 linger.ms

###### linger.ms

####### 含义

######## 每个batch发送之前的等待时间

####### 注意

######## 调大会增加延迟，也会提升吞吐量

######## 当发现生产者总是发送空batch，则应该增大该值

###### max.in.flight.requests.per.connection

####### 含义

######## 生产者在收到broker响应之前可以发送多少消息

####### 注意

######## 调大会增加内存占用，会提高吞吐量

######## 但设置过高会降低吞吐量

######### 因为 batching becomes less efficient

######## 如果对有序性要求高，建议设置为 1

######### 乱序场景

########## in.flight > 1&& retries > 0 时，依次发送batch-1 / 2 --> batch-1 失败 --> batch-2 成功 --> batch-1 重试成功，则乱序

######### 设置 in.flight = 1 可确保当有重试时，下一个消息不会被发送

########## 但会严重影响吞吐量

########## 或者设置 retries = 0，但会影响 reliable

###### compression.type

####### 取值

######## gzip

######### CPU 消耗高，压缩比更大

######## snappy

######### CPU 消耗少，压缩比可观

######## LZ4

######## Zstandard (zstd)

####### 注意

######## vs. Broker压缩

######### 如果Broker指定的compression.type != producer，则会重新进行解压压缩；

######### 还会丧失zero copy特性

######## 解压

######### 只会发生在 Consumer 端

########## 如何设置每条消息的 offset、校验 checksum？

######### Producer 端压缩、Broker 端保持、Consumer 端解压缩

##### 不常用

###### buffer.memory

####### 含义

######## 生产者内存缓冲区大小

####### 跟batch什么关系？

###### client.id

####### 含义

######## 客户端的标识，可以为任意字符串

######## 用户 日志、metrics、quotas

###### 超时

####### request.timeout.ms

######## Producer：等待发送数据返回响应的超时

####### metadata.fetch.timeout.ms

######## Producer：请求元数据的超时

####### timeout.ms

######## Broker：等待 In-Sync Replica 返回消息确认的超时

######## 与 acks 相关

######### 若超时，且 acks = all 则认为写入失败

###### max.block.ms

####### 含义

######## 调用 send() 时的阻塞时间

######### 阻塞场景：send buffer 已满

######## 调用 partitionFor()获取元数据时的阻塞时间

######### 阻塞场景：元数据不可用

####### 注意

######## 会抛出 timeout 异常

###### max.request.size

####### 含义

######## 生产者发送的请求大小

####### 注意

######## 配合 Broker端参数：message.max.bytes

######### 最好取值一样

######### 否则生产者发出的大消息 可能被Broker拒绝

###### receive.buffer.bytes / send.buffer.bytes

####### 含义

######## TCP Socket 发送和接受缓冲区大小

### 消费者配置

#### 必选配置

##### bootstrap.servers

##### key.deserializer / value.deserializer

##### group.id

#### 可选配置

##### fetch.min.bytes

###### 含义

####### 消费者能获取的最小字节数

####### broker: 收到消费者请求时如果数据量 < fetch.min.bytes，会等待 fetch.max.wait.ms，直到有足够的数据

###### 注意

####### 提高该值，可以减少网络来回，减少 broker和consumer负载

######## 如果消费者 CPU高、且可用数据量不大，调高该值

######## 如果消费者过多 导致Broker负载过高，调高该值

####### 如果发现 fetch-rate 很高，应该增大该值

####### 配合 fetch.max.wait.ms

##### fetch.max.bytes

###### 如果通过监控发现 fetch-size-agv/max 接近 fetch.max.bytes，则应该增大该值

##### fetch.max.wait.ms

###### 含义

####### 等待足够数据的时间

###### 注意

####### 调高会增加 Latency

##### max.partition.fetch.bytes

###### 含义

####### 服务器从每个分区能返回的最大字节数

####### 每次 poll() 从“单个分区”返回的数据 不会超过此值

###### 注意

####### 必须 > max.message.size

######## 否则会导致大消息无法被消费，consumer被hang住

####### 如果设置过大，会导致 consumer处理耗时，影响poll()频率

##### max.poll.record

###### 含义

####### poll() 返回的最大记录个数

https://time.geekbang.org/column/article/102132

目前java consumer的设计是一次取出一批，缓存在客户端内存中，然后再过滤出max.poll.records条消息返给你 

???

####### 若无限制，broker可能返回超大消息，导致消费者OOM

###### 注意

####### vs. fetch.min.bytes

######## 如果 max.poll.record < fetch.min.bytes，什么结果???

####### vs. max.partition.fetch.bytes

######## 个数 vs. 大小

######## 总体 vs. 按分区

##### session.timeout.ms

###### 含义

####### 消费者被认为死亡的超时时间，超时后会触发 Rebalance

###### 注意

####### 配合 heartbeat.internal.ms

######## 通常一起修改、必须比它大

######## 推荐设置 session.timeout.ms = 3 * heartbeat.internal.ms

####### 调高 session.timeout 可以减少不必要的 rebalance，但可能导致“故障探测时间”更长

##### auto.offset.reset

###### 含义

####### 消费者读取一个没有偏移量、或偏移量无效的分区时，该如何处理

######## 原因：消费者死掉好久，包含偏移量的记录已被删除

###### 取值

####### latest

######## 从最新记录开始读取

####### earliest

######## 从起始位置开始读取

##### enable.auto.commit

###### 含义

####### 是否自动提交，默认为 true

###### 注意

####### 配合 auto.commit.interval.ms

####### 设为false，可减少重复、丢失

##### partition.assignment.strategy

###### 含义

####### 给消费者分配 Partition 的策略

###### 取值

####### Range

######## 把主题的若干“连续”分区分配给消费者

######### C1: t1_p0, t1_p1, t2_p0, t2_p1
C2: t1_p2, t2_p2

连续分区 分配给C1

######## 分配可能不均衡

####### RoundRobin

######## 把主题的所有分区“逐个”分配给消费者

######### C1: t1_p0, t1_p2, t2_p1
C2: t1_p1, t2_p0, t2_p2

连续分区 分配给C1

######## 分配更均衡

##### receive.buffer.bytes / send.buffer.bytes

###### 含义

####### TCP Socket 发送和接受缓冲区大小

##### client.id

## 运维

### 安装

#### Broker安装

##### 启动

###### kafka-server-start.sh

bin/kafka-server-start.sh 
  -daemon
  config/server.properties

##### 主题相关

###### kafka-topics.sh

创建主题
kafka-topics.sh
  --create
  --bootstrap-server host:port
  --zookeeper localhost:2181 (v2.2以前)
  --replication-factor 1
  --partitions 1
  --topic topic_name

描述主题
kafka-topics.sh
  --zookeeper localhost:2181
  --describe
  --topic topic_name

列举主题
kafka-topics.sh 
  --bootstrap-server host:port 
  --list


删除主题（异步）
kafka-topics.sh 
  --bootstrap-server host:port 
  --delete  
  --topic <topic_name>

增加分区（可能导致数据迁移）
kafka-topics.sh 
  --bootstrap-server host:port 
  --alter 
  --topic <topic_name> 
  --partitions < 新分区数 >

减少分区：不支持，因为会导致数据删除
--> 删除主题、重建



###### 变更副本数

####### kafka-reassign-partitions.sh

bin/kafka-reassign-partitions.sh 
  --zookeeper zookeeper_host:port 
  --reassignment-json-file reassign.json 
  --execute

reassign.json:

{"version":1, "partitions":[
 {"topic":"__consumer_offsets","partition":0,"replicas":[0,1,2]}, 
  {"topic":"__consumer_offsets","partition":1,"replicas":[0,2,1]},
  {"topic":"__consumer_offsets","partition":2,"replicas":[1,0,2]},
  {"topic":"__consumer_offsets","partition":3,"replicas":[1,2,0]},
  ...
  {"topic":"__consumer_offsets","partition":49,"replicas":[0,1,2]}
]}`



###### 主题分区迁移

####### kafka-reassign-partitions.sh

###### 修改主题级别参数

####### kafka-configs.sh

修改主题级别
kafka-configs.sh 
  --zookeeper host:port 
  --entity-type topics 
  --entity-name <topic_name> 
  --alter 
  --add-config max.message.bytes=10485760


###### 修改主题限速

####### kafka-configs.sh

kafka-configs.sh 
  --zookeeper zookeeper_host:port 
  --alter 
  --add-config 'leader.replication.throttled.rate=104857600,follower.replication.throttled.rate=104857600' 
  --entity-type brokers 
  --entity-name <broker-id>

kafka-configs.sh 
  --zookeeper zookeeper_host:port 
  --alter 
  --add-config 'leader.replication.throttled.replicas=*,follower.replication.throttled.replicas=*' 
  --entity-type topics 
  --entity-name test



##### 发布消息

###### kafka-console-producer.sh

bin/kafka-console-producer.sh 
  --broker-list localhost:9092 
  --topic topic-demo
>Hello, Kafka!

##### 消费消息

###### kafka-console-consumer.sh

bin/kafka-console-consumer.sh 
  --bootstrap-server localhost:9092 
  --group test-group
  --from-beginning
  --consumer-property enable.auto.commit=false
  --topic topic-demo

##### 测试

###### 查看消息文件数据

####### kafka-dump-log

$ bin/kafka-dump-log.sh 
  --files ../data_dir/kafka_1/test-topic-1/00000000000000000000.log 
  --deep-iteration 
  --print-data-log


###### 消费者组信息

####### kafka-consumer-groups.sh

###### 性能测试

####### kafka-producer-perf-test

$ bin/kafka-producer-perf-test.sh 
  --topic test-topic 
  --num-records 10000000 
  --throughput -1 
  --record-size 1024 
  --producer-props bootstrap.servers=kafka-host:port 
  acks=-1 
  linger.ms=2000 
  compression.type=lz4


####### kafka-consumer-perf-test

$ bin/kafka-consumer-perf-test.sh 
  --broker-list kafka-host:port 
  --messages 10000000 
  --topic test-topic

#### 流处理应用搭建

##### Kafka Connect 组件

cd kafka_2.12-2.3.0

bin/connect-distributed.sh config/connect-distributed.properties


###### 负责收集数据：KV存储，数据库，搜索系统，文件

###### 创建

$ curl -H "Content-Type:application/json" -H "Accept:application/json" http://localhost:8083/connectors -X POST --data '
{"name":"file-connector","config":{"connector.class":"org.apache.kafka.connect.file.FileStreamSourceConnector","file":"/var/log/access.log","tasks.max":"1","topic":"access_log"}}'

{"name":"file-connector","config":{"connector.class":"org.apache.kafka.connect.file.FileStreamSourceConnector","file":"/var/log/access.log","tasks.max":"1","topic":"access_log","name":"file-connector"},"tasks":[],"type":"source"}


##### Kafka Streams 组件

###### 负责实时处理

##### 组件

###### Kafka Connect + Kafka Core + Kafka Streams

##### 其他方案

###### Flume + Kafka + Flink

###### Spark Streaming

###### Flink

### 配置

#### 动态配置

##### 配置分类

###### per-broker

###### cluster-wide

###### 静态配置

####### server.properties

###### kafka 默认值

##### 原理

###### 保存在 zk 持久化节点

####### /config/brokers/

/config/brokers/<default> 
cluster-wider 动态参数

/config/brokers/<broker_id>
per-broker 动态参数

##### 设置

###### kafka-configs.sh --alter --add-config

cluster-wider:
$ bin/kafka-configs.sh 
  --bootstrap-server host:port 
  --entity-type brokers 
  --entity-default 
  --alter 
  --add-config unclean.leader.election.enable=true

per-broker:
$ bin/kafka-configs.sh 
  --bootstrap-server host:port 
  --entity-type brokers 
  --entity-name 1 
  --alter 
  --add-config unclean.leader.election.enable=false


##### 常用动态配置项

###### log.retention.ms

###### num.io.threads / num.network.threads

###### num.replica.fetchers

#### 集群配置参数

##### /config/server.properties

#### 分区管理

##### kafka-preferred-replica-election.sh

###### 触发 prefered replica election，让broker重置首领

##### kafka-reassign-partitions.sh

###### 修改副本分配

###### 修改分区的副本因子 replica-factor

##### kafka-replica-verification.sh

###### 验证副本

### 运维

#### 管理工具

##### KafkaAdminClient

###### 作用

####### 实现.sh的各种功能

###### 原理

####### 前端主线程

######## 将操作转成对应的请求，发送到后端IO线程队列中

####### 后端IO线程

######## 从队列中读取请求，发送到对应的Broker；把结果保存起来，等待前段线程来获取

######### wait / notify 实现通知机制

######## kafka-admin-client-thread-xx

###### 代码

```java
Properties props = new Properties();
props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-host:port");
props.put("request.timeout.ms", 600000);

String groupID = "test-group";

try (AdminClient client = AdminClient.create(props)) {

  ListConsumerGroupOffsetsResult result = client.listConsumerGroupOffsets(groupID);
  
  Map<TopicPartition, OffsetAndMetadata> offsets = 
 result.partitionsToOffsetAndMetadata().get(10, TimeUnit.SECONDS);
 
  System.out.println(offsets);
}


```

##### MirrorMaker

###### 跨集群镜像

###### 本质是一个 消费者 + 生产者 程序

#### 安全

##### 认证机制

###### SSL / TLS

Transport Layer Securit

####### 一般只用来做通信加密；而非认证

###### SASL

####### 用于认证

####### 认证机制

######## GSSAPI

######### 基于 Kerberos 认证

######## PLAIN

######### 简单用户名密码

########## 不能动态增减用户，必须重启Broker

######## SCRAM

######### PLAIN进阶

########## 认证信息保存在ZK，可动态修改

######## OAUTHBEARER

######### 基于OAuth2

######## Delegation Token

##### 授权管理

###### 权限模型

####### ACL

######## Access Control List

####### RBAC

######## Role Based Access Control

####### ABAC

######## Attribute Based Access Control

####### PBAC

######## Policy Based Access Control

###### Kafka ACL

####### 启用：server.properties

authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer


####### 设置：kafka-acls

$ kafka-acls 
  --authorizer-properties
  zookeeper.connect=localhost:2181 
  --add 
  --allow-principal User:Alice 
  --operation All 
  --topic '*' 
  --cluster


#### 监控

##### 线程

###### 服务端

####### Log Compaction 线程

######## kafka-log-cleaner-thread-*

####### 副本拉取线程

######## ReplicaFetcherThread*

###### 客户端

####### 生产者消息发送线程

######## kafka-producer-network-thread-*

####### 消费者心跳线程

######## kafka-coordinator-heartbeat-thread-*

##### 端到端监控

###### Kafka Monitor

https://github.com/linkedin/kafka-monitor

####### Linkedin

###### Kafka Manager

####### Yahoo，GUI界面

###### Burrow

####### Linkedin，监控消费进度

###### Kafka Eagle

###### JMXTrans + InfluxDB + Grafana

##### 指标

###### Broker

####### Under-Replicated 非同步分区数

MBEAN:
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions

######## 概念

######### 作为首领的Broker有多少个分区处于非同步状态

######## 原因

######### 数量稳定时：某个Broker离线

具体现象：
- 有问题的Broker不上报metric;
- 其他Broker的under-replicated数目 == 有问题的Broker上的分区数目。


########## 排查：此时先执行“preferred replica election” 再排查

######### 数量变动时：性能问题

########## 排查：通过 kafka-topics.sh --describe --under-replicated 列出非同步分区

########## Cluster 问题

########### 负载不均衡

############ 排查：看每个Broker的 分区数量、Leader分区数量、消息速度 是否均衡

############ 解决：kafka-reassign-partitions.sh

自动化：linkedin kafka-assign

########### 资源耗尽

############ 排查：关注 all topics bytes in rate 趋势

########## Broker问题

########### 硬件问题

########### 进程冲突

############ top 命令

########### 本地配置不一致

############ 使用配置管理系统：Chef / Puppet

####### Offline 分区数

MBEAN:
kafka.controller:type=KafkaController,name=OfflinePartitionsCount

######## 概念

######### 没有首领的分区个数

######### 只在 Controller上会上报此 metric

######## 原因

######### 所有相关的broker都挂了

######### ISR 副本未能成为 leader，因为消息数量不匹配

########## ？

####### Active controller 数目

MBEAN:
kafka.controller:type=KafkaController,name=ActiveControllerCount

######## = 1

######### 正常

######## = 0

######### 可能zk产生分区

######## > 1

######### 一个本该退出的控制器线程被阻塞了；需要重启brokers

######## ActiveControllerCount

######### 如果多台broker该值为1，则出现脑裂

####### Request handler 空闲率

MBEAN:
kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent

######## 含义

######### 请求处理器负责处理客户端请求，包括读写磁盘

######### 空闲率不可太低，< 20% 则潜在问题

######## 原因

######### 集群过小

######### 线程池过小

########## == cpu核数

######### 线程负责了不该做的事

########## 例如老版本会负责消息解压、验证消息、分配偏移量、重新压缩

0.10 的优化：
- batch里可以包含相对位移。
- broker则无需再解压、重新压缩；

####### All topics bytes in / out

MBEAN:
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec

kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec

######## 含义

######### 消息流入流出速率

######### bytes in: 据此决定是否该扩展集群

######### bytes out: 也包含 replica的流量

####### Partition count

MBEAN:
kafka.server:type=ReplicaManger,name=PartitionCount

######## 含义

######### 每个Broker上的分区数目

######### 如果运行自动创建分区，则需关注此指标

####### Leader count

MBEAN:
kafka.server:type=ReplicaManger,name=LeaderCount

######## 含义

######### 每个Broker上的Leader分区数目

######### 用于判断集群是否不均衡

########## 执行 preferred replica election 来重平衡

####### Reqeust Metrics

######## Total time

######## request per second

###### Topic

####### Bytes in/out rate

MBEAN:
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=X

####### Failed fetch rate

MBEAN:
kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestPerSec,topic=XX

####### Fetch request rate

MBEAN:
kafka.server:type=BrokerTopicMetrics,name=TotalFetchRequestPer,topic=XX

####### Produce request rate

MBEAN:
kafka.server:type=BrokerTopicMetrics,name=TotalProduceRequestPerSec,topic=XX

###### Partition

####### Partition Size

MBEAN:
kafka.log:type=Log,name=Size,topic=XX,partition=0

####### Log segment count

MBEAN:
kafka.log:type=Log,name=NumLogSegments,topic=XX,partition=0

####### Log start/end offset

MBEAN:
kafka.log:type=Log,name=LogStartOff,topic=XX,partition=0

####### ISRShrink / ISRExpand

######## 副本频繁进出ISR

什么原因？


###### 生产者

####### Overall

MBEAN:
kafka.producer:type=producer-metrics,client-id=CLIENT

######## 错误相关

######### record-error-rate

########## 经过重试仍然失败，表明严重错误

######### record-retry-rate

######## 性能相关

######### request-latency-avg

######### record-queue-time-avg

######## 流量相关

######### outgoing-byte-rate

########## 每秒消息大小

######### record-send-rate

########## 每秒生产的消息数目

######### request-rate

########## 每秒发送的请求数目

########### = N * batch?

######## 大小相关

######### request-size-avg

######### batch-size-avg

########## 据此调整 batch设置

######### record-size-avg

########## = msg

######### record-per-request-avg

########## 关联 max.partition.bytes / linger.ms

######## 限流

######### produce-throttle-time-avg

MBEAN:
kafka.producer:type=producer-metrics,client-id=CLIENT, attribute produce-throttle-time-avg

####### Per-Broker

MBEAN:
kafka.producer:type=producer-node-metrics,client-id=CLIENTID,node-id=node-BROKERID

####### Per-Topic

MBEAN:
kafka.producer:type=producer-topic-metrics,topic=TOPICNAME

######## 适用于 MirrorMaker 

###### 消费者

####### Overall

MBEAN:
kafka.consumer:type=cons-metrics,client-id=CLIENT

####### Fetch Manager

MBEAN:
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=CLIENT

######## 性能

######### fetch-latency-avg

########## 关联 fetch.min.bytes / fetch.max.wait.ms

########## 对于慢主题，该指标会时慢时快

######## 流量

######### bytes-consumed-rate

######### records-consumed-rate

######### fetch-rate

######## 大小

######### fetch-size-avg

######### records-per-request-avg

######### 注意没有类似 record-size-avg，消费者并不能知道每条消息的大小？

######## 限流

######### fetch-throttle-time-avg

MBEAN:
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=CLIENT, attribute fetch-throttle-time-avg

######## Lag

######### records-lag-max

########## 只考虑lag最大的那个分区

########## consumer 挂了就监控不到lag了

######### records-lag

######### 外部监控：Burrow

https://engineering.linkedin.com/apache-kafka/burrow-kafka-consumer-monitoring-reinvented

####### Per-Topic

MBEAN:
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=CLIENT,topic=XX

######## 如果消费者对应多个主题，则可用

####### Per-Broker

MBEAN:
kafka.consumer:type=consumer-node-metrics,client-id=CLIENT,node-id=XX

####### Coordinator

MBEAN:
kafka.consumer:type=consumer-coordina-metrics,client-id=CLIENT

######## sync-time-avg / sync-rate

######## commit-latency-avg

######## assigned-partitions

### 问题排查

#### 主题删除失败

##### 原因

###### 副本所在Broker宕机

###### 待删除主题部分分区依然在执行迁移过程

##### 解决

###### 手动删除zk /admin/delete_topics/xx

###### 手动删除该主题在磁盘上的分区目录

###### 执行zk rmr /controller，触发controller重选举

#### __consumer_offsets 磁盘占用大

##### 原因

###### kafka-log-cleaner-thread 线程挂掉，无法清理此内部主题

###### 用 jstack 确认

##### 解决

###### 重启broker

## 高级应用

### Connect

#### 概念

##### 通过“connector + 配置”实现数据移动：Kafka <--> 其他存储

##### 无需编码实现 Producer / Consumer

#### 组件

##### Worker Processes

###### Connect 集群

###### Worker 重启时，会 rebalance 重新分配 connectors / tasks

###### 职责

####### 是 Connector / Task 的容器

####### 处理 REST API、配置管理、可靠性、高可用、负载均衡

####### offset 自动提交

######## Source Connector

######### worker 包含逻辑partition (数据表)、逻辑offset (主键)，将其发送到Kafka broker

######## Sink Connector

######### 写入数据后，提交 offset

####### 重试 （task 报错时）

##### Connector Plugins

###### 负责移动数据

###### 可通过 REST API 配置和管理 connector

###### 职责

####### 决定需要运行多少个 Task：并行度

######## = min { max.tasks, 表的个数 }

####### 按照任务来 拆分数据复制

####### 从worker获取任务的配置，并启动任务

##### Tasks

###### Connector 执行 tasks

####### Source connector tasks

######## 读取数据，提供Connect数据对象给 worker processes

####### Sink connector tasks

######## 从worker processes获取 connect数据对象，写入目标系统

###### 职责

####### 负责实际的数据读取、写入

##### Convertors

###### 支持将数据对象存储为不同的格式：JSON / Avro / string

#### 场景

##### 如果可以修改应用代码，则用传统API；否则用 Connect

##### 关注点分离

###### Connect 内置处理 REST API、配置管理、可靠性、高可用、负载均衡

##### 类比

###### Flume

###### Logstash

###### Fluentd

#### 配置

##### 1. 安装 connector plugin

###### github --> mvn install --> 拷贝到 /libs

##### 2. 启动 Connect Workers

###### connect-distributed.sh config

##### 3. 配置 connector plugin

###### 命令

####### POST host/connectors 

####### 查可选输入参数

######## GET host/connector-plugins/JdbcSourceConnector/config/validate 

###### 类别

####### Source connector

####### Sink connector

### MirrorMaker

#### 概念

##### 在不同数据中心之间同步数据

##### 原则

###### 一个数据中心至少一个kafka集群

###### exactly-once 复制

###### 尽量从远程数据中心 消费数据，而不往远程数据中心 生产数据

#### 架构

##### Hub-Spokes 架构

###### 概念

####### 一个中央集群，从其他分支集群拉取数据、聚合

######## 简化：两个集群，一个 Leader， 一个 Follower

####### 场景：各分支集群数据集完全隔离、没有依赖

######## 例如银行的各个分行

###### 优点

####### 简单：易于部署、配置、监控

######## 生产者只关心本地集群

######## Replication 是单向的

###### 缺点

####### 不可跨区读取

##### Active-Active 架构

###### 概念

####### 各数据中心 共享部分或全部数据

####### 互相复制

###### 优点

####### 便于就近服务

####### 数据冗余、可靠

###### 缺点

####### 数据异步读写，需要避免数据冲突

######## 同步延迟问题

######### stick-session

######## 数据冲突问题：多个集群同时写入同一个数据集

####### Mirroring Process 过多

######## 要避免“来回复制”

一般可引入“逻辑主题” / “namespace”：
- 常规主题：user
- 逻辑主题：clusterA-user, clusterB-user

##### Active-Standby 架构

###### 概念

####### 一个数据中心专门用于 Inactive / Cold 复制，而不对外提供服务

###### 优点

####### 简单；无需考虑冲突

###### 缺点

####### 数据丢失

####### 资源浪费

######## 优化

######### DR 集群用较小规模 --> 有风险！

DR = Disaster Recov

######### DR 集群服务部分只读请求

####### Failover 困难

######## 数据丢失、不一致

######## Failover 之后的“起始偏移量”

1. Auto offset reset: 
earliset / latest

2. Replicate offset topic: 
对 `__consumer_offsets` 主题进行镜像；


3. Time-based failover:
每个消息包含一个 timestamp，表示何时被写入kafka；Broker 支持根据 timestamp  查找offset

4. External offset mapping:
用外部存储保存两个数据中心的偏移量对应关系。

##### Stretch Clusters

###### 概念

####### 跨数据中心，部署单一的 Kafka 集群

######## 通过 Rack

###### 优点

####### 同步复制

####### 没有资源浪费

#### 原理

##### 组件

###### 一个 Producer

####### 用于写入 Target Cluster

###### 多个 Consumer

####### 用于读取 Source Cluster

####### 所有consumer属于同一个消费者组

####### 个数可配

######## num.streams

####### 每隔 60s 提交 offset 到 Source Cluster

######## auto.commit.enable = false

##### 注意

###### MirrorMaker 部署位置

####### 一般部署在 Target Cluster

######## “远程消费” 比 “远程生产” 更安全，否则可能丢失数据

如果消费者断链，只是不能读取，消息还会保存在Broker;
而如果生产者断链，则消息可能丢失

####### 如果集群间是加密传输，则可部署在 Source Cluster

######## 消费加密传输更影响性能

######## 此时注意 acks = all，减少数据丢失

###### 部署多个 MirrorMaker 实例，可以提高吞吐量

####### 注意要是同一个消费者组

###### 建议提前创建好主题，否则会按照默认配置自动创建主题

#### 问题

##### Rebalance 期间会停止消费

###### whitelist 正则匹配，分区变化 会导致 Rebalance

###### MirrorMaker 实例增删 会导致Rebalance

###### 参考 Uber uReplicator

https://eng.uber.com/ureplicator-apache-kafka-replicator/

####### 引入 Apache Helix 提供分片管理

####### 引入自定义 Consumer，接受Helix分配的分区

##### 配置信息如何保证一致

###### Confluent's Replicator

https://www.confluent.io/product/confluent-platform/global-resilience/

### Stream

#### 概念

持续地从一个无边界的数据集读取数据，进行处理、生成结果，那么就是流式处理。

##### 名词

###### 时间 Time

####### 分类

######## 事件时间 Event time

######### 事件的发生时间、记录的创建时间

######## 日志追加时间 Log append time

######### 事件达到Broker的时间

######## 处理时间 Processing time

######### stream-processing应用收到并处理事件的时间

###### 状态 State

####### 存储于事件与事件之间的信息称为状态

####### 分类

######## 本地状态、内部状态

######### 存储在应用实例内存中

######## 外部状态

######### 存储在外部数据存储中，例如Cassandra

###### 流 vs. 表：流表二元性

####### 流包含修改历史（ChangeLog），而表是一个快照（Snapshot）

####### 表 --> 流

######## CDC: Change Data Capture

####### 流 --> 表

######## Materializing

回放stream中的所有biang

###### 时间窗口 Time Windows

####### 窗口大小

####### 移动间隔 Advance Interval

######## 滚动窗口 Tumbling Window

######### 当移动间隔 == 窗口大小

######## 滑动窗口 Sliding Window

######### 当窗口随着每条记录移动

####### 多久之后禁止更新？

##### 特点

###### 无界

###### 有序

###### 数据记录不可变

###### 可重播

#### 设计模式

##### 单个事件处理 
Single-Event Processing

https://github.com/gwenshap/kafka-streams-wordcount/blob/master/src/main/java/com/shapira/examples/streams/wordcount/WordCountExample.java

###### 又称 Map / Filter 模式：转换、过滤

####### 独立地处理单个事件

####### 无需维护内部状态，易于恢复

###### 例子

####### 从流中读取日志消息，将ERROR写入一个高优先级流，其他日志写入低优先级流

####### 从流中读取JSON，转成Avro

##### 使用本地状态 
Processing with Local State

https://github.com/gwenshap/kafka-streams-stockstats/blob/master/src/main/java/com/shapira/examples/streams/stockstats/StockStatsExample.java

###### 当需要“group by 聚合”数据时，需要维护状态

###### 例子

####### 每个股票的一日均价

###### 注意

####### 内存使用

####### 持久化

######## RocksDB：同时存入磁盘

######## Kafka Streaming: 将状态同时发送到一个主题，通过Log Compaction保证容量

####### 重平衡

######## 重平衡后的新consumer需要能读取到最新的状态

##### 多阶段处理 / 重分区
Multiphase Processing / Repartitioning

###### “高级聚合”需要同时使用多个partition的数据，则需要阶段处理

####### 阶段一：聚合单个分区

####### 阶段二：聚合所有分区的结果

###### 例子

####### 当日股票市场的 top 10 股票

######## 1. 对每只股票计算当日收益，写入单一分区

######## 2. 使用单一程序 处理此单一分区

####### 类似 MapReduce

##### 使用外部查找: 流与表连接
Processing with External Lookup: Stream-Table Join

https://github.com/gwenshap/kafka-clickstream-enrich/blob/master/src/main/java/com/shapira/examples/streams/clickstreamenrich/ClickstreamEnrichment.java

###### 读取外部Table数据进行 Enrich --> CDC 更新本地 Cache

###### 例子

####### 使用数据库存储的规则，校验交易

####### 将用户点击流 关联到用户信息表 --> 生成 Enriched clicks topic

##### 流与流连接
Streaming Join

https://github.com/gwenshap/kafka-clickstream-enrich/blob/master/src/main/java/com/shapira/examples/streams/clickstreamenrich/ClickstreamEnrichment.java

###### 基于同一个时间窗口、同一个key

####### 对比 Stream-Table 连接：table只用考虑当前数据，而stream需要连接所有相关的历史变更

####### 需要保证按key分区，不同主题的同一个分区号被分配到相同的任务

###### 例子

####### 流1：搜索事件流；流2：搜索结果点击流

##### 乱序事件
Out-of-Sequence Events

###### 乱序很常见，尤其对IoT事件

###### 需要处理的场景

####### 识别乱序

######## 检查Event Time

####### 决定是reconcile还是忽略

######## 定一个时间阈值

####### Reconcile能力

####### 支持更新结果

##### 重新处理
Reprocessing

###### 两个变种

####### 对流式处理应用进行了改进，生成新的结果、对比、切换

######## 更安全，更好回退

####### 对老的流式处理应用进行修复，重新计算结果

#### 架构

##### 拓扑 Topology

###### 又称 DAG，定义了一系列操作

###### Processors 操作算子

####### 操作类型

######## filter, map, aggregate

####### 种类

######## Source Processors

######## Sink Processors

##### Scaling

###### 手段

####### 同一实例内可以开启多个执行线程

####### 可以启动多个实例

###### Tasks

####### 将拓扑分为多个task，以便并行执行

######## 每个 Task 独立运行

####### Task 数目

######## 取决于Streams引擎、分区数

##### Failover

###### 本地状态存储在kafka

###### 任务失败时，通过消费者协调机制，保证高可用

##### 精确一次语义

Kafka Streams 在底层大量使用 Kafka 事务机制和幂等性 Producer 来实现多分区的原子性写入，又因为它只能读写 Kafka，因此 Kafka Streams 很容易地就实现了端到端的 EOS。

###### Kafka 事务机制

###### 幂等性 Producer

## 文档

### 参考

#### 论文

https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying


https://www.kancloud.cn/kancloud/log-real-time-datas-unifying/58708


#### doc

http://kafka.apache.org/documentation/#design

#### acks vs. min.insync.replicas

https://medium.com/better-programming/kafka-acks-explained-c0515b3b707e

#### improvement proposal

https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals

### 源码

#### Consumer

##### 变量

###### SubscriptionState

####### 维护订阅主题、分区的消费位置

####### 消费者订阅时，传入 topic list，存于此处

###### ConsumerMetadata

####### 维护Broker节点、主题和分区在节点上的分布

####### 维护Coordinator 给消费者分配的分区信息

##### subscribe(topic list): 订阅

```
public void subscribe(Collection<String> topics, ConsumerRebalanceListener listener) {
  acquireAndEnsureOpen();
   ry {
      // 省略部分代码

      // 重置订阅状态
      this.subscriptions.subscribe(new HashSet<>(topics), listener);

      // 更新元数据
      metadata.setTopics(subscriptions.groupSubscription());
  } finally {
      release();
  }
}

```

###### aquireAndEnsureOpen()

####### 保证只能被单线程调用

###### subscriptions.subscribe()

####### 存储主题列表

###### metadata.setTopics()

```
public synchronized int requestUpdate() {
  this.needUpdate = true;
  return this.updateVersion;
}
```

####### 只是更新标志位 needUpdate = true

####### 在poll() 之前会实际去更新元数据

##### poll(): 拉取消息

###### updateAssignmentMetadataIfNeeded


> coordinator.poll()
> > ConsumerNetworkClient.ensureFreshMetadata()
> >
> > > ConsumerNetworkClient.poll()  与cluster通信。

####### 更新元数据

####### 底层用ConsumerNetworkClient，封装网络通信，异步实现

###### pollForFetches

```
 private Map<TopicPartition, List<ConsumerRecord<K, V>>> pollForFetches(Timer timer) {

  // 如果缓存里面有未读取的消息，直接返回这些消息
  final Map<TopicPartition, List<ConsumerRecord<K, V>>> records = fetcher.fetchedRecords();
  
  if (!records.isEmpty()) {
      return records;
  }
  
  // 构造拉取消息请求，并发送
  fetcher.sendFetches();

  // 发送网络请求拉取消息，等待直到有消息返回或者超时
   client.poll(pollTimer, () -> {
      return !fetcher.hasCompletedFetches();
   });
   
   // 返回拉到的消息
   return fetcher.fetchedRecords();
 }

```

####### 拉取消息

####### fetcher.fetchedRecords(): 若已有数据，则直接返回

######## 从ConcurrentLinkedQueue中获取

####### fetcher.sendFetches(): 构造拉取消息请求，并发送

######## 回调：rep 会被暂存到ConcurrentLinkedQueue

####### client.poll(): 发送网络请求

######## 等待有消息返回，或超时

####### fecther.fetchedRecords()
