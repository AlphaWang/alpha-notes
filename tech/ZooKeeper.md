

# | 基础

**单机 Standalone**

- zoo.cfg

  ```
  tikeTime=200
  dataDir=
  clientPort=2181
  ```

- zkServer.sh start

- telnet xx 2181

- srvr

**集群 Ensemble**

- 奇数

- quorum： 过半节点

- 节点过多会降低性能

- 配置

  - zoo.cfg

    ```
    tikeTime=200
    dataDir=
    clientPort=2181
    initLimit=20 # 
    syncLimit=5
    server.1=xx:2888:3888  
    server.2=yy:2888:3888
    # server.myId = ho:peerPort:leaderPort
    # peerPort: 节点间通信端口
    # leadPort: 首领选举端口					
    ```

  - myid

    ```
    # 位于dataDir目录
    1
    ```

    

**配置**

- clientPort：对客户端提供服务的端口
- dataDir：保存快照文件的目录
- dataLogDir：保存事务日志文件的目录

**命令**

- `ls -R /` 列出所有节点
- 四字命令
  - `ruok`
  - `conf`
  - `stat`
  - `dump` 查看临时节点
  - `wchc` 查看watcher
    	

**设计目标**

- 简单的数据模型：树形结构
- 集群
  - 超过一半工作即可对外服务
  - 每个机器都存储当前所有服务器状态；客户端只要连接到任意一台服务器即可
- 顺序访问
- 高性能：数据存储于“内存”



# | 原理



## || Version 保证原子性

- version 表示修改次数，类似乐观锁



## || 数据模型

**ZNode**

- 节点类型
  - 持久 Persistent
  - 临时 Ephemeral
  - 顺序 Sequential

- 节点信息

  - `czxid / mzxid` 创建/修改时对应的 zxid

  - `ctime /mtime`

  - `version` 当前节点的修改次数，实现“乐观锁”机制

  - `cversion` 子节点版本号

  - `aversion` ACL 版本号

  - `dataLength` 数据内容长度

  - `ephemeralOwner` 创建该临时节点对应的会话ID

    

**数据持久化**

- 事务日志

  - 配置 **logDir**

  - 流程 

    1. 确定是否有日志文件 --> 创建文件，命名为当前ZXID

    2. 确定日志文件是否需要扩容 --> 64M

    3. 事务序列化

    4. 生成 CheckSum

    5. 写入事务日志文件流

    6. 刷盘

       

- 数据快照

  - 配置 **dataDir**
  - 流程
    1. 确定是否需要进行快照 --> 基于已记录的事务日志数
    2. 切换快照文件
    3. 创建数据快照异步线程 -- 避免影响zk主流程
    4. 获取全量数据和会话信息 -- 获取DataTree
    5. 生成快照数据文件名 -- 当前已提交的最大ZXID
    6. 数据序列化

**数据同步**

- 概念：Learner 启动后向 Leader请求数据
- 同步方式
  - 直接差异化同步 DIFF
  - 先回滚再差异化同步 TRUNK + DIFF：当Learner中包含Leader没有的事务
  - 仅回滚同步 TRUNK
  - 全量同步 SNAP -- Learner 中没有提议



## || Watcher

**流程**

- **客户端**

  - 客户端注册 Watcher

  - 客户端将 Watcher 对象存到 WatchManager: `Map<String, Set<Watcher>> `

    > key = path
    > value = 所有监听者

- **服务端**

  - 服务端存储 ServerCnxn

    - ServerCnxn 代表一个客户端和服务端的连接

    - 存储结构

      ```java
      // watchTable: key = path
      Map<String, Set<Watcher>>
      	
      // watch2Paths: key = watcher
      Map<Watch, Set<String>>
      ```

    - 客户端并不会将 Watcher 对象真正传到服务端，否则服务端内存紧张

  - 服务端触发通知

    - Step-1. 封装 WatchedEvent。注意不包含被更新的数据！！！，需要推拉结合

      > **KeeperState**
      >
      > - SyncConnected
      > - Disconnected
      > - Expired
      > - AuthFailed
      >
      > **EventType**
      >
      > - None
      > - NodeCreated
      > - NodeDeleted
      > - NodeDataChanged
      > - NodeChildrenChanged
      >
      > **Path**

    - Step-2. 查询并删除Watcher
      
    - Step-3. process: send response (header = -1)：通过 ServerCnxn 将 Event 对象传递给客户端

- **客户端**执行回调

  - Step-1. SendThread 接收通知， 放入EventThread （NIO）
  - Step-2. 查询并删除Watcher
  - Step-3. process: 执行回调

**特性**

- 一次性
- 客户端串行执行，所以不要执行耗时操作
- 轻量
  - 推拉结合
  - 注册 watche r时只传输 “ServerCnxn”



>  Q: Curator 如何解决一次性watcher问题？





## || ACL

**Scheme**

- IP:192.168.1.1:permission
- Digest:username:sha:permission
- World:anyone:permission
- Super:username:sha:permission

**Permission**

- C, Create
- D, Delete
- R, Read
- W, Write
- A, Admin

权限扩展体系

- 实现 AuthenticationProvider

- 注册

  - 系统属性 -Dzookeeper.authProvider.x=
  - zoo.cfg: authProvider.x=

  

## || 客户端 

**通讯协议**

- 请求

  - RequestHeader

    - xid：记录客户端发起请求的先后顺序

    - type

      > - 1: OpCode.Create
      > - 2: delete
      > - 4: getData

  - Request

- 响应

  - ReplyHeader
    - xid：原值返回
    - zxid：服务器上的最新事务ID
    - err
  - Response



**ClientCnxn：网络IO**

- outgoingQueue：待发送的请求Packet队列
- pendingQueue：已发送的、等待服务端响应的Packet 队列
- SendThread: IO线程
- EventThread: 事件线程，waitingEvents队列



## || Session

**SessionID**

- 服务器myid + 时间戳



**SessionTracker**: 服务器的会话管理器

- 内存数据结构

  - sessionById `HashMap<Long, SessionImpl>`
  - sessionWithTimeout `ConcurrentHashMap<Long, Integer>`
  - sessionSets `HashMap<Long, SessionSet>`，超时时间分桶

- 分桶策略

  - 将类似的会话放在同一区块进行管理
  - 按照“下次超时时间”
  - 好处：清理时可批量处理

- 会话激活

  - 触发
    - 客户端发送任何请求时
    - sessionTimeout / 3时，发送PING请求
  - 流程
    - 心跳检测
    - 重新计算下一次超时时间
    - 迁移到新区块

- 超时检测

  - 独立线程，逐个清理

- 会话清理

  1. isClosing 设为 true
  2. 发起“会话关闭”请求
  3. 收集需要清理的临时节点
  4. 添加“节点删除”事务变更
  5. 删除临时节点
  6. 移除会话、关闭NIOServerCnxn

- 重连

  - 连接断开
  - 会话失效
  - 会话转移

  

# | 应用场景

## || 配置中心

案例

- 数据发布订阅

设计：推拉结合

- 推：Watch 事件
- 拉：客户端收到事件后，需要拉取数据

**实现**

- 数据节点

  - “持久”节点
  - `/configserver/app1/database_config`
  - 如何记录版本信息？回退版本？

- 获取

  - 客户端初始化时，从配置节点读取数据；同时注册监听

- 变更

  - Watch

  

## || 负载均衡

案例

- 域名注册、发现
- 动态DNS服务

**实现**

- “持久”节点
- 域名配置：`/DDNS/app1/service1.compnay1.com`



## || 命名服务

案例

- 全局ID生成器

实现

- “顺序”节点
- `/jobs/type1/job-00001, ...`



## || 分布式协调、通知

案例

- 任务注册
- 任务状态记录

实现

- **任务注册**
  - 任务节点
    - “持久”节点
    - `/app/tasks/copy_hot_item`
  - 子节点：任务执行者
    - “临时顺序”节点
    - `/app/tasks/copy_hot_item/instances/host-1`
    - task机器启动时，都会找到对应任务节点，在其下创建自己的临时顺序节点
- **任务热备份**
  - 小序号优先策略
  - task机器判断自己的“临时顺序节点”是否序号最小
  - 如果最小，则设置状态为 RUNNING；否则设为 STANDBY
- **热备切换**
  - 一旦 RUNNING 的机器故障，则其“临时顺序节点”被删除
  - 其他 STANDBY 机器监听到，再次按照“小序号优先”策略选出RUNING机器
- **记录执行状态**
  - 子节点：执行状态
    - “持久”节点
    - `/app/tasks/copy_hot_item/lastCommit`
  - RUNING机器定时向这个节点写入状态
  - 目的：热备切换后 共享状态



## || 机器间通信

优点

- 降低系统耦合

案例

- **心跳检测**

  - 每个客户端创建“临时子节点”
  - 根据该节点判断对应客户端是否存活

- **工作进度汇报**

  - 每个客户端创建“临时子节点”，并将任务进度写入该节点

- **系统调度**

  - AS-IS：控制台发送命令给客户端
  - TO-BE：控制台命令写入zk，再通知给客户端

  

## || 集群管理

**目的**

- AS-IS: 每个客户端部署Agent，汇报监控信息



**案例1：分布式日志收集系统**

> 日志收集系统要解决：
> 1. 变化的日志源机器
> 2. 变化的收集器机器
>
> 思路：
>
> - 注册日志收集器，非临时节点：`/logs/collectors/[host]`
>
> - 节点值为日志源机器。
>
> - 创建子节点，记录状态信息 `/logs/collectors/[host]/status`
>
> - 系统监听collectors节点，当有新收集器加入，或有收集器停止汇报，则要将之前分配给该收集器的任务进行转移。

- 目的：为每个收集器分配对应的日志源机器
- **注册收集器机器**
  - 节点：`/logs/collectors/host-N`
  - “持久节点”。不能是“临时节点”！
    - 因为子节点要存储日志源信息
    - 如果收集器挂了，节点怎么处理？
      见：收集器状态汇报
      根据最后更新时间判活；放弃监听，用主动轮询	
- **任务分发**
  - 将日志源机器分为N组，分别写入到 host-N 节点
- **收集器状态汇报**
  - 节点
    - `/logs/collectors/host-N/status`
    - “持久”节点
  - 系统根据该子节点的最后更新时间判断收集器是否存活，而不是根据“临时节点”判断状态
  - 监听
    - status 更新会非常频繁，且大部分更新通知都是无用的。
    - 所以考虑放弃“监听”，而由日志系统主动轮询	
- **动态分配**
  - 目的：有新收集器加入、或老收集器故障，则重新分配任务
  - 监听：监听host-N
  - 全局动态分配：
    - 对所有日志源机器重新分组
  - 局部动态分配
    - 故障：将当前节点任务分配到负载低的机器上
    - 新加：将负载高机器的任务分配给此新机器



**案例2：云主机管理**

- 目的：集群机器存活性监控

- 节点

  - “临时”节点
  - `/app/machine/host-N`
  - 节点值：运行状态

- 监听

  - 监控中心监听 /app/machine 节点，监听主机上下线
  - 监控中心监听 节点值变更，获取运行时信息

  

## || Master选举

利用zk强一致性，保证客户端无法重复创建已存在的节点

**目的**

- 选主，且挂了能得到通知
- AS-IS: 利用数据库主键约束，所有机器都插入相同主键的记录；但挂了不能通知

**实现**

- 节点
  - “临时”节点
  - `/master_election/2019-01-01/binding`
  - 所有客户端都注册同一名称的“临时节点”
- 监听
  - 所有客户端都监听 `/master_election/2019-01-01` “子节点变更”

> Q：这种监听会导致羊群效应？



## || 分布式锁

**排他锁**

- 节点
  - “临时”节点
  - `/exclusive_lock/lock`
- 获取锁
  - 只有一个客户端能成功创建“临时节点”
  - 其他客户端在 `/exclusive_lock` 节点上注册 “子节点变更”通知
- 释放锁
  - 临时节点被删除
  - 其他客户端重新发起获取锁流程

**共享锁**

- 节点

  - “临时顺序”节点

  - `/shared_lock/Hostname-<请求类型>-<序号>`，例如`/shared_lock/hostN-(R|W)-index`

    > 序号在R/W之间会不会重复？--> 不会

    

- 获取锁
  - 区分读写；
    - 读请求：请求类型=R；
    - 写请求：请求类型=W
  - 监听设计-1
    - 创建完后，监听/shared_lock “子节点变更”通知		
    - 问题：羊群效应
      - 所有客户端都收到事件通知
      - 但只有序号最小的客户端有意义
  - **监听设计改进**
    - 只监听比自己序号小的“相关节点”
    - R 读请求：监听前一个 “写请求”节点
    - W 写请求：监听前一个节点
  - **判断读写顺序**
    - R 读请求
      - 比自己序号小的节点都是`R`，则加锁成功，执行读取逻辑
      - 如果其中有W，则进入等待
    - W 写请求
      - 如果自己是最小节点，则加锁成功
      - 如果之前有任何W/R节点，则进入等待
- 释放锁



## || 分布式队列

**FIFO 队列**

> - 注册临时顺序节点；
> - 监听比自己小的最后一个节点。

- 节点
  - 类似“全写”的共享锁
  - `/queue_fifo/元素+序号`
  - “临时顺序”节点
- 流程
  - 创建顺序节点
  - 判断是否最小节点
    - 如果是，则处理
    - 如果不是，则等待，并监听比自己序号小的最后一个节点



**Barrier 队列**

> 等相关元素集聚之后统一处理 
>
> - 父节点`/queue_barrier`，值为需要等待的节点数目N。
> - 监听其子节点数目；
> - 统计子节点数目，如果数目小于N，则等待

- 节点

  - `/queue_barrier `
  - 值 =N，等待N个元素
  - 子节点 = 队列元素

- 流程

  - 创建顺序节点，注册监听

  - 创建完后，统计子节点个数	

    - 如果 < N，则等待事件通知
    - 如果 = N，则处理

    

## || 服务注册发现



# | 实例

## || Hadoop

**ResourceManager HA**

- 多个ResourceManager并存，但只有一个处于Active状态。
  - 有父节点 `yarn-leader-election/pseudo-yarn-rm-cluster`, RM 启动时会去竞争 Lock**临时**子节点。
  - 只有一个 RM 能竞争到，其他 RM 注册 Wather



**ResourceManager 状态存储**

- Active状态的RM会在初始化阶段读取 `/rmstore` 上的状态信息，并据此信息继续进行相应的处理。



## || HBase

RegionServer 系统容错

RootRegion 管理



## || Kafka

**Broker注册**

- `/broker/ids/[brokerId]`
- 临时节点



**Topic注册**

> 每个topic对应一个节点`/brokers/topics/[topic]`；
>
> Broker启动后，会到对应Topic节点下注册自己的ID **临时节点**，并写入Topic的分区总数，`/brokers/topics/[topic]/[BrokerId] --> 2`

- /brokers/topics/[topic]
- 与Broker对应关系 
  - /brokers/topics/[topic]/[BrokerId]
  - 临时节点
  - value = 分区数目



**Producer负载均衡**

Producer会监听
- Broker的新增与减少、
- Topic的新增与减少、
- Broker与Topic关联关系的变化



**Consumer注册**

> - 消费者启动后，注册**临时节点** `/consumers/[groupId]/ids/[consumerI]`。并将自己订阅的Topic信息写入该节点 
>
> - 每个消费者都会监听ids子节点。
> - 每个消费者都会监听Broker节点：`/brokers/ids/`

- /consumers/[groupId]/ids/[consumerId]

- 临时节点

  

**Consumer负载均衡**

> 当消费者确定了对一个分区的消费权利，则将其 ConsumerId 写入到分区**临时节点**上：
> `/consumers/[groupId]/owners/[topic]/[brokerId-partitionId] --> consumerId`

- /consumers/[groupId]/owners/[topic]/[brokerId-partitionId]
- 节点值：consumerId
- 临时节点



**消费进度offset记录**

> 消费者重启或是其他消费者重新接管消息分区的消息消费后，能够从之前的进度开始继续进行消费：`/consumers/[groupId]/offsets/[topic]/[brokerId-partitionId] --> offset`

- /consumers/[groupId]/offsets/[topic]/[brokerId-partitionId]




# | Reference

- 应用于资源分配：
  https://jack-vanlightly.com/blog/2019/2/1/building-a-simple-distributed-system-the-implementation
- 