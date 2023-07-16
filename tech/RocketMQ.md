# | 客户端

RocketMQ 的客户端连接服务端是需要经过客户端寻址的。首先和 NameServer 完成寻址，拿到 Topic/MessageQueue 和 Broker 的对应关系后，接下来才会和 Broker 进行交互。



## || Producer

分区选择：

- **轮询策略**：保证了每个 Queue 中可以均匀地获取到消息
- **最小投递延迟策略**：统计每次消息投递的时间延迟，然后根据统计出的结果将消息投递到时间延迟最小的 Queue



## || Consumer



消费模型

- Pull：默认
- Push：底层用一个 Pull 线程不断地去服务端拉取数据，拉到数据后，触发客户端设置的回调函数
- Pop：将消费分区、分区分配关系、重平衡都移到了服务端，减少了重平衡机制给客户端带来的复杂性。



消费进度保存

- RocketMQ 会为每个消费分组维护一份消费位点信息，信息中会保存消费的最大位点、最小位点、当前消费位点等内容。





# |服务端

## || NameServer

目的：

- 为客户端提供关于 topic 的**路由信息**。

  > 生产者如何知道消息要发往那台服务器？
  >
  > 服务器宕机后，生产者如何在不重启的情况下感知？

- 元数据存储
  - RocketMQ 的元数据信息实际是存储在 Broker 上的，Broker 启动时将数据上报到 NameServer 模块中汇总缓存。
  - NameServer 是一个简单的 TCP Server，专门用来接收、存储、分发 Broker 上报的元数据信息。这些元数据信息是存储在 NameServer 内存中的，NameServer 不会持久化去存储这些数据。



交互

- Broker 每隔 30s 向 NameServer 的每一台机器发送心跳包，其中包含自身创建的 Topic 路由等信息。
- NameServer 收到心跳包会记录时间戳，每隔 10s 扫描一次 brokerLiveTable，如果发现 120s 内没有收到心跳包，则认为 Broker 失效 --> 更新主题路由信息、将失效的Broker信息移除。
- 客户端每隔 30s 向 NameServer 更新对应的主题路由信息。



## || 消息存储



存储单元：

- Topic 可以包含一个或多个 MessageQueue，数据写入到 Topic 后，最终消息会分发到对应的 MessageQueue 中存储。



底层存储

- 在底层的文件存储方面，并不是一个 MessageQueue 对应一个文件存储的，而是一个节点对应一个总的存储文件，单个 Broker 节点下所有的队列共用一个日志数据文件（CommitLog）来存储。

- **CommitLog** 是消息主体以及元数据存储主体，每个节点只有一个，客户端写入到所有 MessageQueue 的数据，最终都会存储到这一个文件中。会进行分段存储。

- **ConsumeQueue** 是逻辑消费队列，是**消息消费的索引，不存储具体的消息数据**。

  > 引入的目的主要是提高消息消费的性能。由于 RocketMQ 是基于主题 Topic 的订阅模式，消息消费是针对主题进行的，如果要遍历 Commitlog 文件，基于 Topic 检索消息是非常低效的。
  >
  > Consumer 可根据 ConsumeQueue 来查找待消费的消息，ConsumeQueue 文件可以看成是基于 Topic 的 CommitLog 索引文件。

- **IndexFile** 是索引文件，它在文件系统中是以 HashMap 结构存储的。在 RocketMQ 中，通过 Key 或时间区间来查询消息的功能就是由它实现的。



# | 设计 



## || 主从同步



## || 主从切换





# | 特性

## || 顺序消息



## || 消息过滤



## || 消息回溯



## || 定时消息 



## || 消息重试



# | 运维



## || ACL 



## || Trace



## || 监控