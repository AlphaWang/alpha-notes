# Redis知识图谱

## 数据类型

### string

set name codehole
set age 30
incr age
incrby age 5

#### SDS, Simple Dynamic String

##### capacity

##### len

##### content：以字节\0结尾

##### 扩容：< 1M 之前，加倍扩容； > 1M后，每次扩容1M 

#### 内部编码

##### raw

##### int

##### embstr

#### 常用命令

##### set k v

##### setnx k v: 不存在才设置(add)

##### set k v xx: key存在 才设置(update)

##### setex k 60 v: 生存时间

##### getset: 返回旧值

##### append: 

##### strlen:

##### incr / decr / incrby / decrby

##### mget / mset

##### del

### hash

hset books java "think in java"


#### dict -> dictht[2] -> disctEntry

##### 扩容时机：元素的个数等于第一维数组的长度; 
bgsave时不扩容，除非达到dict_force_resize_ratio

##### 扩容：扩容 2 倍

##### 缩容：元素个数低于数组长度10%时

##### ziplist: 元素个数较小时，用ziplist节约空间

压缩列表是一块连续的内存空间，元素之间紧挨着存储，没有任何冗余空隙。

##### 渐进式rehash

它会同时保留旧数组和新数组，然后在定时任务中以及后续对 hash 的指令操作中渐渐地将旧数组中挂接的元素迁移到新数组上。这意味着要操作处于 rehash 中的字典，需要同时访问新旧两个数组结构。如果在旧数组下面找不到元素，还需要去新数组下面去寻找。

#### 内部编码

##### hashtable

##### ziplist

###### 节约内存

###### 配置

####### hash-max-ziplist-entries

####### hash-max-ziplist-value

#### 常用命令

##### hget / hset name k v

##### hsetnx name k v

##### hdel

##### hincrby / hincrbyfloat

##### hgetall name

##### hvals name: 返回所有value

##### hkeys name: 返回所有key

### list

rpush books python java golang

#### 内部编码

##### 后期：quicklist

`quicklist` 是 `ziplist` 和 `linkedlist` 的混合体，
- 它将 `linkedlist` 按段切分，
- 每一段使用 `ziplist` 来紧凑存储，多个 `ziplist` 之间使用双向指针串接起来。



##### 早期：元素少时用 ziplist，元素多时用 linkedlist

#### 常用命令

##### lpush / rpop

###### lpush list-name c b a

##### rpush / lpop

##### linsert key before|after v new-v

##### lset key index v

##### llen

##### lrem key count v 

###### count = 0: 删除所有v

###### count > 0: 从左到右删除count个

###### count < 0: 从右到左删除count个

##### ltrim key start end: 修剪

##### lrange key start end: 范围

##### blop / brpop key timeout: 阻塞pop

#### 应用

##### lpush + lpop --> stack

##### lpush + rpop --> queue

##### lpush + ltrim --> capped collection

##### lpush + brpop --> MQ

### set

 sadd books python
 

#### 内部编码

##### hashtable

##### IntSet

intset 是紧凑的数组结构，同时支持 16 位、32 位和 64 位整数。

###### 当元素都是整数并且个数较小时，使用 intset 来存储

#### 常用命令

##### sadd key e

##### srem key e

##### scard: 集合大小

##### sismember: 判断存在

##### srandmember: 随机取元素

##### smembers: 所有元素，慎用

##### sdiff / sinter / sunion

sdiff / sinter / sunion + store destkey
将集合操作的结果存入destkey


#### 应用

##### sadd --> tagging

##### spop / srandmember --> random item

##### sadd + sinter --> social graph

### zset

zadd books 9.0 "think in java"

它类似于 Java 的 SortedSet 和 HashMap 的结合体，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以给每个 value 赋予一个 score，代表这个 value 的排序权重。它的内部实现用的是一种叫做「跳跃列表」的数据结构。

#### hash: value -> score

hash 结构来存储 value 和 score 的对应关系

#### 内部编码

##### skiplist

###### 二分查找

skiplist提供指定 score 的范围来获取 value 列表的功能，二分查找

##### ziplist

###### 元素个数较小时，用ziplist节约空间

#### 常用命令

##### zadd key score e

##### zscore key e

##### zrem key e

###### zremrangebyrank key start end

###### zremrangebyscore key min-score max-score

##### zincrby key score e

##### zcard

##### zrange key start end [withscores]

###### zrevrange

###### zrangbyscore key min-score max-score

##### zcount key min-score max-score

##### zinterstore / zunionstore

## 原理

### AP: 最终一致性

- Redis 的主从数据是 **异步同步** 的，所以分布式的 Redis 系统并不满足「一致性」要求。
- Redis 保证「最终一致性」，从节点会努力追赶主节点，最终从节点的状态会和主节点的状态将保持一致。

### 通讯协议：RESP, Redis Serialization Protocal

单行字符串 以 + 符号开头。
多行字符串 以 $ 符号开头，后跟字符串长度。
整数值 以 : 符号开头，后跟整数的字符串形式。
错误消息 以 - 符号开头。
数组 以 * 号开头，后跟数组的长度。

### 多路复用

#### 指令队列

Redis 会将每个客户端套接字都关联一个指令队列。客户端的指令通过队列来排队进行顺序处理，先到先服务。

#### 响应队列

Redis 服务器通过响应队列来将指令的返回结果回复给客户端。 
- 如果队列为空，那么意味着连接暂时处于空闲状态，不需要去获取写事件，也就是可以将当前的客户端描述符从write_fds里面移出来。等到队列有数据了，再将描述符放进去。避免select系统调用立即返回写事件，结果发现没什么数据可以写。出这种情况的线程会飙高 CPU。

#### epoll事件轮询API

最简单的事件轮询 API 是select函数，它是操作系统提供给用户程序的 API。
输入是读写描述符列表read_fds & write_fds，输出是与之对应的可读可写事件。

同时还提供了一个timeout参数，如果没有任何事件到来，那么就最多等待timeout时间，线程处于阻塞状态。

一旦期间有任何事件到来，就可以立即返回。时间过了之后还是没有任何事件到来，也会立即返回。拿到事件后，线程就可以继续挨个处理相应的事件。处理完了继续过来轮询。于是线程就进入了一个死循环，我们把这个死循环称为事件循环，一个循环为一个周期。

### pipeline

客户端通过改变了读写的顺序带来的性能的巨大提升.

``` 
Pipeline pl = jedis.pipelined();
loop pl.hset("key", "field", "v");
pl.syncAndReturnAll()
```

#### 节省网络开销

#### 注意每次pipeline携带的数据量

#### 注意m操作与pipeline的区别：原子 vs 非原子

#### pipeline每次只能作用在一个redis节点上

### 事务

#### multi/exec/discard

- 所有的指令在 exec 之前不执行，而是缓存在服务器的一个事务队列中，服务器一旦收到 exec 指令，才开执行整个事务队列，执行完毕后一次性返回所有指令的运行结果。

- 因为 Redis 的单线程特性，它不用担心自己在执行队列的时候被其它指令打搅，可以保证他们能得到的「原子性」执行

#### 隔离性

#### 不能保证原子性

Redis 的事务根本不能算「原子性」，而仅仅是满足了事务的「隔离性」，隔离性中的串行化——当前执行的事务有着不被其它事务打断的权利。

#### 结合pipeline

较少网络IO

pipe = redis.pipeline(transaction=true)
pipe.multi()
pipe.incr("books")
pipe.incr("books")
values = pipe.execute()

#### watch

服务器收到了 exec 指令要顺序执行缓存的事务队列时，Redis 会检查关键变量自 watch 之后，是否被修改了 (包括当前事务所在的客户端)。如果关键变量被人动过了，exec 指令就会返回 null 回复告知客户端事务执行失败，这个时候客户端一般会选择重试。

> watch books
OK
> incr books  # 被修改了
(integer) 1
> multi
OK
> incr books
QUEUED
> exec  # 事务执行失败
(nil)

### 过期策略

#### 惰性策略

#### 定时扫描

Redis 默认会每秒进行十次过期扫描，过期扫描不会遍历过期字典中所有的 key，而是采用了一种简单的贪心策略。

从过期字典中随机 20 个 key；
删除这 20 个 key 中已经过期的 key；
如果过期的 key 比率超过 1/4，那就重复步骤 1；

#### 实践：过期时间随机化

### 内部存储结构

#### redisObject

##### type 数据类型

string
hash
list
set
sorted set


##### encoding 编码方式

raw
int
ziplist
linkedlist
hashmap
intset



##### ptr 数据指针

##### vm 虚拟内存

##### 其他

## 持久化

### RDB

#### 触发条件

##### SAVE

###### 同步

###### 阻塞客户端命令

###### 不消耗额外内存

##### BGSAVE

###### 异步

###### 不阻塞客户端命令

###### fork子进程，消耗内存

子进程名：redis-rdb-bgsave


##### 配置文件: save seconds changes

save 900 1

save after 900 seconds if there is at least 1 change to the dataset

###### BGSAVE

###### 不建议打开

##### SHUTDOWN命令时

##### 从节点SYNC时 (=BGSAVE)

#### 原理

##### fork子进程生成快照


- 调用 glibc 的函数fork产生一个子进程，快照持久化完全交给子进程来处理，父进程继续处理客户端请求。

###### 内存越大，耗时越长

###### info: latest_fork_usec

##### COW (Copy On Write)

使用操作系统的 COW 机制来进行数据段页面的分离。
当父进程对其中一个页面的数据进行修改时，会将被共享的页面复制一份分离出来，然后对这个复制的页面进行修改。这时子进程相应的页面是没有变化的，还是进程产生时那一瞬间的数据。

#### 缺点

##### 不可控、会丢失数据

##### 耗时 O(n)、耗性能、耗内存

### AOF

- 先执行指令才将日志存盘.
- 可用 `bgrewriteaof` 指令对 AOF 日志进行瘦身。

#### 触发条件

##### always

##### every second

##### no

#### 原理

##### 写命令刷新到缓冲区

##### 每条命令 fsync 到硬盘AOF文件

#### AOF重写

##### bgrewriteaof 命令

###### 类似bgsave, fork子进程重新生成AOF

##### 配置文件

###### auto-aof-rewrite-min-size

###### auto-aof-rerwrite-percentage

###### 推荐配置

``` 
appendonly yes
appendfilename "appendonly-${port}.aof"
appendfsync everysec
dir /bigdiskpath

no-appendfsync-on-rewrite yes

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

###### 动态应用配置

####### config set appendonly yes

####### config rewrite

#### AOF追加阻塞

##### 对比上次fsync时间，>2s则阻塞

##### info: aof_delayed_fsync (累计值)

### 建议

#### 混合

在 Redis 重启的时候，可以先加载 rdb 的内容，然后再重放增量 AOF 日志。比 AOF 全量文件重放要快很多。

#### 持久化操作主要在从节点进行

## 集群

### 主从

#### 配置

##### slaveof

- 命令
- 配置

##### slave-read-only yes

##### 查看主从状态：info replication

127.0.0.1:6379> info replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=27806,lag=1
slave1:ip=127.0.0.1,port=6381,state=online,offset=27806,lag=1
master_repl_offset:27806
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:27805

#### 主从复制流程

##### 全量复制

###### 1.【s -> m】psync runId offset

首次：`psync ? -1`

###### 2.【m -> s】+FULLRESYNC {runId} {offset}

###### 3.【s】save masterInfo

###### 4.【m】bgsave / write repl_back_buffer

###### 5.【m -> s】send RDB

###### 6.【m -> s】send buffer

###### 7.【s】flush old data

###### 8.【s】load RDB

##### 部分复制

###### 1.【s -> m】psync runId offset

###### 2.【m -> s】CONTINUE

###### 3.【m -> s】send partial data

#### 问题

##### 开销大

###### 【m】bgsave时间开销

###### 【m】RDB网络传输开销

###### 【s】清空数据时间开销

###### 【s】加载RDB时间开销

###### 【s】可能的AOF重写时间

##### 读写分离问题

###### 复制数据延迟

###### 读到过期数据

###### 从节点故障

##### 主从配置不一致

###### maxmemory配置不一致

####### 可能丢失数据

master: maxmemory=4g
slave: maxmemory=2g

当数据>2g，slave会使用maxmemory-policy删除数据，failover之后的表现就是丢失数据。

###### 数据结构优化参数不一致

####### 内存不一致

##### 规避全量复制

###### 第一次全量复制

####### 不可避免

####### 优化：小主节点（小分片），低峰

###### 节点runId不匹配导致复制

####### 主节点重启后runId变化

####### 优化：故障转移（哨兵、集群）

###### 复制积压缓冲区不足

####### 网络中断后无法进行部分复制

####### 优化：rel_backlog_size（默认1m）

##### 规避复制风暴

###### 主节点重启，多个从节点复制

###### 优化：更换复制拓扑

### sentinel

#### 原理

##### 三个定时任务

###### 每1秒，sentinel对其他sentinel和redis执行ping

####### 心跳检测

####### 失败判定依据

###### 每2秒，sentinel通过master的channel交换信息

####### master频道：__sentinel__:hello

####### 交换对节点的看法、以及自身信息

###### 每10秒，sentinel对m/s执行info

####### 发现slave节点

sentinel初始配置只关心master节点

####### 确认主从关系

##### 故障转移流程

######  sentinel 集群可看成是一个 ZooKeeper 集群

###### 【1. 故障发现】多个sentinel发现并确认master有问题

####### 主观下线

相关配置
```
sentinel monitor myMaster <ip> <port> <quorum>
sentinel down-after-milliseconds myMaster <timeout>
```

####### 客观下线

###### 【2. 选主】选举出一个sentinel作为领导

####### 原因：只有一个sentinel节点完成故障转移

####### 实现：通过sentinel is-master-down-by-addr命令竞争领导者

###### 【3. 选master】选出一个slave作为master, 并通知其余slave

####### 选新 master 的原则

######## 选slave-priority最高的

######## 选复制偏移量最大的

######## 选runId最小的

####### 对这个slave执行slaveof no one

###### 【4. 通知】通知客户端主从变化

###### 【5. 老master】等待老的master复活成为新master的slave

####### sentinel会保持对其关注

##### 客户端流程

###### 【0. sentinel集合】 预先知道sentinel节点集合、masterName

###### 【1. 获取sentinel】遍历sentinel节点，获取一个可用节点

###### 【2. 获取master节点】get-master-addr-by-name masterName

###### 【3. role replication】获取master节点角色信息

###### 【4. 变动通知】当节点有变动，sentinel会通知给客户端 （发布订阅）

####### JedisSentinelPool -> MasterListener --> sub "+switch-master"

####### sentinel是配置中心，而非代理！

#### 消息丢失

##### min-slaves-to-write 1

主节点必须至少有一个从节点在进行正常复制，否则就停止对外写服务，丧失可用性

##### min-slaves-max-lag 10

如果 10s 没有收到从节点的反馈，就意味着从节点同步不正常

#### 运维

##### 上下线节点

###### 下线主节点

####### sentinel failover <masterName>

###### 下线从节点

####### 考虑是否做清理、考虑读写分离

###### 上线主节点

####### sentinel failover进行替换

###### 上线从节点

####### slaveof

###### 上线sentinel

####### 参考其他sentinel节点启动

##### 高可用读写分离

###### client关注slave节点资源池

###### 关注三个消息

####### +switch-master: 从节点晋升

####### +convert-to-slave: 切换为从节点

####### +sdown: 主观下线

### codis

#### 用zookeeper存储槽位关系

#### 代价

- 不支持事务；
- 同样 rename 操作也很危险；
- 为了支持扩容，单个 key 对应的 value 不宜过大。
- 因为增加了 Proxy 作为中转层，所有在网络开销上要比单个 Redis 大。
- 集群配置中心使用 zk 来实现，意味着在部署上增加了 zk 运维的代价

##### 不支持事务

##### rename 操作也很危险

##### 为了支持扩容，单个 key 对应的 value 不宜过大

##### 网络开销更大

##### 需要运维zk

### cluster

#### 创建

##### 原生

###### 配置文件：cluster-enabled yes

cluster-enabled yes
cluster-config-file "xx.conf"
cluster-require-full-coverage no
cluster-node-timeout 15000

###### 启动: redis-server *.conf

###### gossip通讯：cluster meet ip port

###### 分配槽(仅对master)：cluster addslots {0...5461}

###### 配置从节点：cluster replicate node-id

##### 脚本

###### 安装ruby

###### 安装rubygem redis

###### 安装redis-trib.rb

##### 验证

###### cluster nodes

###### cluster info

###### cluster slot

###### redis-trib.rb info ip:port

#### 特性

##### 复制

###### 主从复制 (异步)：SYNC snapshot + backlog队列

- slave启动时，向master发起 `SYNC` 命令。

- master收到 SYNC 后，开启 `BGSAVE` 操作，全量持久化。

- BGSAVE 完成后，master将 `snapshot` 发送给slave.

- 发送期间，master收到的来自clien新的写命令，正常响应的同时，会再存入一份到 `backlog 队列`。

- snapshot 发送完成后，master会继续发送backlog 队列信息。

- backlog 发送完成后，后续的写操作同时发给slave，保持实时地异步复制。

####### 快照同步

####### 增量同步

异步将 buffer 中的指令同步到从节点，从节点一边执行同步的指令流来达到和主节点一样的状态，一边向主节点反馈自己同步到哪里了 (偏移量)。

####### 无盘复制

无盘复制是指主服务器直接通过套接字将快照内容发送到从节点，生成快照是一个遍历的过程，主节点会一边遍历内存，一边将序列化的内容发送到从节点，从节点还是跟之前一样，先将接收到的内容存储到磁盘文件中，再进行一次性加载。


####### wait 指令

wait 指令可以让异步复制变身同步复制，确保系统的强一致性。
- `wait N t`: 等待 wait 指令之前的所有写操作同步到 N 个从库，最多等待时间 t。

##### 高可用

##### 分片

###### slots

####### 16384个

####### 槽位信息存储于每个节点中

######## Rax

`Rax slots_to_keys` 用来记录槽位和key的对应关系
- Radix Tree 基数树


####### 每个节点通过meet命令交换槽位信息

####### 定位：crc16(key) % 16384

###### 计算槽位

####### cluster keyslot k

###### 扩展性：迁移slot (同步)

- Redis 迁移的单位是槽，当一个槽正在迁移时，这个槽就处于中间过渡状态。这个槽在原节点的状态为`migrating`，在目标节点的状态为`importing`，  


- 迁移过程是同步的，在目标节点执行`restore指令`到原节点删除key之间，原节点的主线程会处于阻塞状态，直到key被成功删除。 >> 要尽可能避免大key

原理：
- 以原节点作为目标节点的「客户端」
- 原节点对当前的key执行dump指令得到序列化内容，然后发送指令restore携带序列化的内容作为参数
- 目标节点再进行反序列化就可以将内容恢复到目标节点的内存中
- 原节点收到后再把当前节点的key删除


####### dump

####### restore

####### remove

#### 原理

##### 伸缩

###### 扩容

####### 准备新节点

####### 加入集群

######## meet

######## redis-trib.rb add-node

redis-trib.rb add-node new_host:new_port existing_host:existing_port --slave --master-id

####### 迁移槽和数据

######## 手工

######### 1_对目标节点：cluster setslot {slot} importing {sourceNodeId}

######### 2_对源节点：cluster setslot {slot} migrating {targetNodeId}

######### 3_对源节点循环执行：cluster getkeysinslot {slot} {count}，每次获取count个键

######### 4_对源节点循环执行：migrate {targetIp} {targetPort} key 0 {timeout}

0: db0


######### 5_对所有主节点：cluster setslot {slot} node {targetNodeId}

######## pipeline migrate

######## redis-trib.rb reshard

redis-trib.rb reshard host:port
--from
--to
--slots

host:port是任一个节点的

###### 收缩

####### 迁移槽

####### 忘记节点

######## cluster forget {downNodeId}

######## redis-trib.rb del-node

redis-trib.rb del-node ip:port {downNodeId}

####### 关闭节点

###### 迁移slot过程中如何同时提供服务？--> ask

####### 0.先尝试访问源节点

####### 1.源节点返回ASK转向

####### 2.向新节点发送asking命令

在迁移没有完成之前，这个槽位还是不归新节点管理的，它会向客户端返回一个`-MOVED`重定向指令告诉它去源节点去执行。如此就会形成 `重定向循环`。
asking指令的目标就是打开目标节点的选项，告诉它下一条指令不能不理，而要当成自己的槽位来处理。

####### 3.向新节点发送命令

##### 故障转移

###### 故障发现

####### 通过ping/pong发现故障

####### 主观下线

- node1 发送ping消息
- node2 回复pong消息
- node1 收到pong，并更新与node2的`最后通信时间`
- cron定时任务：如果最后通信时间超过node-timeout，则标记为`pfail`

####### 客观下线

- 接受ping
- 消息解析：其他pfail节点 + 主节点发送消息
- 维护故障链表
- 尝试客观下线：计算有效下线报告数量
- if > 槽节点总数一半，则更新为客观下线；
-并向集群广播下线节点的fail消息。

######## 当半数以上主节点都标记其为pfail

###### 故障恢复

####### 资格检查

每个从节点：检查与主节点断线时间；
- if > `cluster-node-timeout` * `cluster-slave-validity-factor`，则取消资格

####### 准备选举时间

offset越大，则延迟选举时间越短


- slave通过向其他master发送FAILOVER_AUTH_REQUEST消息发起竞选，master回复FAILOVER_AUTH_ACK告知是否同意。

####### 选举投票

收集选票，if > N/2 + 1，则可替换zhu'jie'dian

####### 替换主节点

1. slaveof no one
2. clusterDelSlot撤销故障主节点负责的槽；
3. clusterAddSlot把这些槽分配给自己；
4. 向集群广播pong消息，表明已经替换了故障jie

##### 一致性: 保证朝着epoch值更大的信息收敛

保证朝着epoch值更大的信息收敛: 每一个Node都保存了集群的配置信息`clusterState`。

- `currentEpoch`表示集群中的最大版本号，集群信息每更新一次，版本号自增。
- nodes列表，表示集群所有节点信息。包括该信息的版本epoch、slots、slave列表

配置信息通过Redis Cluster Bus交互(PING / PONG, Gossip)。
- 当某个节点率先知道信息变更时，将currentEpoch自增使之成为集群中的最大值。
- 当收到比自己大的currentEpoch，则更新自己的currentEpoch使之保持最新。
- 当收到的Node epoch大于自身内部的值，说明自己的信息太旧、则更新为收到的消息。


#### 客户端

##### 客户端路由

###### moved

####### 1.向任意节点发送命令

####### 2.节点计算槽和对应节点

####### 3.如果指向自身，则执行命令并返回结果

####### 4.如果不指向自身，则回复-moved (moved slot ip:port)

####### 5.客户端重定向发送命令

###### tips

####### redis-cli -c 会自动跳转到新节点

####### moved vs. ask

######## 都是客户端重定向

######## moved: 表示slot确实不在当前节点（或已确定迁移）

######## ask: 表示slot在迁移中

##### 批量操作

###### 问题：mget/mset必须在同一个槽

###### 实现

####### 串行 mget

####### 串行IO

######## 客户端先做聚合，crc32 -> node，然后串行pipeline

####### 并行IO

######## 客户端先做聚合，然后并行pipeline

####### hash_tag

#### 运维

##### 集群完整性

###### cluster-require-full-coverage=yes

####### 要求16384个槽全部可用

####### 节点故障或故障转移时会不可用：(error) CLUSTERDOWN

###### 大多数业务无法容忍

##### 带宽消耗

###### 来源

####### 消息发送频率

节点发现与其他节点最后通信时间超过 cluster-node-timeout/2 时，会发送ping消息


####### 消息数据量

- slots槽数据：2k
- 整个集群1/10的状态数据

####### 节点部署的机器规模

集群分布的机器越多，且每台机器划分的节点数越均匀，则集群内整体可用带宽越高。

###### 优化

####### 避免“大”集群

####### cluster-node-timeout: 带宽和故障转移速度的均衡

####### 尽量均匀分配到多机器上

###### 集群状态下的pub/sub

####### publish在集群中每个节点广播，加重带宽

####### 解决：单独用一套sentinel

##### 倾斜

###### 数据倾斜

####### 节点和槽分配不均

######## redis-trib.rb info 查看节点、槽、键值分布

######## redis-trib.rb rebalance 重新均衡（慎用）

####### 不同槽对应键值数量差异较大

######## CRC16一般比较均匀

######## 可能存在hash_tag

######## cluster countkeyinslot {slot} 查看槽对应键值数

####### 包含bigkey

######## 从节点执行 redis-cli --bigkeys

######## 优化数据结构

####### 内存配置不一致

###### 请求倾斜

####### 原因：热点key、bigkey

####### 优化

######## 避免bigkey

######## 热键不要用hash_tag

######## 一致性不高时，可用本地缓存，MQ

####### 热点key解决思路

######## 客户端统计

```java
AtomicLongMap<String> COUNTER = AtomicLongMap.create();

String get(String key) {
  countKey(key);
  ...
}

String set(String key) {
  countKey(key);
  ...
}
```

######### 实现简单

######### 内存泄露隐患，只能统计单个客户端

######## 代理统计

######### 增加代理端开发部署成本

######## 服务端统计（monitor）

######### monitor本身问题，只能短时间使用

######### 只能统计单个redis节点

######## 机器段统计（抓取tcp）

######### 无侵入

######### 增加了机器部署成本

##### 读写分离

###### 只读连接

####### 从节点不接受任何读写请求

####### 会重定向到负责槽的主节点（moved）

####### readonly命令

- 默认情况下，某个slot对应的节点一定是一个master节点。客户端通过`MOVED`消息得知的集群拓扑结构也只会将请求路由到各个master中。

- 即便客户端将读请求直接发送到slave上，slave也会回复MOVED到master的响应。

- 客户端向slave发送READONLY命令后，slave对于读操作将不再返回moved，而是直接处理。

###### 读写分离客户端会非常复杂

####### 共性问题：复制延迟、过期数据、从节点故障

####### cluster slaves {nodeId} 获取从节点列表

##### 数据迁移

###### redis-trib.rb import

####### 只能 单机 to 集群

####### 不支持在线迁移

####### 不支持断点续传

####### 单线程迁移，影响速度

###### 在线迁移

####### 唯品会 redis-migrate-tool

####### 豌豆荚 redis-port

#### 缺点

##### key批量操作支持有限

###### mget/mset 必须在同一个slot

##### key事务和lua支持有限

###### 操作的key必须在同一个slot

##### key是数据分区最小粒度

###### bigkey无法分区

###### 分支主题

##### 复制只支持一层

###### 无法树形复制

## 应用

### 分布式锁

#### 命令

##### setnx + expire

##### set xx ex 5 nx

#### 集群问题

主节点挂掉时，从节点会取而代之，客户端上却并没有明显感知。原先第一个客户端在主节点中申请成功了一把锁，但是这把锁还没有来得及同步到从节点，主节点突然挂掉了。然后从节点变成了主节点，这个新的节点内部没有这个锁，所以当另一个客户端过来请求加锁时，立即就批准了。这样就会导致系统中同样一把锁被两个客户端同时持有

##### Redlock算法

过半节点加锁成功


### 延时队列

#### lpush / rpush 

#### rpop / lpop -> brpop / blpop

### 位图

位图不是特殊的数据结构，它的内容其实就是普通的字符串，也就是 byte 数组。

我们可以使用普通的 get/set 直接获取和设置整个位图的内容，也可以使用位图操作 getbit/setbit 等将 byte 数组看成「位数组」来处理。

#### type: string, 最大512M

#### 命令

##### setbit k offset v

零存：`setbit s 4 1`
整存：`set s <string>`

##### getbit

整取：`get s`
零取：`getbit s 1`


##### bitcount k [start end] 统计

##### bitop op destKey key1 key2 位运算

op:
- and
- or
- not
- xor

##### bitpos k targetBit [start] [end] 查找

##### bitfield操作多个位

### HyperLogLog

HyperLogLog 提供不精确的去重计数方案

#### 极小空间完成独立数量统计

#### type: string

#### 缺点

##### 有错误率 0.81%

##### 不能返回单条元素

#### 命令

##### 添加：pfadd key e1 e2...

##### 计数：pfcount key

##### 合并：pfmerge destKey sourceKey1 sourceKey2

### 布隆过滤器

布隆过滤器能准确过滤掉那些已经看过的内容，那些没有看过的新内容，它也会过滤掉极小一部分 (误判)

#### 操作

##### bf.exists / bf.mexists

##### bf.add / bf.madd

#### 原理

##### 参数

###### m 个二进制向量

###### n 个预备数据

###### k 个哈希函数

##### 构建

###### n个预备数据，分别进行k个哈希，得出offset，将相应位置的二进制向量置为1

##### 判断

###### 进行k个哈希，得出offset，如果全为1，则判断存在

#### 误差率

##### 与 k (哈希函数)个数成反比

##### 与 n (预备数据)个数成正比

##### 与 m (二进制向量)长度成反比

### 简单限流: zset实现滑动时间窗口

#### key: clientId-actionId

#### value: ms

#### score: ms

### 漏斗限流: redis-cell模块

#### cl.throttle key capacity count period 1

### GeoHash

GeoHash 算法将二维的经纬度数据映射到一维的整数，这样所有的元素都将在挂载到一条线上，距离靠近的二维坐标映射到一维后的点之间距离也会很接近。

#### 用于地理经纬度计算

#### type: zset

#### 命令

##### 添加：geoadd key longitude latitude member

##### 获取：geopos key member

##### 距离：geodist key member1 member2 [unit]

##### 范围：georadius/georadiusbymember 

##### 删除：zrem key member

##### geohash

### 搜索key

#### keys

- 没有 offset、limit 参数，一次性吐出所有满足条件的 key。
- keys 算法是遍历算法，复杂度是 O(n)

#### scan

scan <cursor> match <regex> count <limit>
在 Redis 中所有的 key 都存储在一个很大的字典中，scan 指令返回的游标就是第一维数组的位置索引，limit 参数就表示需要遍历的槽位数
  

### Stream

### PubSub

#### publish channel-name msg

#### subscribe/unsubscribe channel-name

## 运维

### eviction

#### LRU: Least Recently Used

当字典的某个元素被访问时，它在链表中的位置会被移动到表头。

所以链表的元素排列顺序就是元素最近被访问的时间顺序。

位于链表尾部的元素就是不被重用的元素，所以会被踢掉。

- 缺点：需要大量的额外的内存


#### 近似LRU

- **随机**采样出 5(可以配置) 个 key，
- 然后淘汰掉最旧的 key，
- 如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于 maxmemory 为止。

Redis给每个 key 增加了一个额外的小字段，这个字段的长度是 24 个 bit，也就是最后一次被访问的时间戳。



#### LFU: Least Frequently Used

### 内存

#### 内存回收

##### 无法保证立即回收已经删除的 key 的内存

##### flushdb

#### 内存查看：info memory

##### used_memory

mem_allocator 分配的内存量

###### redis自身内存

###### 对象内存

####### 优化

######## key: 不要过长

######## value: ziplist / intset 等优化方式

####### 内存碎片

###### 缓冲内存

####### 客户端缓冲区

client-output-buffer-limit <class> hard_limit soft_limit soft_seconds

  - class: normal, slave, pubsub
  

######## 输出缓冲区

######### 普通客户端 

########## normal 0 0 0

########## 默认无限制，注意防止大命令或 monitor：可能导致内存占用超大！！

########## 找到monitor客户端：client list | grep -v "omem=0"

######### slave 客户端

########## slave 256mb 64mb 60

########## 可能阻塞：主从延迟高时，从节点过多时

######### pubsub 客户端 

########## pubsub 32mb 8mb 60

########## 可能阻塞：生产大于消费时

######## 输入缓冲区

######### 最大 1GB

####### 复制缓冲区

######## repl_back_buffer

######## 默认1M，建议调大 例如100M

######## 防止网络抖动时出现slave全量复制

####### AOF 缓冲区

######## 无限制

###### lua内存

##### used_memory_rss

###### 从操作系统角度看redis进程占用的总物理内存

##### mem_fragmentation_ratio

###### 内存碎片 used_memory_rss / used_memory > 1

###### 内存碎片必然存在

###### 优化

####### 避免频繁更新操作：append, setrange

####### 安全重启

##### mem_allocator

#### 子进程内存消耗

##### 场景

###### bgsave

###### bgrewriteaof

##### 优化

###### 去掉 THP 特性

###### 观察写入量

###### overcommit_memory = 1

#### 内存管理

##### 设置内存上限

###### 一般预留 30%

###### config set maxmemory 6GB

###### config rewrite

##### 动态调整内存上限

##### 内存回收策略

###### 删除过期键值

####### 惰性删除

######## 访问key

######## expired dict

######## del key

####### 定时删除

######## 每秒运行 10 次，采样删除

######## 慢模式：随机检查 20 个key

######### 如果超过25%的key过期

########## 循环执行：快模式？？

######### 否则退出

###### 内存溢出控制策略

####### 配置

######## maxmemory-policy

####### 策略

######## noeviction

######### 默认策略，拒绝写入操作

######## volatile-lru

######### LRU算法删除 有expire的key

######## allkeys-lru

######### LRU算法删除所有key

######## allkeys-random

######### 随机删除所有key

######## volatile-random

######### 随机删除过期key

######## volatile-ttl

######### 删除最近将要过期key

##### 序列化与压缩

###### 拒绝Java原生

###### 推荐protobuf, kryo, snappy

### 保护

####  spiped: SSL代理

#### 设置密码

##### server: requirepass / masterauth

##### client: auth命令 、 -a参数

#### rename-command flushall ""

##### 不支持config set动态配置

#### bind 内网IP

### 懒惰删除

#### del -> unlink

#### flushdb -> flushdb async

### 慢查询

#### 配置

##### slowlog-max-len

###### 先进先出队列、固定长度、内存

###### 默认10ms, 建议1ms

##### slowlog-log-slower-than

###### 建议1000

#### 命令

##### slowlog get [n]

##### slowlog len

##### slowlog reset

## 开发规范

### kv设计

#### key设计

##### 可读性、可管理型

##### 简洁性

###### string长度会影响encoding

####### embstr

####### int

####### raw

####### 通过 `object encoding k` 验证

##### 排除特殊字符

#### value设计

##### 拒绝bigkey

###### 最佳实践

####### string < 10K

####### hash,list,set元素不超过5000

###### bigkey的危害

####### 网络阻塞

####### redis阻塞

####### 集群节点数据不均衡

####### 频繁序列化

###### bigkey的发现

####### 应用异常

######## JedisConnectionException

######### read time out

######### could not get a resource from the pool

####### redis-cli --bigkeys

####### scan + debug object k

####### 主动报警：网络流量监控，客户端监控

####### 内核热点key问题优化

###### bigkey删除

####### 阻塞（注意隐性删除，如过期、rename）

####### unlink命令 （lazy delete, 4.0之后）

####### big hash渐进删除：hscan + hdel

##### 选择合适的数据结构

###### 多个string vs. 一个hash

###### 分段hash

####### 节省内存、但编程复杂

###### 计算网站独立用户数

####### set

####### bitmap

####### hyperLogLog

##### 过期设计

###### object idle time: 查找垃圾kv

###### 过期时间不宜集中，避免缓存穿透和雪崩

### 命令使用技巧

#### O(N)命令要关注N数量

##### hgetall, lrange, smembers, zrange, sinter

##### 更优：hscan, sscan, zscan

#### 禁用危险命令

##### keys, flushall, flushdb

##### 手段：rename机制

#### 不推荐select多数据库

##### 客户端支持差

##### 实际还是单线程

#### 不推荐事务功能

##### 一次事务key必须在同一个slot

##### 不支持回滚

#### monitor命令不要长时间使用

### 连接池

#### 连接数

##### maxTotal

###### 如何预估

例如：
- 一次命令耗时1ms，所以一个连接QPS=1000;
- 业务期望QPS = 50000;

> 则 maxTotal = 50000 / 1000 = 50

####### 业务希望的 redis 并发量

####### 客户端执行命令时间

####### redis 资源：应用个数 * maxTotal < redis最大连接数

##### maxIdle

##### minIdle

#### 等待

##### blockWhenExhausted

资源用尽后，是否要等待；建议设成true


##### maxWaitMillis

#### 有效性检测

##### testOnBorrow 

##### testOnReturn

#### 监控

##### jmxEnabled

#### 空闲资源监测

##### testWhileIdle

##### timeBetweenEvictionRunsMillis

监测周期


##### numTestsPerEvictionRun

每次监测的采样数

