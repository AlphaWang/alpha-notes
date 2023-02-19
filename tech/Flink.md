[toc]

# | 流式计算

## || 大数据计算模式

- **批量计算 Batch Computing**
  - MapReduce
  - Spark
  - Hive
  - Flink
  - Pig
- **流式计算 Stream Computing**
  - Storm
  - Spart Streaming
  - Flink
  - Samza
- **图计算 Graph Computing**
  - Giraph - Facebook
  - Graphx - Spark
  - Gelly - Flink
- **交互计算 Interactive Computing**
  - Presto
  - Impala
  - Druid
  - Drill



对比

|                   | Flink                                                     | Spark Streaming                          | Storm                                    | Samza                    |
| ----------------- | --------------------------------------------------------- | ---------------------------------------- | ---------------------------------------- | ------------------------ |
| **架构**          | 主从结构 - 类sp streaming<br />DataFlowGraph - 类storm    | 主从模式，每个batch都依赖主              | 主从模式，依赖zk，处理过程对主的依赖不大 | 依赖kafka                |
| **处理模式**      | Native                                                    | Micro-batch                              | Native                                   | Native                   |
| **容错**          | 基于 Chandy-Lamport distributed snapshots checkpoint 机制 | WAL 及 RDD 血统机制                      | Records ACK                              | Log-Based                |
| **处理模型&延迟** | 单条事件处理；亚秒级延迟                                  | 一个事件窗口内的所有事件；秒级延迟       | 每次传入的一个事件；亚秒级延迟           | 单条事件处理；亚秒级延迟 |
| **吞吐量**        | High                                                      | High                                     | Low                                      | High                     |
| **数据处理保证**  | Exactly Once                                              | Exactly Once                             | At Least Once (基于 record level ack)    | At Least Once            |
| **高级 API**      | 高级类库                                                  | 容易对接Spark生态组件                    | 需按照特定的Storm定义的规则编写          | 只支持 JVM 语言          |
| **易用性**        | 支持 SQL Streaming，批流一体                              | 支持 SQL Streaming，批流采用统一编程框架 | 不支持 SQL Streaming                     | 好于 Storm               |
| **部署**          | 简单，只依赖 JRE                                          | 简单，只依赖 JRE                         | 依赖 JRE 和 zk                           | 依赖 JRE                 |



## || 分布式流处理模型

- Google 论文《The Dataflow Model》：https://research.google/pubs/pub43864/ 

- Flink doc: https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/learn-flink/overview/ 

![image-20220116233209988](../img/flink/stream-dataflow.png)

- 数据从上一个 Operation 节点直接 push 到下一个 Operation 节点；
- 各节点可以分布在不同的 task 线程中运行；



**流处理演进**

- **Lambda 架构**
  - 在传统周期性批处理架构的基础上，增加一个提速层（Speed Layer），由低延迟流处理引擎驱动。
  - 数据同时发往流处理引擎和批量存储。批处理引擎周期性处理并将精确结果写入“批处理表”；流处理引擎近实时计算出近似结果并写入“提速表”。
  - 应用最终对“批处理表”、“提速表” 将精确结果与近似结果进行合并。



## || Dataflow











# | Flink 架构

## || 核心特性

- 统一数据处理组件栈：Batch, Stream, ML, Graph；
- 支持事件时间 - `Event Time`、接入时间 - `Ingestion Time`、处理时间 - `Processing Time`；
- 容错：基于轻量级分布式快照；

  > Flink is able to provide fault-tolerant, exactly-once semantics through a combination of state snapshots and stream replay.  
- 支持有状态计算；

  > The set of parallel instances of a stateful operator is effectively a sharded key-value store. Each parallel instance is responsible for handling events for a specific group of keys, and the state for those keys is kept **locally** (heap, or disk).
  >
  > https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/datastream/fault-tolerance/queryable_state/ 
- 支持高度灵活的窗口操作；
- 带 “反压” 的连续流模型；
- 基于 JVM 实现独立的内存管理；



### Stateful

**分类**

- Raw State

  > 用户自己管理，需要自己序列化，适用于自定义 Operator

- Managed State

  > Flink runtime 管理，推荐使用。

  - **Keyed State**：只能用于 KeyedStream 算子中。
    keyed state 是一种分片的键/值存储，每个 keyed state 的工作副本都保存在负责该键的 taskmanager 本地中。
    - ValueState
    - MapState
    - AppendingState
      - ListState
      - ReducingState - 加入元素后，直接计算reduce (例如计数)，减少存储量
      - AggregatingState 
  - **Operator State**：用于所有算子 
    每个 operator state 都与一个 operator 实例绑定，例如 source state = offset.
    - ListState
    - BroadcastState



Flink 提供了为 RocksDB 优化的 `MapState` 和 `ListState` 类型。 相对于 `ValueState`，更建议使用 `MapState` 和 `ListState`。

> 因为使用 RocksDBStateBackend 的情况下， `MapState` 和 `ListState` 比 `ValueState` 性能更好。 



**State 操作代码**

```java
DataStream<Event> events = env.addSource();
DataStream<Alert> alerts = events
  .keyBy(Event::sourceAddress)
  .flatMap(new StateMachineMapper());

class StateMachineMapper extends RichFlatMapFunction<Event, Alert> {
  private ValueState<String> curState;
  
  public void open(Configuration conf) {
    //load state: via RuntimeContext
    curSatate = getRuntimeContext().getState(new ValueStateDescriptor<>("state", String.class));
  }
  
  public void flatMap(Event evt, Collector<Alert> out) {
    //access state value
    String state = curState.value(); 
    if (needAlert) {
      out.collect(new Alert(evt));
    } else if (isTerminal) {
      //clear state value
      curState.clear(); 
    } else {
      //update state value
      curState.update("newState"); 
    }
  }
}
```



**State backend**

- `EmbeddedRocksDBStateBackend`：本地磁盘，慢10倍
- `HashMapStateBackend`：Jvm heap，更快.

|                     | State                   | Checkpoint             | 场景                                     |
| ------------------- | ----------------------- | ---------------------- | ---------------------------------------- |
| MemoryStateBackend  | TM内存，单个state最大5M | JM内存                 | 本地测试，JM 如果挂了就无法恢复          |
| FsStateBackend      | TM内存                  | 外部文件系统，例如HDFS | 常规状态作业                             |
| RocksDBStateBackend | TM KV数据库             | 外部文件系统，例如HDFS | 超大状态作业，rocksdb 支持增量checkpoint |

配置

- state backend: `env.setStateBackend(new FsStateBackend(...))`
- 状态保存时间：[`table.exec.state.ttl`](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/config/#table-exec-state-ttl) defines for how long the state of a key is retained without being updated before it is removed.



### Fault Tolerance

> - 官网 https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/learn-flink/fault_tolerance/ 
>   https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/ops/state/checkpoints/ 
>
> - Checkpoint & barrier: https://nightlies.apache.org/flink/flink-docs-release-1.10/internals/stream_checkpointing.html 
>
> - Lightweight Asynchronous Snapshots for Distributed Dataflows 
>
>   https://arxiv.org/pdf/1506.08603.pdf  
>
> - 状态管理 https://www.infoq.cn/article/VGKZA-S9fMBgABP71Pgh 
>
> - Checkpoint 原理 https://www.infoq.cn/article/wkgozmqqexq6xm5ejl1e
>
> - Flink Fault Tolerance Video: https://www.bilibili.com/video/BV1vf4y1x7se?p=5



**Checkpoint Storage**

问题：如果 TM 挂了，其中保存的 State 也丢失了。

解决：Flink 定期获取所有 State 的快照，并将这些快照复制到持久化的位置，例如分布式文件系统。

- FileSystemCheckpointStorage：分布式文件系统
- JobManagerCheckpointStorage：测试用

> 默认情况 **State** 保存在 TM 内存中，**Checkpoint** 保存在 JM 内存中。



**概念**

- **Snapshot**
  - 包括指向每个数据源的指针（例如，到文件或 Kafka 分区的偏移量）、以及每个作业的有状态运算符的**状态副本**
- **Checkpoint**
  - Flink 自动生成的 snapshot。可全量可增量。
- **Externalized Checkpoint**
  - checkpoint 完成时在指定的外部持久化存储保存，以便通过指定checkpoint路径手工恢复作业状态。
- **Savepoint**
  - 手工触发的全量 snapshot；用于有计划地备份以便作业停止后的恢复，例如作业升级、调整并发



**Fault Tolerance Guarantee**

- Exactly once 
  - Each event *affects the managed state* exactly once  
  - Note: This does not mean that events are processed exactly once! 
- At least once  
  - Each event *affects the managed state* of a program at least once 
- At most once 
  - All state is lost in case of a failure



**原理：异步全局一致性快照算法**

- 利用 stream **replay** + **checkpointing**

- 定期生成 Snapshot；恢复时加载最新快照

- 挑战：

  - 如何确保状态拥有 exactly-once 容错保证？

  - 如何对多个拥有本地状态的算子，产生一个全局一致的快照？

    > 思路一：时钟同步。clock skew 会导致不能保证因果一致性，可能 node-2 上的果保存到快照、但 node-1 上的因却没有。
    >
    > 思路二：全局同步。Stop-the-world. 

  - 如何在不中断运算的情况下，生成一致性快照？--> 异步

    > Lightweight Asynchronous Snapshots for Distributed Dataflows (Chandy-Lamport) 
    > https://arxiv.org/pdf/1506.08603.pdf 
    > https://www.bilibili.com/video/BV1vf4y1x7se?p=5

- 算法

  - 发起快照

    > 假设有两个进程 P1, P2, P3；Channel C12, C21, ...
    >
    > 任意进程都可发起快照。
    >
    > - P1 第一步记录自己的本地状态 (A, B)、
    > - 然后立刻向所有 output channel 发送 marker 消息、
    > - 之后开始记录所有 input channel 消息：C11, C21, C31 = recording.
    >
    > ![image-20230214132150385](../img/flink/chandy-lamport-trigger-1.png)

  - 分布式执行快照
    > P3 接收到来自 C13 的 marker 后，如果是P3看到的第一个marker消息，则：
    >
    > - 记录本地状态
    > - 标记 C13 = [empty] ——后续所有来自 C13 的消息不再包含进本次快照
    > - 向所有 output channel 发送 marker 消息
    > - 开始记录除 C13 之外的 input channel 消息 = recording
    >
    > ![image-20230214132433931](../img/flink/chandy-lamport-exe-1.png)
    >
    > 如果此前已经收过 marker 消息（例如 P1 接收到 C31发来的marker）
    >
    > - 停止记录 C31 的消息，C31 = [empty] ——后续所有来自 C31 的消息不再包含进本次快照
    > - 同时将此前记录的所有 C31 消息作为 C31 在本次快照中的最终状态。 
    >
    > ![image-20230214132700405](../img/flink/chandy-lamport-exe-2.png)
    >
    > 
    >
    > 之后 P2 接收到 C32，是第一个marker：本地快照、关闭C32=empty、向其他channel发送marker
    >
    > ![image-20230214132821714](../img/flink/chandy-lamport-exe-3.png)
    >
    > 之后 P2 接收到 C12，不是第一个 marker: 关闭所有 input channel、并记录channel状态
    >
    > ![image-20230214133007648](../img/flink/chandy-lamport-exe-4.png)
    >
    > 之后 P1 接收到 C21，不是第一个marker：关闭所有 input channel、记录channel状态（C11=C, C21=H->D）
    >
    > ![image-20230214133407129](../img/flink/chandy-lamport-exe-5.png)
    >
    > 最后 P3 收到 C23，不是第一个marker：关闭所有 input channel、记录 channel 状态（C33=J）
    >
    > ![image-20230214133618919](../img/flink/chandy-lamport-exe-6.png)

  - 终止快照

    > 终止快照的两个条件：
    >
    > - 所有进程都已经接收到 marker 消息，并记录在本地快照；
    >
    > - 所有进程都从他的 n-1 个input channel 收到了marker 消息，并记录了管道状态；
    >
    >   
    >
    > 此后 快照收集器就开始收集每一个部分的快照形成全局一致性快照。如上例，每一个进程的所有 input channel 都已关闭。
    >
    > ![image-20230214133754989](../img/flink/chandy-lamport-close-1.png)



checkpoint vs. state

- State 是 checkpoint 所做的持久化备份的数据



**流程：Checkpointing**

> 官方：https://nightlies.apache.org/flink/flink-docs-release-1.10/internals/stream_checkpointing.html
>
> 图示：https://juejin.cn/post/6951628600428724254 
>
> 分解图示：https://www.infoq.cn/article/wkgozmqqexq6xm5ejl1e

![](../img/flink/checkpoint-flow.svg)



- **插入屏障**：JM CheckpointCoordinator定期将 checkpoint barrier 插入到数据源从 Source 算子的数据流中
  ![](../img/flink/checkpoint_barriers.svg)

  - 从数据源出来的数据流被 checkpoint barrier 分成了一个一个的段落、Stage
  - The point where the barriers for snapshot *n* are injected (let’s call it *Sn*) is the position in the source stream up to which the snapshot covers the data. For example, in Apache Kafka, this position would be the last record’s offset in the partition. 
  - This position *Sn* is reported to the *checkpoint coordinator* (Flink’s JobManager).

- **Source 算子**：Source 收到每条消息，记录当前状态(offset)；当收到 barrier后，将状态记录到持久化存储表 `checkpoint-data: source-1 = offset`

  - 即执行 *状态快照*。

- **中间算子：对齐屏障**：中间算子*等待*所有输入流的 Sn 屏障（align），之后向输出流发送屏障n。并将状态记录到表 `checkpoint-data: operator-1 = state`
  ![](../img/flink/checkpoint_aligning.svg)

  - 对齐会引入 Latency、反压，可配置忽略对齐。缺点是恢复后会重复处理。

  - 对于单个流，不用对齐

  - 对齐 = Exact-Once, 不对齐 = At-Least-Once. 

    > ——注意指的是计算过程的精确一次，每条event会且只会对 state 产生一次影响；如果不对齐，多处理的event会被回放。
    >
    > 端到端精确一直还需要 source / sink 的支持。

- **状态快照**：算子在接收到输入流的所有 barrier 之后、将 barrier 发送给输出流之前，将算子的状态进行快照(snaoshot the state)，存储到 state backend。  

  - 完成备份后，会将备份数据的地址 (state handle) 通知给 Checkpoint Coordinator
  - 默认同步进行状态快照，会停止处理输入数据。
  - 可配置为*异步快照*，前提是 further modifications to the operator state do not affect that state object，例如用 copy-on-write. 

- **Sink 算子**：Sink 收到所有输入流的 barrier n 之后，记录到表 `checkpoint-data: sink-1 = offset`

  - 向 JM  *checkpoint coordinator* 确认 Sn。所有 sink 都确认后，表示已完成。
  - Sn 完成后 Job 不再请求Sn之前的数据：Once snapshot n has been completed, the job will never again ask the source for records from before Sn, since at that point these records (and their descendant records) will have passed through the entire data flow topology.

- **完成：**Checkpoint Coordinator 收集齐所有 task 的 state handle 之后，认为这次 checkpoint 全局完成，向持久化存储再备份一个 `completed checkpoint meta` 文件。



**流程：Recovery**

- 选择前一次完成的 checkpoint k，然后重新部署整个分布式 dataflow，将随 k 一起保存的状态 分配给各个算子。
  - Flink 默认只保留最新一个checkpoint，可配置flink-conf.yaml:  `state.checkpoints.num-retained=1`
- 即 source 开始从 S<sub>k</sub> 处读取消息。
  - 因为本地的state尚未被snapshot持久化，故障后会丢失；所以需要回滚到上一个checkpoint，重建该state (例如word count)
  - 必要条件：**Source 要支持数据重新发送！**



配置

- **Restart Strategy**: decide whether and when the failed/affected tasks can be restarted
  - `none, off, disable`: No restart strategy.
  - `fixeddelay, fixed-delay`: Fixed delay restart strategy. 已固定延迟、重启固定次数，超过则失败
  - `failurerate, failure-rate`: Failure rate restart strategy. 重启超过失败频率时，则失败
- **Failover Strategy**: decide which tasks should be restarted to recover the job
  - 







### Backpressure

- TM 之间的背压
  ![image-20230107165135485](../img/flink/backpressure-tms.png)

- TM 内部 Task 的背压
  ![image-20230107165223975](../img/flink/backpressure-task.png)

- 1.5 之后基于 Credit 的背压原理
  ![image-20230107165351908](../img/flink/backpressure-credit.png)

  - InputChannel 往下游发数据时，会告知 credit；下游返回剩余 local buffer 大小；上游只发送合适的数据大小。

    



## || 集群架构

> - Flink Runtime https://www.infoq.cn/article/RWTM9o0SHHV3Xr8o8giT 
> - Flink Runtime arch video: https://www.bilibili.com/video/BV1vf4y1x7se?p=4
>
> 

组件

- **JobManager**
  - 管理节点，每个集群至少一个；
  - 管理集群计算资源、Job调度、checkpoint协调
- **TaskManager**
  - 每个集群有多个 TM；
  - 负责计算资源提供
- **Client**
  - 本地执行 main，解析 JobGraph 对象、提交到 JobManager 运行

![image-20220116233910154](../img/flink/flink-components.png)



![The processes involved in executing a Flink dataflow](../img/flink/arch-components.svg)



![image-20221007224223282](../img/flink/flink-components3.png)

### JobManager

三大子组件

- **Dispatcher** 
  - Rest Interface、Web UI
  - 处理 job 提交，为每个 Job 启动 JobMaster
- **JobMaster**
  - 每个 Job 对应一个 JobMaster；二者生命周期一致。
  - **作业生命周期管理**：将 job 分配到处理槽、监控 task 执行；
    ![image-20230212230608767](../img/flink/job-lifecycle.png)
  - **任务调度**
    ![image-20230212231335999](../img/flink/task-scheduling-state.png)
  - 出错恢复
  - CheckpointCoordinator：分布式状态快照
- **Resource Manager**
  - **指派 TaskManager 槽**：当 JM 申请 `TaskManager 处理槽`时，`ResourceManager` 会指示一个拥有空闲处理槽的 TaskManager 将其处理槽提供给 JobManager。
  - **申请创建 TaskManager**：如果当前处理槽无法满足 JM 的请求，则`ResourceManager` 与资源提供者通信，让它们提供额外容器来启动更多 TM 进程。
  - 其中 SlotManager 维护与 TaskManager 之间的心跳。

![image-20220117115327190](../img/flink/flink-components2.png)

流程

- **JobGraph --> ExecutionGraph**：JM 将 JobGraph 转化成物理 Dataflow 图（ExecutionGraph），其中包含哪些可以并行执行的任务。
- **申请资源**：JM 从 `ResourceManager` 申请执行任务的必要资源，即 `TaskManager 处理槽`。
- **任务分发**：一旦收到足够多的 slot，则将 ExecutionGraph 中的任务分发给 `TaskManager` 来执行。
- **协调**：在执行过程中，还负责所有需要集中协调的操作，例如创建检查点。



要点：**协调**

- Checkpoint 协调
- JobGraph --> Execution Graph
- Task 部署与调度
- RPC 通信（Actor System）
- 集群资源管理（Resource Manager）
- TaskManager 注册与管理

![image-20220116234001575](../img/flink/flink-components-jobmanager.png)





### TaskManager

流程

- TM 启动后，向 `ResourceManager` 注册其处理槽。
- 当接收到 ResourceManager 指示时，会向 RM 提供一个或多个处理槽。
- 之后 JM 就可以向处理槽中分配任务。



要点

- Task Execution 

  - 任务在一个独占的线程中执行

  - 流程：Input Gate --> Operator Chain --> Result Partition

- Task slot 共享

  - 同一个共享组 SlotSharingGroup 中的不同类型（JobVertex）的任务，可以在同一个 slot 中运行

  - 降低数据交换开销

  - 负载均衡

- Network Manager 

  > 基于 Netty，实现 TM 与TM 之间数据交互

- Shuffle Environment 管理

- RPC 通信（Actor System）

- Heartbeat with JM (RM): 存活、状态、资源消耗信息

- TM 之间的 Data Exchange：数据流动到下一个节点

- Memory Management

- Register to RM

- Offer Slots to JobManager

  - 每个slot是一个线程


![image-20220116234053630](../img/flink/flink-components-taskmanager.png)



### Client

- 执行 main 方法；
- JobGraph generate
- Execution Environment 管理
- Job 提交与运行
- Dependency Jar Ship
- RPC with JobManager
- 集群部署

![image-20220114235349214](../img/flink/flink-components-client.png)





### JobGraph

- 通过有向无环图，表达用户程序

- 是不同接口程序的抽象表达：方便与 JobManager 通信

- 是客户端和集群之间 Job 描述载体

  



![image-20220116124630533](../img/flink/flink-components-jobgraph1.png)

StreamGraph --> JobGraph

- StreamGraph 只描述转换的大概逻辑：Source - Map() - keyby() - Sink
- JobGraph 根据算子并行度拆解、形成 DAG
  - JobGraph - 逻辑图
  - ExecutionGraph - 执行图 




![image-20220116124719160](../img/flink/flink-components-jobgraph2.png)



## || 集群部署模式



**Session Mode**

- 定义	
  - 共享 JobManager 和 TaskManager；
  - 所有提交的 Job 都在一个 Runtime 中运行；
  - JobManager 生命周期不受 Job 影响，会长期运行；
- 优点
  - 资源充分共享、提高资源利用率
  - Job 在 Flink Session 集群中管理，运维简单；
- 缺点
  - 资源隔离相对较差；
  - 非 Native 类型部署：TM 不易扩展，Slot 计算资源伸缩性较差；

![image-20220116160637955](../img/flink/flink-deploy-session.png)



**Per-Job Mode**

- 定义
  - 单个 Job 独占 JobManager 和 TaskManager；
  - 每个 Job 单独启动一个 Runtime；TM 中的 slot 资源根据 Job 指定；
  - JobManager 生命周期与 Job 生命周期绑定；
- 优点
  - Job 直接资源隔离充分；
  - 资源根据 Job 需要进行申请，TM Slot 数量可以不同；
- 缺点
  - 资源相对浪费，JobManager 需要消耗资源；
  - Job 管理完全交给 ClusterManagement，管理复杂；

![image-20220116161706250](../img/flink/flink-deploy-perjob.png)



**Application Mode**

- 定义
  - Application的 main() 运行在 Cluster 上，而不是客户端；客户端无需上传dependency，释放客户端压力。
  - 每个 Application 对应一个 Runtime，Application 中可以包含多个 Job；
- 优点
  - 降低带宽消耗和客户端负载；
  - Application 之间资源隔离；Application 中实现资源共享；
- 缺点
  - 仅支持 Yarn / K8S；

![image-20220116162916232](../img/flink/flink-deploy-application.png)







# | DataStream API

编码

- 获取一个 Execution Env
- 加载、创建初始数据
- 转换
- 指定结果存储位置
- 触发 execution

Sample: 

```java
public class WindowWordCount {

    public static void main(String[] args) throws Exception {
        // 1. 设置运行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 2. 读取源数据、执行转换操作
        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(value -> value.f0)
                .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
                .sum(1);
        // 3. 默认 sink 操作
        dataStream.print();
        // 4. 执行 Flink 应用程序
        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }
}
```





## || Execution Env

StreamExecutionEnvironment 功能

- TimeCharacteristic 管理
- Transformation 存储与管理
- StreamGraph 创建和获取
- CacheFile 注册与管理
- 任务提交与运行
- 重启策略管理
- StateBackend 管理
- Checkpoint 管理
- 序列化器管理
- 类型和序列化注册
- DataStream 数据源创建
  - 基本数据源接口 - 集合、Socket、File
  - 数据源连接器 - Kafka, ES
  - Custom DataSource



## || 操作

DataStream 转换操作

- 基于单条记录：
  - `filter`, `map`, `flatmap`
- 基于窗口：
  - NonKeyed: `timeWindowAll`, `countWindowAll`, `windowAll`
  - Keyed: `timeWindow`, `countWindow`,  `window`
- 合并多条流：
  - NonKeyed: `union`, `join`, `connect`
  - Keyed: `Interval join`
- 拆分单条流：`split`, `keyBy`

![image-20220116233344476](../img/flink/flink-datastream-operators.png)

## || Stateful

- 为什么流式处理需要 Stateful? 
  - 所有 DataStream function 都可以是 stateful: filter, map, flatmap, ...
  - State 的存储：on-heap, off-heap, local-disk-backed storage.
- 存储的内容：Keyed State
  - ValueState<T>
  - ListState<T>
  - MapState<UK, UV>
  - ReducingState<T>
  - AggregatingState<IN, OUT>
- 何时被 GC
  - Flink 默认永久保留 state；
  - 可以在 ProcessFunction 的同时使用 Timer 来清理状态；
  - 使用 `StateTtlConfig` 来配置清理策略。
- State Backend
  - RocksDBStateBackend
  - FsStateBackend
  - MemoryStateBackend



## || Data Type

- 基本类型：String, Long, Integer, Boolean, ... Array

- 组合类型：

  - Row：常用于 Table / SQL API

    ```java
    Row person = Row.of("Alpha", 30);
    String name = person.get(0);
    ```

  - Tuples

    ```java
    Tuple2<String, Integer> person = new Tuple2<>("Alpha", 30);
    String name = person.get(0);
    ```

  - POJOs

- 数据序列化

  - 因为数据会在不同TM之前传输，所以需要序列化、反序列化。
  - 或自定义序列化器：`env.getConfig().registerTypeWithKyroSerializer(Type.class, Serializer.class)`
  - 禁止 fallback to kyro，减少消耗: `env.getConfig().disableGenericTypes();`





# | 时间

**时间设置：**

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```



## || 四种时间

![image-20220116233422906](../img/flink/flink-time.png)

- **Event Time**

  - 事件发生的时间

  - 可以处理乱序数据


- Storage Time
  - 例如写入主题的时间

- Ingestion Time
  - SourceOperator 消费到的时间

- **Processing Time**

  - 处理时的机器本地时间

  - 处理过程最小延迟




## || Watermark

> https://nightlies.apache.org/flink/flink-docs-release-1.9/dev/event_time.html#event-time-and-watermarks 
>
> In order to handle out-of-order events and distinguish between on-time and late events in streaming, we need to extract timestamps from events and make some kind of progress in time (so-called watermarks).
>
> - 一旦收到 wm，则可以认为该 wm 之前的所有事件都已被处理。
> - watermark defines when to stop waiting for earlier events.





概念

- Watermark 本身也属于特殊的事件；用于标记 Event-Time 的前进过程；
- Watermark 跟随 DataStream Event-Time 变动，并自身携带 TimeStamp；
- Watermark 用于表明所有较早的时间已经（可能）达到；

更新时机

- 每当有新的最大时间戳事件出现时，则产生新的 Watermark；



并行中的 Watermark 

- **Source Operator 产生 watermark**，下发给下游 Operator
- 每个 Operator 根据 watermark 对 “自己的时钟” 进行更新、并将 watermark 发送给下游算子。
- 默认情况，多个input的wm是其中最小的wm，保证数据不丢。
  - 例如消费多个 partition时，发到下游的 `wm = min(max hwm)`
  - 如果某个partition没有数据，则无法计算出 min hwm，阻塞下游。——解决办法：标记为 **Idle Source**




![image-20220116233510102](../img/flink/flink-watermark-flow.png)

**Watermark & Window**

- Watermark = Max EventTime - Late Threshold；Late Threshold 越高，数据处理延时越高；
- 启发式更新；
- 解决一定范围内的乱序事件；
- 窗口触发条件：`Current Watermark > Window EndTime`
- Watermark 的主要目的是告诉窗口不再会有比当前 Watermark 更晚的数据达到。——原理：Trigger event time *timers*.



![image-20220116234955591](../img/flink/flink-watermark-window.png)



**Watermark 生成**

API：同时指定 timestamp & watermark 

- assignTimestampsAndWatermarks 指定策略
  ```java
  //方式一：FlinkKafkaConsumer.assignTimestampsAndWatermarks
  //Pre-partition watermarking.
  DataStream<MyType> stream = env.addSource(
    new FlinkKafkaConsumer<>().assignTimestampsAndWatermarks(WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(20))));
  
  //方式二：DataStream.assignTimestampsAndWatermarks
  //Normal datastream watermarking.
  env.addSource(new FlinkKafkaConsumer<>())
     ..assignTimestampsAndWatermarks()
  ```

  

- SourceOperator 指定固定值

  ```java
  collect(T element);
  collectWithTimestamp(T element, long timestamp);
  emitWatermark(Watermark wm);
  
  ```
  
  

- Watermark 是 StreamElement的子类，存储一个 timestamp
  ```java
  public final class Watermark extends StreamElement {
    priate final long timestamp;
  }
  
  public final class StreamRecord<T> extends StreamElement {
    private T value; //Watermark 不包含value
    private long timestamp;
  }
  ```

  

两种类型

- **Periodic Watermark**

  - 常用。based on a timer，间隔固定的时间插入 wm。

  - 原理：

    ```java
    public class BoundedOutOfOrdernessGenerator implements WatermarkGenerator<MyEvent> {
      private final long maxOutOfOrderness = 3500; // 3.5 seconds
      private long currentMaxTimestamp;
      
      @Override
      public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        currentMaxTimestamp = Math.max(currentMaxTimestamp, eventTimestamp); 
      }
      
    	@Override
    	public void onPeriodicEmit(WatermarkOutput output) {
    		// emit the watermark as current highest timestamp minus the out-of-orderness bound
    		output.emitWatermark(new Watermark(currentMaxTimestamp - maxOutOfOrderness - 1)); 
      }
    }
    ```

    

- **Punctuated Watermark**

  - Based on something in the event stream.

  - 原理：

    ```java
    public class PunctuatedAssigner implements WatermarkGenerator<MyEvent> {
    	@Override
    	public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
    		if (event.hasWatermarkMarker()) {
    			output.emitWatermark(new Watermark(event.getWatermarkTimestamp())); 
        }
    	}
      
    	@Override
    	public void onPeriodicEmit(WatermarkOutput output) {
    		// don't need to do anything because we emit in reaction to events above
    	} 
    }
    ```

    

**Watermark 选择：Latency vs. Completeness**

- Longer delay: 越谨慎 越不会有数据丢失，但延时会增加。
- Shorter delay: 如果希望更实时，则需容忍数据丢失。



## || Late Event

迟到事件

> 例：
>
> - 插入 Watermark = ts - 6
>
> - W(15) 表示后续不会再出现 < 15 的事件。
>
> ![image-20230106114346387](../img/flink/watermark-example.png)



- 但实际W(15)之后还是可能出现 < 15 的事件，例如6；Late Event 即是指比当前 Watermark 更小的时间戳事件。

- **如何处理 Late Event**

  - 默认忽略

  - **Side Output**：发送到另一个stream 
    https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/learn-flink/event_driven/#side-outputs 

  ```java
  OutputTag<Event> lateTag = new OutputTag<Event>("late"){};
  
  SingleOutputStreamOperator<Event> result = stream.
      .keyBy(...)
      .window(...)
      //sideOutputLateData
      .sideOutputLateData(lateTag)
      .process(...);  
  DataStream<Event> lateStream = result.getSideOutput(lateTag);
  
  //or
  SingleOutputStreamOperator<Event> mainDataStrem = stream.process(
  	new ProcessFunction<Event, Event>() {
      @Override
      public void processElement(Event v, Context ctx, Collector<Event> out) {
        //emit data to regular output
        out.collect(v);
        //emit data to side output
        ctx.output(outputTag, v);
      }
    });
  ```

  

  - **allowedLateness**：specify an interval of *allowed lateness* during which the late events will continue to be assigned to the appropriate window(s)
    ——收到 watermark 之后在等 N 个 timestamp.
    —— N 即是 **Lateness**

    ![image-20230106121102018](../img/flink/watermark-lateness.png)

  ```java
  stream.
      .keyBy(...)
      .window(...)
      .allowedLateness(Time.seconds(10)) //为何不把watermark设长10s？
      .process(...);
  ```

​		

## || Window

- https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/datastream/operators/windows/ 



作用：

- 无界 --> 有界

流程：

![image-20220118000234100](../img/flink/flink-window-flow.png)

- `Window Assigners`: assign events to windows (creating new window objects as necessary),  
- `Window Functions`:  applied to the events assigned to a window.
- `Window Triggers`: determine when to call the window function.
- `Window Evictors`: remove elements collected in a window.



**1. Window Assigner**

![image-20220116233806080](../img/flink/time-window.png)

- 用法
  ```java
  DataStream<T> input = ...;
  // tumbling event-time windows
  input.keyBy(<key selector>)
    //or TumblingProcessingTimeWindows
    .window(TumblingEventTimeWindows.of(Time.seconds(5))) 
    .<windowed transformation>(<window function>);
  ```

  

- **Session Window 会话窗口** 

  - 例：page views per session. 
  - Unaligned, variable-length
    ![image-20230106202543874](../img/flink/agg-session-window.png)

- **Sliding Time Window 滑动窗口 （HOP?）**

  - 例：page views per minute computed every 10 seconds.

  - 时间窗口大小固定、可以重叠；Aligned, fixed-length, overlapping

    ![flink-windows-sliding](../img/flink/flink-windows-sliding.svg)

- **Tumbling Time Window 滚动窗口**

  - 将每条记录分配到特定的时间窗口；是特殊的滑动窗口：Window size == Window slide
  - 例：page views per minute.；
  - 时间窗口大小固定、不重叠，Aligned, fixed-length, non-overlapping

  ![flink-windows-tumbling](../img/flink/flink-windows-tumbling.svg)

- **Global Window**

  - 用于自己指定window策略
    ```java
    stream.keyBy(...)
      .window(GlobalWindows.create())
      .trigger(new MyCustomTrigger()) //何时启动 Window Function 来处理窗口中的数据
      .evictor(new MyCustomEvictor()) //剔除 window 中不需要的数据。
      .process(...)
    ```

  

- **Count Window**

- **CUMULATE Window** 

  - 时间窗口大小**不固定**、可以重叠
  - The `CUMULATE` function assigns elements to windows that cover rows within an initial interval of step size and expand to one more step size (keep window start fixed) every step until the max window size. 
  - For example, you could have a cumulating window for 1 hour **step** and 1 day **max size**, and you will get windows: `[00:00, 01:00)`, `[00:00, 02:00)`, `[00:00, 03:00)`, …, `[00:00, 24:00)` for every day.

  ![flink-windows-cumulating](../img/flink/flink-windows-cumulating.png)






**2. Window Trigger**

- 作用：决定何时启动 Window Function 来处理窗口中的数据、何时将窗口内的数据清理。

| Window Trigger                  | 触发频率 | 功能                                                         |
| ------------------------------- | -------- | ------------------------------------------------------------ |
| ProcessingTimeTrigger           | 一次触发 | 基于 ProcessingTime，当机器时间大于窗口结束时间时触发        |
| EventTimeTrigger                | 一次触发 | 基于 EventTime，当 Watermark 大于窗口结束时间时触发          |
| ContinuousProcessingTimeTrigger | 多次触发 | 基于 ProcessingTime，固定时间间隔触发                        |
| ContinuousEventTimeTrigger      | 多次触发 | 基于 EventTime，固定时间间隔触发                             |
| CountTrigger                    | 多次触发 | 基于元素固定条数触发                                         |
| DeltaTrigger                    | 多次触发 | 基于当前元素与上次触发的元素做delta计算，超过指定threshold触发 |
| PuringTrigger                   |          | 对 Trigger的封装，用于触发后额外清理中间状态数据             |



**3. Window Evictor**

- 作用：

  - 剔除 window 中不需要的数据。

  - 可用于 Window Function 之前，或之后。


| Window Evictor | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| CountEvictor   | 保留一定数目的元素，多余的元素按照从前到后顺序清理           |
| TimeEvictor    | 保留一个时间段的元素，早于该时间段的元素会被清理             |
| DeltaEvictor   | 窗口计算时，最近一条元素和其他元素做 Delta 计算，仅保留 Delta 在指定Threshold 内的元素 |





**4. Window Function**

3 种：reduce, aggregate, process

1. as a batch, using a `ProcessWindowFunction` that will be passed an `Iterable` with the window’s contents;
2. incrementally, with a `ReduceFunction` or an `AggregateFunction` that is called as each event is assigned to the window;
3. or with a combination of the two, wherein the pre-aggregated results of a `ReduceFunction` or an `AggregateFunction` are supplied to a `ProcessWindowFunction` when the window is triggered.



- **全量元素 Function** 

  - `ProcessWindowFunction.java`：性能较差

    ```java
    //ProcessWindowFunction: 计算传感器的最大值
    DataStream<SensorReading> input = ...
    input
        .keyBy(x -> x.key)
        .window(TumblingEventTimeWindows.of(Time.minutes(1)))
        .process(new MyWastefulMax());
    
    public static class MyWastefulMax extends ProcessWindowFunction<
            SensorReading,                  // input type
            Tuple3<String, Long, Integer>,  // output type
            String,                         // key type
            TimeWindow> {                   // window type
        
        @Override
        public void process(
                String key,
                Context context, //包含信息： window(), windowState(), globalState(), currentWatermark(), currentProcessingTime()
                Iterable<SensorReading> events,
                Collector<Tuple3<String, Long, Integer>> out) {
    
            int max = 0;
            for (SensorReading event : events) {
                max = Math.max(event.value, max);
            }
            out.collect(Tuple3.of(key, context.window().getEnd(), max));
        }
    }
    ```



- **增量元素 Function**

  - **reduce()** - `ReduceFunction`：类似 map reduce，累积

    ```java
    // ReduceFunction：累加
    DataStream<Tuple2<String, Long>> input = ...; 
    input
      .keyBy(<key selector>)
      .window(<window assigner>)
      .reduce(new ReduceFunction<Tuple2<String, Long>> {
        public Tuple2<String, Long> reduce(Tuple2<String, Long> v1, Tuple2<String, Long> v2) {
          return new Tuple2<>(v1.f0, v1.f1 + v2.f1); 
        } 
      });
    ```

    

  - **aggregate()** - `AggregateFunction`：

    ```java
    // AggregateFunction: 例如统计平均数
    DataStream<Tuple2<String, Long>> input = ...; 
    input
      .keyBy(<key selector>) 
      .window(<window assigner>) 
      .aggregate(new AverageAggregate());
    
    private static class AverageAggregate implements AggregateFunction<Tuple2<String, Long>, Tuple2<Long, Long>, Double> {
      // createAccumulator: 创建累积器，sum & count
      @Override
      public Tuple2<Long, Long> createAccumulator() { 
        return new Tuple2<>(0L, 0L);
    	}
      
      // 累加时：增加 sum & count
    	@Override
    	public Tuple2<Long, Long> add(Tuple2<String, Long> value, Tuple2<Long, Long> accumulator) {
    		return new Tuple2<>(accumulator.f0 + value.f1, accumulator.f1 + 1L); 
      }
      
      // 结果：sum/count
    	@Override
    	public Double getResult(Tuple2<Long, Long> accumulator) { 
        return ((double) accumulator.f0) / accumulator.f1;
    	}
      
      // 非核心：用于并发合并
      @Override
      public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) { 
        return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
      } 
    }
    ```
  
    
  
  - `FoldFunction`



- 内置 aggregations

  - sum(key)

  - min(key)

  - max(key)

    ```java
    DataStream<Tuple2<String, Integer>> counts = tokenized 
      .keyBy(value -> value.f0) 
      .window(TumblingEventTimeWindows.of(Time.seconds(5))) 
      // group by the tuple field "0" and sum up tuple field "1" 
      .sum(1); // 基于第二个字段
    ```



Q: 能否集成全量+增量的优点？





## || Join

e.g. 每个用户的点击 **JOIN** 该用户最近十分钟的浏览



两种合并类型：

- **Window Join**

  ![image-20220118111953470](../img/flink/flink-stream-join-windowjoin.png)

  ```java
  DataStream<Integer> orangeStream = ...
  DataStream<Integer> greenStream = ...
  orangeStream.join(greenStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>) 
    .window(TumblingEventTimeWindows.of(Time.milliseconds(2))) 
    .apply(new JoinFunction<Integer, Integer, String> () {
      @Override
      public String join(Integer first, Integer second) {
        return first + "," + second; 
      }
  });
  ```

  

- **Interval Join**
  `orangeElem.ts + lowerBound <= greenElem.ts <= orangeElem.ts + upperBound`
  ![image-20220118113025447](../img/flink/flink-stream-join-intervaljoin.png)

  ```java
  KeyedStream<Tuple2<String, Integer>, String> streamOne = ...
    .assignTimestampsAndWatermarks(new AscendingTuple2TimestampExtractor()) 
    .keyBy(new Tuple2KeyExtractor());
  KeyedStream<Tuple2<String, Integer>, String> streamTwo = ... 
    .assignTimestampsAndWatermarks(new AscendingTuple2TimestampExtractor()) 
    .keyBy(new Tuple2KeyExtractor());
  
  streamOne.intervalJoin(streamTwo)
    .between(Time.milliseconds(0), Time.milliseconds(10)) // 定义上下界
    .process(new ProcessJoinFunction<Tuple2<String, Integer>, Tuple2<String, Integer>, String>() {
      
      @Override
      public void processElement(
        Tuple2<String, Integer> left,
        Tuple2<String, Integer> right, 
        Context ctx, 
        Collector<String> out) throws Exception { 
        
        out.collect(left + ":" + right);
      }
    }).addSink(new ResultSink());
  ```





Q: Join 操作中的watermark 如何更新？对于不同输入流中的 watermark 如何选择？





# | Table API

> SQL end-to-end demo: https://flink.apache.org/2020/07/28/flink-sql-demo-building-e2e-streaming-application.html 

程序结构

- 构造 TableEnvironment
- Connect() 创建 Temporary Table
- tableEnv.from().select()  /  tableEnv.sqlQuery()
- 输出 tableEnv.executeInsert()
- 调用 tableEnv.execute()



## || Table 创建

**DataStream 转为 Table**

- tableEnv.fromDataStream
- tableEnv.createTemporaryView

- 每条记录相当于是对 table 的 INSERT，类似 CDC

```java
//Convert the DataStream into a Table with default fields "f0", "f1" 
Table table1 = tableEnv.fromDataStream(stream);

//Convert the DataStream into a Table with fields "myLong", "myString" 
Table table2 = tableEnv.fromDataStream(stream, $("myLong"), $("myString"));

//逻辑表 register the DataStream as View "myTable" with fields "f0", "f1" 
tableEnv.createTemporaryView("myTable", stream);
// register the DataStream as View "myTable2" with fields "myLong", "myString" 
tableEnv.createTemporaryView("myTable2", stream, $("myLong"), $("myString"));
```



**Table 转为 DataStream**

- tableEnv.toAppendStream

```java
// convert the Table into an append DataStream of Row by specifying the class 
DataStream<Row> dsRow = tableEnv.toAppendStream(table, Row.class);

// convert the Table into an append DataStream of Tuple2<String, Integer> 
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>( Types.STRING(), Types.INT()); DataStream<Tuple2<String, Integer>> dsTuple = tableEnv.toAppendStream(table, tupleType);

// convert the Table into a retract DataStream of Row.
// A retract stream of type X is a DataStream<Tuple2<Boolean, X>>.
// The boolean field indicates the type of the change.
// True is INSERT, false is DELETE.
DataStream<Tuple2<Boolean, Row>> retractStream = tableEnv.toRetractStream(table, Row.class);
```



**Table Connector**

```java
tableEnvironment
  .connect(...) // Table Connector Eg:Kafka 
  .withFormat(...) // Format Type Eg:JSON
  .withSchema(...) // Table Schem
  .inAppendMode() // update mode 
  .createTemporaryTable(“MyTable”) // Register Table
```





## || Dynamic Table

定义

- 动态表：基于无界序列，dynamic tables change over time. 
- Querying dynamic tables yields a `Continuous Query` 
  - A continuous query never terminates and produces a dynamic table as result.
  - 持续查询的结果会不断更新。



**原理**

- `物化视图 Materialized View`：缓存查询结果；基表修改时，物化视图将过期。
- 基于动态表查询
  - 查询动态表将生成一个连续查询 ;
  - 一个连续查询永远不会终止，结果会生成一个动态表；查询不断更新其(动态)结果表，以反映其(动态)输入表上的更改;
  - 动态表上的连续查询非常类似于定义物化视图的查询;
  - 连续查询的结果在语义上总是等价于以批处理模式在输入表快照上执行的相同查询的结果;



**类型**

- **Source tables** are data sources. 

  - Flink SQL 必须至少有一个 source table.

- **Side tables** are lookup tables. 

  - 创建时指定 dimension identifier `PERIOD FOR SYSTEM_TIME` 

    ```sql
    CREATE TABLE Behavior (
        docId INT,
        name VARCHAR,
        price INT,
        PERIOD FOR SYSTEM_TIME
    ) WITH (
        'connector.type'='couchbase',
        'connector.default-values'='0,ANN,23'
    )
    ```

  - 支持 cache

  - Retry Handler: 查询 external data store 失败时会重试

- **Sink tables** are data sinks.

  - 输出



### Table to Stream

https://nightlies.apache.org/flink/flink-docs-release-1.11/dev/table/streaming/dynamic_tables.html#table-to-stream-conversion 



- Convert to **Retract** stream
  - 
- Convert to **Upsert** stream
  - 

-  **append-only 流**：INSERT
  ![image-20220119204144268](../img/flink/flink-dynamictable-stream-insert.png)
  
- **Retract 流**：INSERT + DELETE
  >INSERT --> add
  >
  >DELETE --> retract
  >
  >UPDATE --> retract & add
  
  - UPDATE: 先 delete 再 insert，实现 update 效果
    ![image-20220119204247811](/Users/zhongxwang/dev/git/alpha/alpha-notes/img/flink/flink-dynamictable-stream-insertdelete.png)
  
- **Upsert 流**：UPSERT + DELETE

  > INSERT --> upsert
  >
  > DELETE --> delete
  >
  > UPDATE --> upsert：更新操作不像 Retract 那样生成多条记录
  
  - 包含两种类型的 msg: upsert messages 和delete messages；
  
  - 根据 key 进行 update 和 delete
  
  - Q: 与 retract 流的区别？
  
    The main difference to a retract stream is that `UPDATE` changes are encoded with a single message and hence more efficient. 
  
  ![image-20220119204343500](/Users/zhongxwang/dev/git/alpha/alpha-notes/img/flink/flink-dynamictable-stream-upsertdelete.png)



### TimeStamp & Watermark

**创建**

Table 定义，分两部分：

- logical schema conf
- connector conf

语法：

- 建表：https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/create/#create-table 

- Watermark https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/create/#watermark
- Time Attribute https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/concepts/time_attributes/

```sql
CREATE TABLE FileSource (
    pageId INT,
    url VARCHAR,
    userId VARCHAR,
    `timestamp` BIGINT,
    WATERMARK wk FOR `timestamp` AS withOffset(`timestamp`, 1000)
) WITH (
    'connector.type'='File',
    'connector.path'='/tmp/test/test.txt'
);
```



**Processing Time 定义**

- 建表 DDL 中定义
  -- AS PROCTIME()

  ```sql
  CREATE TABLE user_actions (
    user_name STRING,
    data STRING,
    user_action_time AS PROCTIME() -- 声明一个额外的列作为处理时间属性
  ) WITH ( 
    ...
  );
  
  SELECT TUMBLE_START(user_action_time, INTERVAL '10' MINUTE), COUNT(DISTINCT user_name) 
  FROM user_actions
  GROUP BY TUMBLE(user_action_time, INTERVAL '10' MINUTE);
  ```

- DataStream 转 table 时定义

  ```java
  DataStream<Tuple2<String, String>> stream = ...;
  // 声明一个额外的字段作为时间属性字段
  Table table = tEnv.fromDataStream(stream, $("user_name"), $("data"),
                                    $("user_action_time").proctime());
  
  WindowedTable windowedTable = table.window( 
    Tumble.over(lit(10).minutes())
    .on($("user_action_time")) 
    .as("userActionWindow"));
  ```

- 自定义 StreamTableSource



**EventTime 定义**

- 建表 DDL 中定义 
  -- 与watermark一起定义

  ```sql
  CREATE TABLE user_actions (
    user_name STRING,
    data STRING,
    user_action_time TIMESTAMP(3),
    -- 声明 user_action_time 是事件时间属性，并且用 延迟 5 秒的策略来生成 
    watermark WATERMARK FOR user_action_time AS user_action_time - INTERVAL '5' SECOND
  ) WITH ( 
    ...
  );
  
  SELECT TUMBLE_START(user_action_time, INTERVAL '10' MINUTE), COUNT(DISTINCT user_name) 
  FROM user_actions
  GROUP BY TUMBLE(user_action_time, INTERVAL '10' MINUTE);
  ```

  

- DataStream 转 table 时定义

  ```java
  //方式一:基于 stream 中的事件产生时间戳和 watermark
  DataStream<Tuple2<String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);
  // 声明一个额外的逻辑字段作为事件时间属性
  Table table = tEnv.fromDataStream(stream, $("user_name"), $("data"),
  $("user_action_time").rowtime()");
                                    
  // 方式二:从第一个字段获取事件时间，并且产生 watermark
  DataStream<Tuple3<Long, String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);
  // 第一个字段已经用作事件时间抽取了，不用再用一个新字段来表示事件时间了
  Table table = tEnv.fromDataStream(stream, $("user_action_time").rowtime(), $("user_name"), $("data"));
                                    
  // Usage:
  WindowedTable windowedTable = table.window(Tumble
                                             .over(lit(10).minutes()) 
                                             .on($("user_action_time")) 
                                             .as("userActionWindow"))
  ```

- 自定义 StreamTableSource



注意：

- 如果原始字段是 字符串，例如 `2020-04-15 20:13:40.564`，建议定义为 TIMESTAMP 类型
- 如果原始字段是 Long，建议定义为 TIMESTAMP_LTZ 类型



### Versioned Table

Flink SQL operates over dynamic tables that evolve, which may either be append-only or updating. Versioned tables represent a special type of updating table that remembers *the past values for each key*.

定义：PRIMARY KEY + Time attribute

- https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/concepts/versioned_tables/

  ```sql
  CREATE TABLE products (
  	product_id    STRING,
  	product_name  STRING,
  	price         DECIMAL(32, 2),
  	update_time   TIMESTAMP(3) METADATA FROM 'value.source.timestamp' VIRTUAL,
  	PRIMARY KEY (product_id) NOT ENFORCED,
  	WATERMARK FOR update_time AS update_time
  ) WITH (...);

场景：

- 产品价格历史表，在不同时间点计算价格





### Temporal Table

A Temporal table function provides access to the version of a temporal table at a specific point in time.

示例：关联汇率表

- 原始数据

  ```
  SELECT * FROM currency_rates;
  
  update_time   currency   rate
  ============= =========  ====
  09:00:00      Yen        102
  09:00:00      Euro       114
  09:00:00      USD        1
  11:15:00      Euro       119
  11:49:00      Pounds     108
  ```

- 注册 Temporal Table Function

  ```java
  TemporalTableFunction rates = tEnv
      .from("currency_rates").
      .createTemporalTableFunction("update_time", "currency");
   
  tEnv.registerFunction("rates", rates);  
  ```

- 关联

  ```sql
  SELECT
    SUM(amount * rate) AS amount
  FROM
    orders,
    LATERAL TABLE (rates(order_time))
  WHERE
    rates.currency = orders.currency
  ```

  





## || Query

- https://nightlies.apache.org/flink/flink-docs-release-1.10/dev/table/sql/queries.html 
- https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/overview/ 



Querying dynamic tables yields a *Continuous Query*：

- 永不停止
- 产生动态结果、不断更新



```sql
INSERT INTO
  Console
  SELECT
      a.p,
      b.url,
      c.weight,
  FROM
      (Select p, ciid, TNAME FROM Pulsar WHERE p<> '12345') a
  LEFT JOIN
      SIDE_TABLE_1 b
  ON
      a.p=b.p
  LEFT JOIN
      SIDE_TABLE_2 c
  ON a.p=c.p
  WHERE c.p <> NULL;
```



- **常规查询**
  - 全表计算，结果集不断被更新；
  - 需要维护更多 state，key 很多时会占用内存资源；

![image-20220119115612375](../img/flink/flink-table-query-continuous.png)

- **窗口查询** - tumbling window
  
  - 结果集不会被更新，而是 append-only
  
  - 不会出现资源不足；
  
  - 使用场景：
  
    - 状态大小查询限制 - 连续运行数周或数月
  
    - Updating Results 查询限制 -  有些查询需要重新计算和更新大量已输出的结果行，即使只添加或更新一条输入记录。
  
      ```sql
      SELECT user, RANK() -- rank() 需要更新大量数据
      OVER (ORDER BY lastLogin) 
      FROM (
        SELECT user, MAX(cTime) AS lastAction FROM clicks GROUP BY user );

![image-20220119115734094](../img/flink/flink-table-query-window.png)



**基于时间的查询**

例子

- 计算最近一分钟平均值
- 关联最近汇率变化表
- 五分钟内如果三次失败则触发报警



特征

- 输入表是 append-only
- 查询条件中包含时间相关条件和算子
- 查询结果也是 append-only 类型



操作

- JOIN

  - Time-windowed JOIN
  - JOIN with a teamporal table (enrichment join)

- Pattern matching (MATCH_RECOGNIZE)

- 聚合

  - **GROUP BY window aggregation**

    ```sql
    -- 计算每个小时中，每个用户的点击次数
    SELECT user,
      TUMBLE_END(cTime, INTERVAL '1' HOURS) AS endT, 
      COUNT(url) AS cnt
    FROM clicks
      GROUP BY TUMBLE(cTime, INTERVAL '1' HOURS), user
    ```

    

  - **OVER window aggregation**

    > https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/over-agg/
    >
    > `OVER` aggregates compute an aggregated value for **every input row** over a range of ordered rows. 
    >
    > In contrast to `GROUP BY` aggregates, `OVER` aggregates do not reduce the number of result rows to a single row for every group. Instead `OVER` aggregates produce an aggregated value for every input row.
    
    ```sql
    -- 每个order发生时，统计前一小时相同 product order 的总数
    SELECT order_id, order_time, amount,
      SUM(amount) OVER (
        PARTITION BY product
        ORDER BY order_time
        RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
      ) AS one_hour_prod_amount_sum
    FROM Orders
    
    -- 还可以在SELECT之外定义 Over Window，可读性更强
    SELECT order_id, order_time, amount,
      SUM(amount) OVER w AS sum_amount,
      AVG(amount) OVER w AS avg_amount
    FROM Orders
    WINDOW w AS (
      PARTITION BY product
      ORDER BY order_time
      RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW)
    ```
    
    



## || Window

https://nightlies.apache.org/flink/flink-docs-release-1.11/dev/table/sql/queries.html#group-windows 

https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/window-agg/ 

Windows are at the heart of processing infinite streams. Windows split the stream into “buckets” of finite size, over which we can apply computations.



**Windowing TVF**

> Windowing Table-Valued Functions
>
> https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/window-tvf/





**Window Aggregation**

> https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/window-agg/#group-window-aggregation
>
> Group Window Aggregation 不推荐使用，更推荐 Window TVF Aggregation. 

- **Group Window Function**

| Group Window Function                | Description                                                  |
| :----------------------------------- | :----------------------------------------------------------- |
| `TUMBLE(time_attr, interval)`        | Defines a tumbling time window. A tumbling time window assigns rows to non-overlapping, continuous windows with a fixed duration (`interval`). For example, a tumbling window of 5 minutes groups rows in 5 minutes intervals. Tumbling windows can be defined on event-time (stream + batch) or processing-time (stream). |
| `HOP(time_attr, interval, interval)` | Defines a hopping time window (called sliding window in the Table API). A hopping time window has a fixed duration (second `interval` parameter) and hops by a specified hop interval (first `interval` parameter). If the hop interval is smaller than the window size, hopping windows are overlapping. Thus, rows can be assigned to multiple windows. For example, a hopping window of 15 minutes size and 5 minute hop interval assigns each row to 3 different windows of 15 minute size, which are evaluated in an interval of 5 minutes. Hopping windows can be defined on event-time (stream + batch) or processing-time (stream). |
| `SESSION(time_attr, interval)`       | Defines a session time window. Session time windows do not have a fixed duration but their bounds are defined by a time `interval` of inactivity, i.e., a session window is closed if no event appears for a defined gap period. For example a session window with a 30 minute gap starts when a row is observed after 30 minutes inactivity (otherwise the row would be added to an existing window) and is closed if no row is added within 30 minutes. Session windows can work on event-time (stream + batch) or processing-time (stream). |



- **Auxiliary Function**

| Auxiliary Function                                     | Description                                   |
| :----------------------------------------------------- | :-------------------------------------------- |
| *_START<br /> `TUMBLE_START(time_attr, interval)`      | `HOP_START(time_attr, interval, interval)`    |
| *_END<br />`TUMBLE_END(time_attr, interval)`           | `HOP_END(time_attr, interval, interval)`      |
| *_ROWTIME<br />`TUMBLE_ROWTIME(time_attr, interval)`   | `HOP_ROWTIME(time_attr, interval, interval)`  |
| *_PROCTIME<br />`TUMBLE_PROCTIME(time_attr, interval)` | `HOP_PROCTIME(time_attr, interval, interval)` |

```sql
INSERT INTO ConsoleSink
    SELECT
        userId,
        TUMBLE_START(wk, INTERVAL '5' SECOND), --
        COUNT(DISTINCT pageId)
    FROM FileSource
    GROUP BY
        userId, TUMBLE(wk, INTERVAL '5' SECOND)
```





**Window TopN**

> https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/topn/
>
> https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/window-topn/

```sql
SELECT *
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS row_num
  FROM ShopSales)
WHERE row_num <= 5

SELECT *
  FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY window_start, window_end ORDER BY price DESC) as rownum
    FROM (
      SELECT window_start, window_end, supplier_id, SUM(price) as price, COUNT(*) as cnt
      FROM TABLE(
        TUMBLE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '10' MINUTES))
      GROUP BY window_start, window_end, supplier_id
    )
  ) WHERE rownum <= 3;
```



**Window Join**

> https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/window-join/

- the window join joins the elements of two streams that share a common key and lie in the same window.



## || UDF

**内置函数**

- 比较函数：`>`, `value IN`
- 逻辑函数：`AND`, `NOT`
- 算术函数：`ABS()`, `EXP()`
- 字符串函数：`UPPER()`, `CONCAT()`
- 时间函数：`WEEK()`
- 条件函数：`CASE value WHEN THEN END`, `NULLIF`
- 类型转换函数：`CAST()`
- JSON 函数：`JSON_EXISTS()`, `JSON_VALUE()`
- 聚合函数：`count()`, `avg()`



**Flink 函数类别**

- **Scalar functions 标量函数** map scalar values to a new scalar value.
- **Table functions 表值函数** map scalar values to new rows.
- **Aggregate functions 聚合函数** map scalar values of multiple rows to a new scalar value.
- **Table aggregate functions 表值聚合函数** map scalar values of multiple rows to new rows.
- **Async table functions 异步表值函数** are special functions for table sources that perform a lookup.



引入 Function:

- `CREATE FUNCTION PARSE_MSG as 'com.ebay.bes.migration.verification.udf.MessageParserUdf';` 



### ScalarFunction

**标量函数：ScalarFunction**

```java
// 定义函数逻辑
public static class SubstringFunction extends ScalarFunction {
  public String eval(String s, Integer begin, Integer end) {
    return s.substring(begin, end);
  }
}

TableEnvironment env = TableEnvironment.create(...);

// 在 Table API 里不经注册直接“内联”调用函数
env.from("MyTable").select(call(SubstringFunction.class, $("myField"), 5, 12));

// 注册函数
env.createTemporarySystemFunction("SubstringFunction", SubstringFunction.class);

// 在 Table API 里调用注册好的函数
env.from("MyTable").select(call("SubstringFunction", $("myField"), 5, 12));

// 在 SQL 里调用注册好的函数
env.sqlQuery("SELECT SubstringFunction(myField, 5, 12) FROM MyTable");


```



### AggregateFunction

**聚合函数 AggregateFunction**

```java
// 聚合某一列的加权平均

//Accumulator for WeightedAvg.
public static class WeightedAvgAccum {
    public long sum = 0;
    public int count = 0;
}

public static class WeightedAvg extends AggregateFunction<Long, WeightedAvgAccum> {

    //必选：创建 Accumulator，用于存储聚合的中间结果
    public WeightedAvgAccum createAccumulator() {
        return new WeightedAvgAccum();
    }

    //必选：计算和返回最终结果
    public Long getValue(WeightedAvgAccum acc) {
        if (acc.count == 0) {
            return null;
        } else {
            return acc.sum / acc.count;
        }
    }

    //必选：对于每一行数据，会调用 accumulate() 方法来更新 accumulator
    public void accumulate(WeightedAvgAccum acc, long iValue, int iWeight) {
        acc.sum += iValue * iWeight;
        acc.count += iWeight;
    }

    //可选：在 bounded OVER 窗口中是必须实现的
    public void retract(WeightedAvgAccum acc, long iValue, int iWeight) {
        acc.sum -= iValue * iWeight;
        acc.count -= iWeight;
    }

    //可选：在许多批式聚合和会话以及滚动窗口聚合中是必须实现的
    public void merge(WeightedAvgAccum acc, Iterable<WeightedAvgAccum> it) {
        Iterator<WeightedAvgAccum> iter = it.iterator();
        while (iter.hasNext()) {
            WeightedAvgAccum a = iter.next();
            acc.count += a.count;
            acc.sum += a.sum;
        }
    }

    public void resetAccumulator(WeightedAvgAccum acc) {
        acc.count = 0;
        acc.sum = 0L;
    }
}

// 注册函数
StreamTableEnvironment tEnv = ...
tEnv.registerFunction("wAvg", new WeightedAvg());

// 使用函数
tEnv.sqlQuery("SELECT user, wAvg(points, level) AS avgPoints FROM userScores GROUP BY user");

```



### Table Function

> A user-defined table function (*UDTF*) takes zero, one, or multiple scalar values as input arguments. However, it can return an arbitrary number of rows (or structured types) as output instead of a single value.





**表值函数：TableFunction**

-- 表值函数的求值方法本身不包含返回类型，而是通过 `collect(T)` 方法来发送要输出的行。

```java
@FunctionHint(output = @DataTypeHint("ROW<word STRING, length INT>"))
public static class SplitFunction extends TableFunction<Row> {

  public void eval(String str) {
    for (String s : str.split(" ")) {
      // eval()无返回值，use collect(...) to emit a row
      collect(Row.of(s, s.length()));
    }
  }
}

TableEnvironment env = TableEnvironment.create(...);

// 在 Table API 里不经注册直接“内联”调用函数
env
  .from("MyTable")
  .joinLateral(call(SplitFunction.class, $("myField")))
  .select($("myField"), $("word"), $("length"));
env
  .from("MyTable")
  .leftOuterJoinLateral(call(SplitFunction.class, $("myField")))
  .select($("myField"), $("word"), $("length"));

// 在 Table API 里重命名函数字段
env
  .from("MyTable")
  .leftOuterJoinLateral(call(SplitFunction.class, $("myField")).as("newWord", "newLength"))
  .select($("myField"), $("newWord"), $("newLength"));

// 注册函数
env.createTemporarySystemFunction("SplitFunction", SplitFunction.class);

// 在 Table API 里调用注册好的函数
env
  .from("MyTable")
  .joinLateral(call("SplitFunction", $("myField")))
  .select($("myField"), $("word"), $("length"));
env
  .from("MyTable")
  .leftOuterJoinLateral(call("SplitFunction", $("myField")))
  .select($("myField"), $("word"), $("length"));

// 在 SQL 里调用注册好的函数
env.sqlQuery(
  "SELECT myField, word, length " +
  "FROM MyTable, LATERAL TABLE(SplitFunction(myField))");
env.sqlQuery(
  "SELECT myField, word, length " +
  "FROM MyTable " +
  "LEFT JOIN LATERAL TABLE(SplitFunction(myField)) ON TRUE");

```



**表值聚合函数 TableAggregateFunction** 

-- 把一个表聚合成另一张表，结果中可以有多行多列

```java
// 聚合 TOP-2

public class Top2Accum {
    public Integer first;
    public Integer second;
}

public static class Top2 extends TableAggregateFunction<Tuple2<Integer, Integer>, Top2Accum> {

    @Override
    public Top2Accum createAccumulator() {
        Top2Accum acc = new Top2Accum();
        acc.first = Integer.MIN_VALUE;
        acc.second = Integer.MIN_VALUE;
        return acc;
    }


    public void accumulate(Top2Accum acc, Integer v) {
        if (v > acc.first) {
            acc.second = acc.first;
            acc.first = v;
        } else if (v > acc.second) {
            acc.second = v;
        }
    }

    public void merge(Top2Accum acc, java.lang.Iterable<Top2Accum> iterable) {
        for (Top2Accum otherAcc : iterable) {
            accumulate(acc, otherAcc.first);
            accumulate(acc, otherAcc.second);
        }
    }

    public void emitValue(Top2Accum acc, Collector<Tuple2<Integer, Integer>> out) {
        // emit the value and rank
        if (acc.first != Integer.MIN_VALUE) {
            out.collect(Tuple2.of(acc.first, 1));
        }
        if (acc.second != Integer.MIN_VALUE) {
            out.collect(Tuple2.of(acc.second, 2));
        }
    }
}

// 注册函数
StreamTableEnvironment tEnv = ...
tEnv.registerFunction("top2", new Top2());

// 初始化表
Table tab = ...;

// 使用函数
tab.groupBy("key")
    .flatAggregate("top2(a) as (v, rank)")
    .select("key, v, rank");


```



### Type Inference

https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/functions/udfs/#type-inference

**@DataTypeHint**

support the automatic extraction *inline* for parameters and return types of a function

```java
// function with overloaded evaluation methods
public static class OverloadedFunction extends ScalarFunction {

  // no hint required
  public Long eval(long a, long b) {
    return a + b;
  }

  // define the precision and scale of a decimal
  public @DataTypeHint("DECIMAL(12, 3)") BigDecimal eval(double a, double b) {
    return BigDecimal.valueOf(a + b);
  }

  // define a nested data type
  @DataTypeHint("ROW<s STRING, t TIMESTAMP_LTZ(3)>")
  public Row eval(int i) {
    return Row.of(String.valueOf(i), Instant.ofEpochSecond(i));
  }

  // allow wildcard input and customly serialized output
  @DataTypeHint(value = "RAW", bridgedTo = ByteBuffer.class)
  public ByteBuffer eval(@DataTypeHint(inputGroup = InputGroup.ANY) Object o) {
    return MyUtils.serializeToByteBuffer(o);
  }
}

```



**@FunctionHint** 

provide a mapping from argument data types to a result data type.

```java
// function with overloaded evaluation methods
// but globally defined output type
@FunctionHint(output = @DataTypeHint("ROW<s STRING, i INT>"))
public static class OverloadedFunction extends TableFunction<Row> {

  public void eval(int a, int b) {
    collect(Row.of("Sum", a + b));
  }

  // overloading of arguments is still possible
  public void eval() {
    collect(Row.of("Empty args", -1));
  }
}
```



```java
// decouples the type inference from evaluation methods,
// the type inference is entirely determined by the function hints
@FunctionHint(
  input = {@DataTypeHint("INT"), @DataTypeHint("INT")},
  output = @DataTypeHint("INT")
)
@FunctionHint(
  input = {@DataTypeHint("BIGINT"), @DataTypeHint("BIGINT")},
  output = @DataTypeHint("BIGINT")
)
@FunctionHint(
  input = {},
  output = @DataTypeHint("BOOLEAN")
)
public static class OverloadedFunction extends TableFunction<Object> {

  // an implementer just needs to make sure that a method exists
  // that can be called by the JVM
  public void eval(Object... o) {
    if (o.length == 0) {
      collect(false);
    }
    collect(o[0]);
  }
}
```



**getTypeInference()**

```java
public static class LiteralFunction extends ScalarFunction {
  public Object eval(String s, String type) {
    switch (type) {
      case "INT":
        return Integer.valueOf(s);
      case "DOUBLE":
        return Double.valueOf(s);
      case "STRING":
      default:
        return s;
    }
  }

  // the automatic, reflection-based type inference is disabled and
  // replaced by the following logic
  @Override
  public TypeInference getTypeInference(DataTypeFactory typeFactory) {
    return TypeInference.newBuilder()
      // specify typed arguments
      // parameters will be casted implicitly to those types if necessary
      .typedArguments(DataTypes.STRING(), DataTypes.STRING())
      // specify a strategy for the result data type of the function
      .outputTypeStrategy(callContext -> {
        if (!callContext.isArgumentLiteral(1) || callContext.isArgumentNull(1)) {
          throw callContext.newValidationError("Literal expected for second argument.");
        }
        // return a data type based on a literal
        final String literal = callContext.getArgumentValue(1, String.class).orElse("STRING");
        switch (literal) {
          case "INT":
            return Optional.of(DataTypes.INT().notNull());
          case "DOUBLE":
            return Optional.of(DataTypes.DOUBLE().notNull());
          case "STRING":
          default:
            return Optional.of(DataTypes.STRING());
        }
      })
      .build();
  }
}
```









## || JOIN

- https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/joins/

| Join type                    | Description                                                  | Left table type    | Right table type       | Scenario                                                     |
| ---------------------------- | ------------------------------------------------------------ | ------------------ | ---------------------- | ------------------------------------------------------------ |
| Side join                    | 动态表 join  side table (lookup/dimension)： `LEFT JOIN`.    | Dynamic fact table | Static dimension table | your business logic is that consuming the kafka topic and enrich every event with the dimension table in Cassandra/Couchbase/Mongo/Mysql/NuKV/Oracle/Restful |
| Regular join                 | 两个动态表联合： `INNER join`.                               | Dynamic fact table | Dynamic fact table     | join the data from one kafka topic with the data from another kafka topic |
| Interval join                | 两个动态表在 **time interval** 内联合：`INNER join`.         | Dynamic fact table | Dynamic fact table     | join the data from one kafka topic with the data from another kafka topic |
| Temporal table function join | 动态表 join  side table (lookup/dimension)： `LEFT JOIN`.  --> 与 Side Join 的区别？ | Dynamic fact table | Static dimension table | write your user defined side table then use side join or write a temporal table function then use it with Temporal table function join |





- Side Join vs. Temporal Table Function Join
  - side join support async query and have cache function. It will take more effect to develop side join user defined table. 

| Table type         | Operating constraints      |
| :----------------- | :------------------------- |
| source table       | Only support `FROM`        |
| side table         | Only support `JOIN`        |
| result(sink) table | Only support `INSERT INTO` |
| view table         | Only support `FROM`        |



Join 静态表

- JOIN 时数据完整可用

Join 动态表

- 一定要有时间条件

**Join 时态表（Temporal Table）**

- 概念

  - Temporal Table 是随时间变化而变化的表；相当于一个快照
  - 主要用于维表关联
  - 例如：查询之前某个指定时间的最新汇率

- vs. Temporal table function 时态表函数：使用 UDTF + LATERAL TABLE 语法 转换成一张表

- 应用

  



### Regular JOIN

**Inner JOIN**

- Join 顺序没有进行优化
- 状态可能无限增长，需要配置 TTL 

**Outer JOIN**

- 同上



### Interval JOIN

- 至少需要一个限制时间的 join 条件；

  - State 会自动在该时间内过期。

- 只支持 INNER JOIN.

  - Q: 无法发出未匹配成功的事件？

- 示例：

  ```sql
  SELECT * 
  FROM Orders o, Shipments s
  WHERE o.id = s.orderId
    AND o.ordertime BETWEEN s.shiptime - INTERVAL'4'HOUR AND s.shiptime
  ```

![](../img/flink/flink-sql-interval-join1.png)

![image-20220119222504551](../img/flink/flink-sql-interval-join2.png)



![image-20220213200623048](../img/flink/flink-interval-join.png)

> https://stackoverflow.com/questions/66537833/compare-2-data-streams-in-flink-to-retrieve-missing-data
>
> Q: 使用 Interval Join 找出两个topic中不匹配的event：
>
> You may try to use the `IN` clause on interval join. So, You basically select all elements from T1 that are also in T2 to create T3 and then only select elements from T1 that are not in T3 using `NOT IN` clause as described in the [documentation](https://ci.apache.org/projects/flink/flink-docs-stable/dev/stream/operators/joining.html). Note that this will only work if the elements are UNIQUE.
>
> 
>
> The other thing You may try is to handle this Yourself by using `CoProcessFunction`, so You do something like:
>
> ```java
> dataStream1.keyBy(_).connect(datastream2.keyBy(_))
> .process(new MyProcessFunction())
> ```
>
> Inside the function You would simply have a state that would keep every element from `dataStream1` and whenever anything from `dataStream2` arrives You would check if You can join, if it has the timestamp in given boundaries, if so You would delete the data from state as it's not going to be emitted. You could also have a registered timer, that would emit all elements that were not joined.





### Temporal JOIN



A Temporal table is a table that evolves over time (dynamic table)

- Event Time temporal joins allow joining against a [versioned table](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/concepts/versioned_tables/). This means a table can be **enriched** with changing metadata and retrieve its value at a certain point in time.
- Temporal joins take an arbitrary table (left input/**probe side**) and correlate each row to the corresponding row’s relevant version in the versioned table (right input/**build side**). 
- Flink uses the SQL syntax of `FOR SYSTEM_TIME AS OF` to perform this operation from the SQL:2011 standard.

```sql
SELECT 
     order_id,
     price,
     currency,
     conversion_rate,
     order_time
-- orders 表：append only dynamic table     
FROM orders 
-- currency_rates 表：versioned table, CDC, compacted topic
LEFT JOIN currency_rates FOR SYSTEM_TIME AS OF orders.order_time
ON orders.currency = currency_rates.currency;
```



对比

- VS. [regular joins](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/joins/#regular-joins), temporal table results will not be affected despite the changes on the build side. 
- VS. [interval joins](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sql/queries/joins/#interval-joins), temporal table joins do not define a time window within which the records will be joined. Records from the **probe side** are always joined with the **build side**’s version at the time specified by the time attribute. Thus, rows on the build side might be arbitrarily old. As time passes, no longer needed versions of the record (for the given primary key) will be removed from the state.



### Lookup JOIN

- A lookup join is typically used to enrich a table with data that is queried from an external system. 



```sql
-- enrich each order with customer information
SELECT o.order_id, o.total, c.country, c.zip
  -- orders 表：append only dynamic table  
FROM Orders AS o
  -- Customers 表：静态
  JOIN Customers FOR SYSTEM_TIME AS OF o.proc_time AS c
    ON o.customer_id = c.id;
```





### Window JOIN

- DataStream 支持，但 Flink sql 不支持
- 





## || Table Aggregation Function

https://ci.apache.org/projects/flink/flink-docs-release-1.11/dev/table/functions/udfs.html#table-aggregation-functions.



# |ProcessFunction

> https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/datastream/operators/process_function/ 

![image-20230214153659078](../img/flink/layered-abstraction.png)



ProcessFunction 是底层 API，可以访问：

- 时间
- 状态：getRuntimeContext().getState()
- 定时器

接口

- 两个回调均有 Collector，可以 emit results
- Context 可以访问 TimerService：获知当前的处理时间、Watermark、当前的time window

```java
//Process one element from the input stream.
void processElement(I value, Context ctx, Collecot<O> out);

//Called when a timer fires.
void onTimer(long timestamp, OnTimerContext ctx, Collector<O> out);
```



示例

![image-20230213235840323](../img/flink/processfunction-example.png)





# | Connector

> - develop connector tips: https://www.youtube.com/watch?v=ZkbYO5S4z18 
> - develop connector example，口音重: https://www.youtube.com/watch?v=LCMfbGv38u8
> - table api? https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/sourcessinks/ 



## || Client

用法：

```java
source = ...
streamExecEnv.addSource(source);

DataStream<SomeObject> dataStream = (...);
sink = dataStream.addSink(sink);
```

例子：

- Kafka Source
  - Flink 自己管理 offset；
  - 提交 offset：一般在 checkpointing 时提交；



## || Design





**Key Challenges**

- How to **parallelize** your data source/sink?
  - subdivide source data --> partition
  - support parallelism changes
- How to provide **fault tolerance**
  - Exactly-once semantic? 
  - Support flink checkpoints & savepoints
- How to support **historical** and **real-time** processing?

- Security 



**Connector Lifecycle**

- Construction
- State Init
- Run
- Cancel/Stop



**E2E Exactly-Once**

> https://flink.apache.org/features/2018/03/01/end-to-end-exactly-once-apache-flink.html 



## || Source 

> Flink doc of DataStream source: https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/sources/ 
>
> Develop a Connector: https://cloud.tencent.com/developer/article/1912489 
>
> FLIP-27 Source Interface https://cwiki.apache.org/confluence/display/FLINK/FLIP-27%3A+Refactor+Source+Interface

****



**基本流程**![image-20221009113212031](../img/flink/flink-connector-source.png)

![](../img/flink/connector-arch2-1.png)

- SplitEnumerator 将分片分配给 SourceReader；
- SourceReader 为分片初始化 state、再分配给指定的 `SplitFetcherManager` --> `SplitFetcher` --> `SplitReader` 
- 数据流程：SplitReader 
  --> `SplitFetcher.elementQueues` 
  --> `SourceReaderBase.elementQueues` 
  --> `RecordsWithSplitIds` 
  --> `RecordEmitter`



**核心组件**

- **Split 分片**

  - A Split is a portion of data consumed by the source, like a file or a log partition. Splits are the granularity by which the source distributes the work and parallelizes reading data.

  - 分片是外部系统的一个分区；Split 是进行任务分配、数据并行读取的基本粒度。

  - 分片是需要记录在 checkpoint 中，所以其中应该保存状态信息。——例如当前分区起止位置、当前进度

    > 例：Split = Kafka Topic Partition.

- **SplitEnumerator 分片枚举器**

  - 分片枚举器负责**发现分片**和**分配分片**：生成分片，维护 pending split backlog、并均衡地分配给 SourceReader。

    - Split discovery / split life-cycle
    - Failover / split re-assignment

  - SplitEnumerator 运行在 Job Manager 中。

  - 1 SplitEnumerator per job

    > 例：SplitEnumerator 连接到 broker，列举出已订阅的 Topics 中的所有 Topic Partitions。
    >
    > Kafka source 的分片枚举器负责检查在当前的 topic / partition 订阅模式下的新分片（partition），并将分片轮流均匀地分配给源读取器（source reader）。注意 Kafka source 的分片枚举器会将分片主动推送给源读取器，因此它无需处理来自源读取器的分片请求。

- **SourceReader 源阅读器**

  - The SourceReader requests Splits and processes them, for example by reading the file or log partition represented by the Split. The SourceReaders run in parallel on the Task Managers in the `SourceOperators` and produce the parallel stream of events/records.

  - SourceReader 负责**具体分片数据的读取**：请求分片，并处理分片。

  - SourceReader 并行运行在 Task Manager 中。

  - 1 SourceReader per source subtask

    > 例：使用 **KafkaConsumer** 读取所分配的分片（主题分区），并反序列化记录。
    >
    > Kafka source 的源读取器扩展了 `SourceReaderBase`，并使用单线程复用（single thread multiplex）的线程模型，使用一个由分片读取器 （split reader）驱动的 `KafkaConsumer` 来处理多个分片（partition）。消息会在从 Kafka 拉取下来后在分片读取器中立刻被解析。分片的状态 即当前的消息消费进度会在 `KafkaRecordEmitter` 中更新，同时会在数据发送至下游时指定事件时间。
    

- **Source**

  - API 入口，将上述三个组件组合起来。
  - *Configuration Holder*



**辅助组件**

> https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/sources/

- **SplitReader**
  - 为了让同步的poll api适配异步的 SourceReader： `SourceReader` API is fully asynchronous and requires implementations to manually manage reading splits *asynchronously*. However, in practice, most sources perform blocking operations. To make this compatible with the asynchronous Source API, these blocking (synchronous) operations need to happen in *separate threads*, which hand over the data to the asynchronous part of the reader.
  - SplitReader is the high-level API for simple synchronous reading/polling-based source implementations, like file reading, Kafka, etc.


- **SplitFetcherManager**
  - 负责创建 SplitFetcher 池，每个 SplitFetcher 有一个 SplitReader 用于拉取数据。
    `SplitFetcherManager` helps create and maintain a pool of `SplitFetcher` each fetching with a `SplitReader`. 
  - 负责决定如何将 Split assign 给 SplitFetcher
    It also determines how to assign splits to each split fetcher.



**线程模型**

- SourceReader 支持三种线程模式：

  - 单分片串行读取、

    > 例如 FileSource 采用了单分片串行读取模式，在一个数据分片读取后，再向 SplitEnumerator 请求新的数据分片。

  - 多分片多路复用、

    > 例如 KafkaSource 采用了多分片多路复用模式，SplitEnumerator 把启动时读取的 partition 列表和定期监测时发现的新的 partition 列表批量分配给 SourceReader。
    >
    > SourceReader 使用 KafkaConsumer API 读取所有分配到的 partition 的数据。

  - 多分片多线程

- SingleThreadMultiplexSourceReaderBase/SingleThreadFetcherManager 抽象出的框架支持前两种线程模型

![](../img/flink/connector-source-thread.png)



**容错设计**

- SplitEnumerator 状态保存了未分配的数据分片。
- SourceReader 状态保存了分配的数据分片以及分片读取状态（例如 kafka offset，文件 offset）
- TODO: checkpointing 



通用模板

- Batch processing

![image-20221127232423795](../img/flink/flink-source-template.png)



**样例代码**

需求：接收 slack 消息，并自动回复

- **Source**
  - Configuration holder，用于创建 SplitEnumerator、SourceReader、序列化器


```java
/**
参数1：Source输出数据类型
参数2：数据分片类型
参数3：SplitEnumerator Checkpoint数据类型
*/
class SlackSource implements Source<SlackMsg, SlackNotification, EnumeratorState> {
  
  SlackSettings settings; //needs to be serializable
  
  Boundedness getBoundeness() {
    return Bondedness.CONTINUOUS_UNBOUNDED;
  }
  
  // createReader: Creates a new SourceReader to read data from the splits it gets assigned.
  // SourceReader: reading the records from the source splits assigned by SplitEnumerator.
  SourceReader<SlackMsg, SlackNotification> createReader(SourceReaderContext ctx) {
    return new SlackReader<>(ctx);
  }
  
  // createEnumerator: Creates a new SplitEnumerator for this source, starting a new input. 
  // SplitEnumerator:
  // 1. discover the splits for the SourceReader to read. 
  // 2. assign the splits to the source reader.
  SplitEnumerator<SlackNotification, EnumeratorState> createEnumerator(SplitEnumeratorContext<SlackNotification> ctx) {
    return new SlackEnumerator(ctx, null);
  }
  
  // restoreEnumerator: Restores an enumerator from a checkpoint.
  SplitEnumerator<SlackNotification, EnumeratorState> restoreEnumerator(SplitEnumeratorContext<SlackNotification> ctx, EnumeratorState checkpoint) {
    return new SlackEnumerator(ctx, checkpoint)
}
```

- **SplitEnumerator**
  - Split = slack notification
  - 1 `SlackEnumerator` keeps track of notifications. 
  - 每当收到消息时，SlackEnumerator 创建一个新 split，assign split on request to reader. 


```java
class EnumeratorState {
  Queue<SlackNotification> upprocessedNotification;
}

class SlackEnumerator implements SplitEnumerator<SlackNotificaiton, EnumeratorState> {
  SplitEnumeratorContext<SlackNotification> context;
  //queue of unprocessed msg
  EnumeratorState state; 
  SlackClient client;
  
  // start: invoked just once. 
  // 注册回调：发现新分片
  void start() {
    //Invoke the given callable periodically and handover the return value to the handler which will be executed by the source coordinator. 
    context.callAsync(client::getNotifications, (msgs, ex) -> {
			state.upprocessedNotification.addAll(msgs)
    }, 0, 60_000);
  }
  
  //
  void handleSplitRequest(int subtaskId, String requesterHostname) {
    context.assignSplit(state.unprocessedNotification.poll(), subtaskId);
  }
  
  //SourceReader 失败时会调用 addSplitsBack()。
  //SplitEnumerator 应当收回已经被分配，但尚未被该SourceReader确认（acknowledged）的分片。
  void addSplitsBack(List<SlackNotification> msgs, int subtaskId) {
    state.unprocessedNotification.addAll(msgs);
  }
  
  // failure management
  EnumeratorState snapshotState(long checkpointId) {
    return state;
  }
}
```

- **Source Reader**
  - 推荐继承 `SourceReaderBase`，减少编码量。
  - 1 `SourceReader` per subtask, reads all msgs belonging to split.
  - After reader is done with a notification
    - It goes idle;
    - Requests a new split from enumerator.

```java
class SlackReader implements SourceReader<SlackMsg, SlackNotification> {
  SlackClient client;
  CompletableFuture<Void> available = new CompletableFuture<>();
  
  SlackReader(SlackSettings settings, SourceReaderContext context) {
    client = new SlackClient(settings);
    context.sendSplitRequest();
  }
  
  //
  void addSplits(List<SlackNotification> splits) {
    client.seek(Iterables.getOnlyElement(splits).getMessageOffset());
    available.complete(null);
  }
  
  CompletableFuture<Void> isAvailable() {
    return available;
  }
  
  // Poll the next available record into the {@link ReaderOutput}.
  InputStatus pollNext(ReaderOutput<SlackMessage> output) {
    if (client.hasMoreMessages()) {
      output.collect(client.nextMessage());
      return InputStatus.MORE_AVAILABLE;
    } else {
      context.sendSplitRequest();
      available = new CompletableFuture<>();
      return InputStatus.NOTHING_AVAILABLE;
    }
  }
  
  List<SlackNotification> snapshotState(long checkpointId) {
    return Collections.singletonList(client.getCurrentSplit());
  }
}
```





## || Sink

> FLIP-143: Unified Sink API https://cwiki.apache.org/confluence/display/FLINK/FLIP-143%3A+Unified+Sink+API 

![image-20221128224546373](../img/flink/flink-sink-arch.png)

- **Streaming**: *Committed on checkpoints*
  - Failed committables are **retried** after some time
  - Committables of failed checkpoints carry over to new checkpoint;
  - On **recovery**, recommit all committables of last checkpoint.
- **Batch**: Committed after all data has been processed
  - Indefinite retries on committables



**Sink 模型**：两阶段提交

- **Writer**
  - 写入、预提交
  - Committable：临时文件、undo/redo log
- **Committer**
  - 提交
- Global Committer
  - 可选，并行度为1



**Sample Code**

- Sink
  - Configuration holder


```java
class SlackSink implements Sink<SlackReply, Draft, Void, Void> {
  SlackSettings settings; //need to be serializable
  
  public SinkWriter<SlackReply, Draft, Void> createWriter(InitContext context, List<void> states) {
    return new SlackWriter(settings);
  }
  
  public Optional<SimpleVersionedSerializer<Void>> getWriterStateSerializer() {
    return Optional.empty(); //writer has no state
  }
  public Optional<Committer<Draft>> createCommitter() {
    return Optional.of(new DraftCommitter(settings));
  }
  public Optional<SimpleVersionedSerializer<Draft>> getCommittableSerializer() {
    return Optional.of(new DraftSerializer());
  }
}

```

- SinkWriter
  - 写入数据


```java
class SlackWriter implements SinkWriter<SlackReply, Draft, Void> {
  SlackClient client;
  Queue<Draft> drafts = new ArrayDeque<>();
  
  // write == client call + save draft
  void write(SlackReply element, Context ctx) {
    drafts.add(client.writeDraft(element));
  }
  
  // call at checkpoint
  List<Draft> prepareCommit(boolean flush) {
    ArrayList<Draft> drafts = new ArrayList<>(this.drafts);
    this.drafts.clear();
    return drafts;
  }
}

```

- Commiter
  - Optional, 两阶段提交
  - Exactly-once


```java
class DraftCommitter implements Committer<Draft> {
  SlackClient client;
  
  // the response will be retried.
  List<Draft> commit(List<Draft> drafts) {
    List<Draft> unsuccessful = new ArrayList<>();
    for (Draft d: drafts) {
      if (!client.undraft(d)) {
        unsuccessful.add(d);
      }
    }
    return unsuccessful;
  }
}
```





# | Ops

## || Metrics



- 是否运行正常告警 `fullRestarts > threshold`
- 恢复是否顺利告警 `restartingTime > threshold`
- 吞吐量 `numRecords(In|Out)PerSecond`
- **处理进度** 告警`currentProcessingTime - currentOutputWatermark > threshold` 
  - 也有可能是 watermark 生成策略有问题
- Lag 告警 `records-lag-max > threshold`
  - flink 是在checkpoint 成功后才提交 offset，所以lag是周期性的。
- Checkpoint 
  - 失败告警`numberOfFailedCheckpoints > threshold`
  - `numberOfCompletedCheckpoints`
  - `lastCheckpointSize`



## || Troubleshooting

Production Readiness

- 设置 max parallelism
- 每个 Operator 都要设置 UUID
- 选择合适的 State backend
- 设置 JobManager HA



Q：Memory 问题

- TM 内存粒度：JVM, Slot, SubTask
- Slot sharing 可能影响内存
- 配置参数
  - Total Flink Memory `taskmanager.memory.flink.size`
  - Total Process Memory `taskmanager.memory.process.size`



Q：消费没有进度、有 Lag

- 确保上游有流量
- 检查 Backpressure （从web ui）
- 找到 block operator、检查其 metrics



Q：Checkpoint Fail

- 检查 Backpressure：插入的 barrier 会因为背压导致无法及时处理；
- 检查 Operator 逻辑是否超时
- State 太大



Q：Connector Fail

- 检查服务状态：kafka cluster, ...
- 加入重试逻辑、处理通用 Exception
- 检查防火墙







# | Reference

- playground https://github.com/apache/flink-playgrounds  
- Sample: https://github.com/apache/flink-training
  - https://github.com/ververica/flink-training 









