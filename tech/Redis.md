[toc]



# | 数据类型

## || string



功能：

- 类似 ArrayList

  ```shell
  set name codehole
  set age 30
  
  mset name1 v1 name2 v2
  mget name1 name2
  
  incr age
  incrby age 5
  ```





**动态字符串：SDS, Simple Dynamic String**

- 内部属性

  - `capacity`
    - 分配的数组长度
    - 内存优化：短时用byte/short，长时用int

  - `len`
    - 实际使用的长度

  - `byte[]`
    - 数组

- 扩容
  - 小于 1M 之前，加倍扩容 
  - 大于 1M后，每次扩容1M

- content：以字节\0结尾



**内部编码**

- 数字： int

- 字符串

  - embstr：短于 44 字节时使用。RedisObject中的元数据、指针、SDS 使用一块连续的内存区域
  - raw：超过 44 字节时使用

  

**常用命令**

- `set k v`

- `mget` / `mset`

- `del`

- `getset`：返回旧值

- `strlen`

- `append`



- `setex k 60 v`：生存时间

- `set k v xx`：key存在 才设置(update)

- `setnx k v`：不存在才设置(add)；可用于分布式锁



- `incr` / `decr` / `incrby` / `decrby`：计数



## || list

功能：

- 类似 LinkedList

- ```shell
  rpush books python java golang
  ```

特点

- 插入删除 快

- 定位慢



**内部编码**

- 早期：元素少时用 ziplist，元素多时用 linkedlist
  - ziplist：LV
  - linkedlist：双向链表

- 后期：quicklist
  - `quicklist` 是 `ziplist` 和 `linkedlist` 的混合体，它将 `linkedlist` 按段切分，每一段使用 `ziplist` 来紧凑存储，多个 `ziplist` 之间使用双向指针串接起来。



**常用命令**

- `lpush` / `rpop`：lpush list_name c b a

- `rpush` / `lpop`

- `linsert key before|after v new-v`

- `lset key index v`

- `llen`

- `lrem key count v`
  - count = 0: 删除所有v
  - count > 0: 从左到右删除count个
  - count < 0: 从右到左删除count个

- `lindex`：类似get(index)，需要遍历链表，数字越大越慢



- `blop / brpop key timeout`：阻塞pop

- `lrange key start end`：范围

- `ltrim key start end`：修剪，保留[start, end]



**应用**

- 队列：lpush + rpop 

- 栈：lpush + lpop
- MQ：lpush + brpop
  - 缺点：不支持消费者组

- Capped Collection：lpush + ltrim



## || hash

功能

- 类似 HashMap

- ```
  hset books java "think in java"
  ```



原理：

- 组成：dict -> dictht[2] -> dictEntry

- 扩容
  - 扩容时机：元素的个数等于第一维数组的长度;
  - 扩容 2 倍
  - bgsave时不扩容，除非达到dict_force_resize_ratio

- 缩容
  - 元素个数低于数组长度10%时

- 渐进式rehash

  - 否则大数组的扩容会很慢
  - 触发
    - 后续指令触发
    - 定时任务触发

  > - 同时保留旧数组和新数组。然后通过定时任务、以及后续对 hash 的指令操作中，渐渐地将旧数组中挂接的元素迁移到新数组上。
  > - 这意味着要操作处于 rehash 中的字典，需要同时访问新旧两个数组结构。如果在旧数组下面找不到元素，还需要去新数组下面去寻找。



**内部编码**

- hashtable

- ziplist：压缩列表是一块连续的内存空间，元素之间紧挨着存储，没有任何冗余空隙。
  - 节约内存：key 和 value 被存成两个相邻的 entry
  - 类似数组：O(N)
  - 配置
    - hash-max-ziplist-entries
    - hash-max-ziplist-value

**常用命令**

- `hget / hset name k v`

- `hsetnx name k v`

- `hdel`

- `hgetall name`

- `hvals name`: 返回所有value

- `hkeys name`: 返回所有key

- `hincrby` / `hincrbyfloat`：计数



**应用**

- 点赞数、评论数、点击数

- 用 string 也行？



## || set

功能

- 类似 HashSet 

- ```
   sadd books python
  ```



**内部编码**

- hashtable

- IntSet
  - intset 是紧凑的数组结构，同时支持 16 位、32 位和 64 位整数。
  - 当元素都是“整数”并且个数较小时，使用 intset 来存储
  - 有序数组
  - 节约内存

**常用命令**

- `sadd key e`

- `srem key e`

- `scard`：集合大小

- `sismember`：判断存在

- `sdiff` / `sinter` / `sunion` + store destkey：将集合操作的结果存入destkey
- `smembers`：所有元素，慎用

- `srandmember`：随机元素



**应用**

- Tagging：`sadd`

- Random Item：`spop` / `srandmember`

- Social Graph：`sadd` / `sinter`

- 保存中奖 ID

- 用户数聚合

  - 累计用户集合：`SUNIONSTORE set_all set_all set_20200803`

  - 新增用户：`SDIFFSTORE set_new set_alll set_20200803`
  - 留存用户：`SINTERSTORE set_rem set_20200802 set_20200803`



## || zset

功能

- 类似 SortedSet + HashMap 
- 类似于 Java 的 SortedSet 和 HashMap 的结合体，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以给每个 value 赋予一个 score，代表这个 value 的排序权重。它的内部实现用的是一种叫做「跳跃列表」的数据结构。

- ```shell
  zadd books 9.0 "think in java"
  ```

  

原理：

- hash: value -> score

- hash 结构来存储 value 和 score 的对应关系

**内部编码**

- skiplist
  - 常规只有 数组 才支持二分查找，如何让 List 支持二分查找？
  - 跳表！新插入元素的层数：随机决定
  - 为什么不用红黑树？--> 红黑树不好根据范围查找

- ziplist
  - 元素个数较小时，用ziplist节约空间
  - value / score 被存成两个相邻的entry



**常用命令**

- `zadd key score e`

- `zscore key e`

- `zrem key e`

- `zremrangebyrank key start end` ：按key范围删除
- `zremrangebyscore key min-score max-score`：按分数范围删除

- `zincrby key score e`

- `zcard`：元素总数

- `zcount key min-score max-score`：元素个数



- `zrange key start end [withscores]`：范围查找

- `zrevrange`：反序查找
- `zrangbyscore key min-score max-score`：按分数查找

- `zrank`：排名
- `zinterstore / zunionstore`：集合



**应用**

- 帖子排序、帖子评论ID列表

- 粉丝列表 + 关注时间排序

- 学生成绩排序

- 延时队列：以到期时间为score



## || rax

功能：

- 类似TreeMap，按key排序
- hash + 按score排序 = zset

- hash + 按key排序 = rax



**应用**

- 居民档案：key = 地区

- 时间序列：key = 时间戳

- redis内部应用

  - Stream结构中存储消息队列：key = 时间序列 + 序号

  - Cluster 槽位和key对应关系的存储：key = hashslot + key

  

# | 应用

### 分布式锁

#### 命令

##### setNX + expire

###### 问题：两条指令，非原子

##### set k v EX 5 NX

#### 问题

##### 超时问题

如果执行任务时间较长，超时后还未执行完、又被其他线程抢去锁，则相当于有两个线程持有锁

###### 思路

####### 保证锁不会被其他线程释放

###### 解决

####### set value为随机数，del 时先判断value是否一致

####### 注意：del与判断value仍然不是原子操作，需要Lua支持

##### 可重入性

###### 需要包装客户端的 set 方法

###### 用 ThreadLocal 存储 Map<lockName, count>

##### 集群问题

主节点挂掉时，从节点会取而代之，客户端上却并没有明显感知。

原先第一个客户端在主节点中申请成功了一把锁，但是这把锁还没有来得及同步到从节点，主节点突然挂掉了。然后从节点变成了主节点，这个新的节点内部没有这个锁，所以当另一个客户端过来请求加锁时，立即就批准了。

这样就会导致系统中同样一把锁被两个客户端同时持有

###### 主从 failover时，如果锁信息还没同步到从节点，则可能导致锁被两个客户端持有

###### Redlock算法

####### 过半节点加锁成功

####### 释放锁时，向所有节点发送 del

##### 客户端加锁失败的处理策略

###### 抛出异常，通知用户重试

###### sleep后重试

###### 转至延时队列，过一会再试

### 延时队列

#### 队列

##### lpush + rpop / rpush + lpop

##### 优化：阻塞读

###### brpop / blpop

#### 延时队列

##### 设计

###### zset, 以到期时间作为 score

##### 命令

###### 入队

####### zadd

###### 出队

####### zrangeByScore(System.currentTimeMillis)

####### zrem

######## 删除成功后再进行后续处理

######## zrem 成功，则相当于当前线程抢到任务

####### 优化

######## zrangeByScore + zrem 合成原子操作：lua

####### 问题

######## 不能阻塞读？只能轮询？

#### 消息广播

##### PubSub模块

###### 命令

####### 生产者发布

######## publish channel-name msg

####### 消费者订阅

######## subscribe/unsubscribe channel-name

######## 模式订阅：psubscribe *

####### 消费者拉取消息

######## 轮询 get_message()

######## 阻塞监听：listen()

###### 缺点

####### 消息未被持久化

####### 消费者宕机重连后，之前的消息都丢失

##### Stream

###### 设计

####### last_delivered_id

######## 服务端存储每个消费者组消费到哪条消息

####### pending_ids

######## PEL: Pending Entries List

######## 消费者存储已消费但未 ACK 的消息

######### 忘了 ACK 会导致 PEL 过大

######## 用来确保客户端至少消费了一次

####### 如何避免消息丢失

######## PEL ?

####### 如何实现高可用

######## 集群 或 哨兵

######## failover 时可能丢失极小部分数据

######### 其他数据结构也是一样

###### 命令

####### CRUD

######## XADD / XREAD

######### 插入 / 读取

######### 可设置定长 stream

########## XADD stream_name maxlen 3

########## 防止 Stream 消息过多

######## XGROUP / XREADGROUP

######### 创建、读取消费组

######## XDEL

######### 软删除

######## XRANGE

######## XLEN

######## DEL

######### 删除所有消息

######## XINFO

######### xinfo stream xx

######### xinfo groups xx

####### 消费

######## xread

######### 独立消费，无消费者组

######### xread count 2 streams xx

######### xread block 0  ...

########## 支持阻塞等待

####### 消费者组

######## xgroup

######### xgroup create xx grp_name 0-0

########## 从头消费

######### xgroup create xx grp_name $

########## 从尾部消费新消息

######## xreadgroup

######### xreadgroup GROUP grp_name consumer_name

######### 同样支持阻塞等待：block time

######## xack

######### xack stream_name group_name ID

###### 特点

####### 消息持久化

####### 类似 Kafka

### 位图 Bitmap

位图不是特殊的数据结构，它的内容其实就是普通的字符串，也就是 byte 数组。

我们可以使用普通的 get/set 直接获取和设置整个位图的内容，也可以使用位图操作 getbit/setbit 等将 byte 数组看成「位数组」来处理。

#### 数据结构

##### type: string, 最大512M

#### 命令

##### 存取

###### SETBIT k offset v

零存：`setbit s 4 1`
整存：`set s <string>`

###### GETBIT

整取：`get s`
零取：`getbit s 1`


##### 统计

###### BITCOUNT k [start end] 统计

注意，start/end 是字节索引，只能是 8 的倍数


##### 查找

###### BITPOS k targetBit [start] [end] 查找

##### 位操作

###### BITOP op destKey key1 key2 位运算

op:
- and
- or
- not
- xor

##### 魔术指令

###### bitfield操作多个位

####### BITFIELD k get u4 2

######## 从第3个位开始，取4个位，结果是无符号数(u) / 有符号数 (i)

####### BITFIELD k set u8 16 97

######## 从第17个位开始，设置接下来的 8 个位，用无符号数97 （字母a）替代

####### BITFIELD k incrby u4 2 1

######## 从第3个位开始，设置接下来的 4 位无符号数，+1

######## 可设置溢出策略子指令：overflow

`bitfield k overflow wrap incrby ...`

1. wrap 折返
增加过多会变成0
2. sat 截断
增加过多会停留在最大值
3. fail 失败
报错

#### 应用

##### 存储用户一年中的签到记录

### HyperLogLog

#### 作用

##### HyperLogLog 提供不精确的去重计数方案

##### 极小空间完成独立数量统计

###### How?

https://www.jianshu.com/p/55defda6dcd2

##### type: string

#### 场景

##### PV 统计

###### incrby

###### key: pageId + date

##### UV 统计

###### set 存储访问用户ID?

####### 数据量太大

###### 优化：HyperLogLog

#### 缺点

##### 有错误率 0.81%

##### 不能返回单条元素

#### 命令

##### 添加：pfadd

###### pfadd key e1 e2...

##### 计数：pfcount

###### pfcount key

##### 合并：pfmerge

###### pfmerge destKey sourceKey1 sourceKey2

###### 可用于合并两个网页的UV统计

##### 不能判断是否存在，没有 pfcontains !!!

### 布隆过滤器

布隆过滤器能准确过滤掉那些已经看过的内容，那些没有看过的新内容，它也会过滤掉极小一部分 (误判)

#### 场景

##### 推荐时去重

##### 爬虫对爬过的URL去重

##### HBase过滤掉不存在的row请求

#### 命令

##### bf.exists / bf.mexists

##### bf.add / bf.madd

##### bf.reserve

自定义布隆过滤器参数

###### key

###### error_rate

####### 误判率越低，需要的空间越大

###### initial_size

####### 预计放入的元素数量

#### 原理

##### 参数

###### m 个二进制向量

###### n 个预备数据

###### k 个哈希函数

##### 构建

###### n个预备数据，分别进行k个哈希，得出offset，将相应位置的二进制向量置为1

##### 判断

###### 进行k个哈希，得出offset，如果全为1，则判断存在

#### 误差

##### 返回存在时，实际可能不存在

##### 误差率

###### 与 k (哈希函数)个数成反比

###### 与 n (预备数据)个数成正比

###### 与 m (二进制向量)长度成反比

### GeoHash

GeoHash 算法将二维的经纬度数据映射到一维的整数，这样所有的元素都将在挂载到一条线上，距离靠近的二维坐标映射到一维后的点之间距离也会很接近。

#### 场景

##### 用于地理经纬度计算

#### 设计

##### 初级：RDB存储经纬度、限制矩形区域、全量计算距离、排序

##### 进阶：将二维经纬度 映射到一维整数

###### 二刀切分成四块：00, 01, 10, 11

###### 如此往复，将所有坐标编程整数

###### 再将整数做base32编码，hash，作为zset的score存储

###### 查询：http://geohash.org

##### 底层类型: zset

###### value = key

###### score = geohash

####### 并没有使用新的数据结构！！！

#### 问题

##### 数据都是存到同一个key下，量大的时候集群迁移时会卡顿

###### 解决

####### 用单独实例部署

####### 拆分

#### 命令

##### GEOADD 添加

###### GEOADD key longitude latitude member

##### GEOPOS 获取

###### GEOPOS key member

##### GEODIST 距离

###### GEODIST key member1 member2 [unit]

##### GEORADIUS 范围

###### GEORADIUS key longitude latitude 20 km withdist count 3 asc

###### GEORADIUSBYMEMBER key member 20km withhash count 3 desc

##### ZREM 删除

###### ZREM key member

##### geohash

### 限流

#### 简单限流

##### 设计

###### zset 实现滑动时间窗口

###### 存储设计

####### key

######## clientId-actionId

####### value

######## ms

####### score

######## ms

##### 实现 isActionAllowed

```
public boolean isActionAllowed(String userId, String actionKey, int period, int maxCount) {
  String key = String.format("hist:%s:%s", userId, actionKey);
  long nowTs = System.currentTimeMillis();
  
  Pipeline pipe = jedis.pipelined();
  pipe.multi();
  
  //1. 记录行为
  pipe.zadd(key, nowTs, "" + nowTs);
  
  //2. 移除窗口之前的记录
  pipe.zremrangeByScore(key, 0, nowTs - period * 1000);
  
  //3. 记录窗口内记录数
  Response<Long> count = pipe.zcard(key);
  
  //4. 设置zset过期时间
  pipe.expire(key, period + 1);
  pipe.exec();
  pipe.close();
  
  //5. 比较数量是否超标
  return count.get() <= maxCount;
}

public static void main(String[] args) {
  Jedis jedis = new Jedis();
  SimpleRateLimiter limiter = new SimpleRateLimiter(jedis);
  for(int i=0;i<20;i++) {
 System.out.println(limiter.isActionAllowed("laoqian", "reply", 60, 5));
    }
  }
```

###### 1. 记录行为

####### zadd(key, nowTs, nowTs)

###### 2. 删除窗口外记录

####### zremrangeByScore(key, 0, nowTs - period * 1000)

###### 3. 窗口内记录数

####### zcard(key)

###### 4. 设置过期

####### expire(key, period + 1)

###### 5. 判断记录数

####### 记录数 <= maxCount

##### 注意

###### 使用 pipeline 提升效率，因为是对同一个key操作

###### 缺点：如果maxCount很大，则会消耗大量空间

#### 漏斗限流

##### 初步实现

```
public class FunnelRateLimiter {

 static class Funnel {
  int capacity;
  float leakingRate;
  int leftQuota;
  long leakingTs;

  void makeSpace() {
    long nowTs = System.currentTimeMillis();
    //1. 距离上次漏出过去了多久
    long deltaTs = nowTs - leakingTs;
    //2. 腾出的空间
    int deltaQuota = (int) (deltaTs * leakingRate);
    
    if (deltaQuota < 0) { // 间隔时间太长，整数数字过大溢出
      this.leftQuota = capacity;
      this.leakingTs = nowTs;
      return;
    }
    if (deltaQuota < 1) { // 腾出空间太小，最小单位是1
      return;
    }
    
    //3. 增加剩余空间
    this.leftQuota += deltaQuota;
    this.leakingTs = nowTs;
    // 剩余空间不得高于容量
    if (this.leftQuota > this.capacity) {
      this.leftQuota = this.capacity;
    }
  }

  boolean watering(int quota) {
    makeSpace();
    if (this.leftQuota >= quota) {
      this.leftQuota -= quota;
      return true;
    }
    return false;
  }
}

private Map<String, Funnel> funnels = new HashMap<>();

public boolean isActionAllowed(String userId, String actionKey, int capacity, float leakingRate) {
  String key = String.format("%s:%s", userId, actionKey);
  Funnel funnel = funnels.get(key);
  
  if (funnel == null) {
    funnel = new Funnel(capacity, leakingRate);
    funnels.put(key, funnel);
  }
  return funnel.watering(1); // 需要1个quota
}
}
```

###### capacity

####### 漏斗容量

###### leakingRate

####### 漏出速率

###### leftQuato

####### 漏斗剩余空间

###### leakingTs

####### 上次漏水事件

##### Redis-Cell模块

###### cl.throttle key capacity count period 1

###### 返回值

####### 1. 是否拒绝

####### 2. 漏斗容量

####### 3. 漏斗剩余空间

####### 4. 如果被拒绝，应该sleep多久

####### 5. 多久后漏斗完全空出

### 搜索key

#### keys

- 没有 offset、limit 参数，一次性吐出所有满足条件的 key。
- keys 算法是遍历算法，复杂度是 O(n)

##### 示例

###### keys *

###### keys code*hole

##### 缺点

###### 一次返回所有

###### O(n)，量大时会导致卡顿

#### scan

scan <cursor> match <regex> count <limit>
在 Redis 中所有的 key 都存储在一个很大的字典中，scan 指令返回的游标就是第一维数组的位置索引，limit 参数就表示需要遍历的槽位数


##### 示例

###### SCAN <cursor> match <regex> count <limit>

##### 参数

###### cursor

####### 游标，每次查询’返回下一个

###### key正则模式

###### limit hint

####### 不精确的限制

####### 用于限定服务器单次遍历的字典槽位数 (Map Entry[] 的索引)

##### 遍历指定容器集合

###### zscan

###### hscan

###### sscan

##### 注意rehash时的处理

###### 按高位进位加法来遍历

####### 保证 rehash 后的操作在遍历顺序上是相邻的

###### 渐进式 rehash

####### 要同时访问新旧两个数组

### 排行榜：zset

ZADD user_score score ui

#### 完整排行榜

##### ZREVRANGE user_score 0 -1 WITHSCORES

#### 前N排行榜

##### ZREVRANGE usre_score 0 N

#### 查询某人的分数

##### ZSCORE user_score uid

#### 查询某人的排名

##### ZREVRANGE user_score uid

#### 更新分数

##### ZINCRBY user_score delta uid

#### 增加删除人

##### ZREM user_score uid

##### ZADD user_score score uid

## 原理

### AP: 最终一致性

- Redis 的主从数据是 **异步同步** 的，所以分布式的 Redis 系统并不满足「一致性」要求。
- Redis 保证「最终一致性」，从节点会努力追赶主节点，最终从节点的状态会和主节点的状态将保持一致。

#### 异步同步

#### 可用性  +  最终一致性

#### 可设置强一致性：wait N t

##### N: 等待wait之前的所有写操作 同步到 N 个从节点

##### t: 最多等待 t 时间

### 通讯协议

#### RESP, Redis Serialization Protocal

#### 开头字符

##### +

###### 单行字符串

##### $

###### 多行字符串

##### :

###### 整数值

##### -

###### 错误消息

##### *

###### 数组

#### 特点

##### 大量冗余换行符

##### 浪费流量，但性能依然高

### 线程 IO 模型

#### 单线程

##### 那为什么快？

###### 纯内存操作

###### 避免频繁的上下文切换

###### IO 多路复用机制

##### 注意单个命令不能执行太长，会卡顿

#### 多路复用（事件轮询）

##### 指令队列

Redis 会将每个客户端 Socket 都关联一个指令队列。

客户端的指令通过队列来排队进行“顺序处理”，先到先服务。

##### 响应队列

Redis 服务器通过响应队列来将指令的返回结果回复给客户端。

- 如果队列为空，那么意味着连接暂时处于空闲状态，不需要去获取写事件，也就是可以将当前的客户端描述符从write_fds里面移出来。等到队列有数据了，再将描述符放进去。避免select系统调用立即返回写事件，结果发现没什么数据可以写。出这种情况的线程会飙高 CPU。

##### epoll事件轮询API

最简单的事件轮询 API 是select函数，它是操作系统提供给用户程序的 API。
输入是读写描述符列表read_fds & write_fds，输出是与之对应的可读可写事件。

同时还提供了一个timeout参数，如果没有任何事件到来，那么就最多等待timeout时间，线程处于阻塞状态。

一旦期间有任何事件到来，就可以立即返回。时间过了之后还是没有任何事件到来，也会立即返回。拿到事件后，线程就可以继续挨个处理相应的事件。处理完了继续过来轮询。于是线程就进入了一个死循环，我们把这个死循环称为事件循环，一个循环为一个周期。

###### select 函数

###### epoll 

###### 基于事件的回调机制

#### 异步处理线程

##### 作用

###### 除了主线程，还有几个异步线程专门用来处理耗时操作

##### 原理

###### 操作后，将key的操作包装成一个任务，塞进“异步任务队列”

###### 后台线程从中取任务

##### 案例

###### unlink 删除key

####### del 删除大key时会卡顿

####### 而 unlink 则通过异步线程回收内存

###### flush async

####### 异步清空

###### AOF sync

####### ？

###### 渐进式 rehash 定时任务触发

### pipeline

客户端通过改变了读写的顺序带来的性能的巨大提升.

``` 
Pipeline pl = jedis.pipelined();
loop pl.hset("key", "field", "v");
pl.syncAndReturnAll()
```

#### 目的

##### 节省网络开销

#### 手段

##### 客户端连续发送请求，而不是每发一个就等回复

#### 注意

##### 注意每次 pipeline 携带的数据量

##### 注意m操作与 pipeline 的区别：原子 vs 非原子

##### pipeline 每次只能作用在一个redis节点上

#### pipeline为什么快？

##### 发送请求后，连续的write 写入缓冲区立即返回，不消耗IO

##### 读取回复时，等待一个网络来回，后续的 read 则直接从缓冲区拿数据

### 事务

#### 命令

##### MULTI / EXEC / DISCARD

- 因为 Redis 的单线程特性，它不用担心自己在执行队列的时候被其它指令打搅，可以保证他们能得到的「原子性」执行

```
MULTI

hmset user:001 hero 'zhangfei' hp_max 8341 mp_max 100
hmset user:002 hero 'guanyu' hp_max 7107 mp_max 10

EXEC

```

###### 执行 EXEC 之前，所有指令保存到服务端 “事务队列”

####### 此时服务端返回QUEUED，而不是OK

###### 执行 EXEC 之后，一次性返回所有指令的运行结果

###### 执行DISCARD，取消

##### WATCH

服务器收到了 exec 指令要顺序执行缓存的事务队列时，Redis 会检查关键变量自 watch 之后，是否被修改了 (包括当前事务所在的客户端)。如果关键变量被人动过了，exec 指令就会返回 null 回复告知客户端事务执行失败，这个时候客户端一般会选择重试。

> WATCH books
> incr books  # 客户端2 修改
> (integer) 1

> MULTI
> incr books  # 客户端1 修改
> QUEUED
> EXEC  # 事务执行失败
> (nil)

###### 作用

####### 如果所watch的关键变量被改动，则不执行事务，返回错误。客户端一般可选择重试

####### WATCH + MULTI 实现乐观锁

###### 问题：乐观锁不适用于更新频繁的场景

####### Redis + Lua 原子化执行多条语句

local current

current = redis.call("incr",KEYS[1])
if tonumber(current) == 1 then
   redis.call("expire",KEYS[1],1)
end

####### 注意Lua中不要有耗时操作

###### 示例代码

```
jedis.watch(key);

Transaction tx = jedis.multi();
tx.set(...)
tx.set(...)
List response = tx.exec()

```

##### 结合pipeline

较少网络IO

pipe = redis.pipeline(transaction=true)

pipe.multi()
pipe.incr("books")
pipe.incr("books")

values = pipe.execute()

###### pipeline.multi()

###### pipeline.incr()

###### pinpeline.execute()

#### 特性

##### 隔离性

##### 不能保证原子性

Redis 的事务根本不能算「原子性」，而仅仅是满足了事务的「隔离性」，隔离性中的串行化——当前执行的事务有着不被其它事务打断的权利。

###### 第一个命令执行失败，后续命令仍然会被执行

### 内存回收策略

#### 缓存过期策略

https://redis.io/topics/lru-cache

##### 定时扫描

###### 设置了过期时间的key，会放入独立的字典中

###### 每秒扫描10次

###### 贪心策略

####### 目的

######## 避免遍历所有key

####### 过程

######## 从“过期字典”中随机 20 个 key；

######## 删除这 20 个 key 中已经过期的 key；

######## 如果过期的 key 比率超过 1/4，那就重复步骤 1；

######## 扫描时间上限：25ms

####### 问题

######## 客户端请求可能会因此多等待25ms，若此时客户端设置的超时时间短，则会timeout

######## 如果正好有大批key过期，可能导致大量timeout；而且不好排查

######## 解决

######### 过期时间随机化

##### 惰性策略

###### 访问key时，检查过期时间

###### 能否只用惰性？

####### 可能导致太多key没被删除

##### 从节点过期策略

###### 被动过期

###### 等待主节点 AOF 里的del指令

##### 原理

###### 过期字典

####### key = 数据key

####### value = 过期时间

#### 内存溢出控制策略

##### 配置

###### maxmemory-policy

##### 策略

###### 不删除，拒绝写入

####### noeviction

######## 默认策略，拒绝写入操作

######## 这个策略作为默认值是否合适？导致数据不可更新

###### 删除有过期时间的key

####### volatile-lru

######## LRU算法删除 有expire的key

####### volatile-ttl

######## 删除最近将要过期key

####### volatile-random

######## 随机删除过期key

####### volatile-lfu

######## LFU算法

###### 删除所有key

####### allkeys-lru

######## LRU算法删除所有key

####### allkeys-random

######## 随机删除所有key

####### allkeys-lfu

######## LFU 算法

#### eviction 算法

https://redis.io/topics/lru-cache

##### LFU: Least Frequently Used

4.0 新加入

##### LRU: Least Recently Used

当字典的某个元素被访问时，它在链表中的位置会被移动到表头。

所以链表的元素排列顺序就是元素最近被访问的时间顺序。

位于链表尾部的元素就是不被重用的元素，所以会被踢掉。

- 缺点：需要大量的额外的内存


###### 附加链表，头部表示刚被访问的元素

###### 要消耗大量额外内存

##### 近似LRU

- **随机**采样出 5个 key(可配置 maxmemory-samples 5) ，
- 然后淘汰掉最旧的 key，
- 如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于 maxmemory 为止。

Redis给每个 key 增加了一个额外的小字段，这个字段的长度是 24 个 bit，也就是最后一次被访问的时间戳。



###### 无需额外内存

###### 原理

####### 每个key增加额外字段：最近被访问的时间

####### 超出maxmemory时，随机采样5个key，淘汰掉最旧的；直至低于maxmemory

### 内部存储结构

#### redisObject

##### type

string
hash
list
set
sorted set


###### 数据类型

##### encoding

raw
int
ziplist
linkedlist
hashmap
intset



###### 编码方式

##### ptr

###### 数据指针

##### vm

###### 虚拟内存

##### 其他

#### 字典结构

##### 类似 HashMap, 数组 + 链表

##### 扩容 Rehash

###### 触发

####### 当LoadFactor达到阈值，则重新分配一个 2倍大小的数组

###### 重新计算元素位置

####### 将元素 hash 值对数组长度进行取模运算

####### 数组长度为2的n次方 --> 取模 等价于 按位于操作

####### a mod 8 = a & (8 - 1) = a & 7

######## 7 即为 mask，作用是保留hash值的低位

####### 即，每个槽位链表中约一半元素还是会放在当前槽位

###### 故，若按高位进位加法的遍历顺序，rehash后的槽位在遍历顺序上是相邻的

####### scan 命令的遍历！！！

#### 小对象压缩

##### hashtable --> skiplist

###### 针对 hash / zset 

##### hashtable --> intset

###### 针对 整数set

## 持久化

### RDB

全量备份


#### 触发条件

##### SAVE 命令

###### 同步

###### 阻塞客户端命令

###### 不消耗额外内存

##### BGSAVE 命令

###### 异步

###### 不阻塞客户端命令

####### 但 fork 执行瞬间是阻塞主线程的

###### fork子进程，消耗内存

子进程名：redis-rdb-bgsave


##### 配置文件: save seconds changes

save 900 1

save after 900 seconds if there is at least 1 change to the dataset

###### BGSAVE

###### 不建议打开

##### SHUTDOWN 命令

##### 从节点 SYNC 时 (=BGSAVE)

#### 原理

##### fork子进程生成快照


- 调用 glibc 的函数fork产生一个子进程，快照持久化完全交给子进程来处理，父进程继续处理客户端请求。

###### 流程

####### 父进程继续处理客户端请求

####### redis-rdb-bgsave 子进程进行持久化

###### 内存越大，耗时越长

###### info: latest_fork_usec

##### COW (Copy On Write)

使用操作系统的 COW 机制来进行数据段页面的分离。
当父进程对其中一个页面的数据进行修改时，会将被共享的页面复制一份分离出来，然后对这个复制的页面进行修改。这时子进程相应的页面是没有变化的，还是进程产生时那一瞬间的数据。

###### 父进程修改数据时，会将被共享的page复制一份，分离出来

###### 会导致内存增长

####### 但redis修改的数据量比例小，所以内存增长不会超过2倍

###### 子进程数据保持不变，所以称为“快照”

#### 缺点

##### 不可控、会丢失数据

##### 耗时 O(n)、耗性能、耗内存

### AOF

增量备份

- 先执行指令才将日志存盘.
- 可用 `bgrewriteaof` 指令对 AOF 日志进行瘦身。

#### 触发条件

##### always

##### every second

##### no

#### 原理

##### 先执行指令，再存储到AOF文件

##### 写命令刷新到缓冲区

##### 每条命令 fsync 到硬盘AOF文件

#### AOF重写

##### bgrewriteaof 命令

###### 原理：类似bgsave, fork子进程重新生成AOF

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

##### 对比上次 fsync 时间，>2s 则阻塞

##### info: aof_delayed_fsync (累计值)

### 建议

#### 混合持久化

在 Redis 重启的时候，可以先加载 rdb 的内容，然后再重放增量 AOF 日志。比 AOF 全量文件重放要快很多。

#### 持久化操作主要在“从节点”进行

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

##### 读从库

###### LettuceClientConfiguration.readFrom(ReadFrom.SLAVE_PREFERRED)

#### 主从复制流程

##### 全量复制

###### 步骤

####### 1.【s -> m】psync runId offset

######## 首次：`psync ? -1`

####### 2.【m -> s】+FULLRESYNC {runId} {offset}

####### 3.【s】save masterInfo

####### 4.【m】bgsave / 写入 repl_back_buffer

######## 生成快照

####### 5.【m -> s】send RDB

####### 6.【m -> s】send buffer

######## 用于复制增量指令

####### 7.【s】flush old data

######## 避免之前数据的影响

####### 8.【s】load RDB

###### 首次同步

####### 从节点加入到集群时，必须先进行一次快照同步 （全量复制）！

###### 问题

####### 如果buffer太小，或rdb复制时间太长，会导致无法进行增量复制

####### 然后就会再次发起快照同步，陷入死循环

##### 部分复制

###### 步骤

####### 1.【s -> m】psync runId offset

####### 2.【m -> s】CONTINUE

####### 3.【m -> s】send partial data

###### 原理

####### 主节点上有个 “内存 buffer”，存储指令数据

######## 是个环形数组

######## 数组满了会被覆盖

######### 此时强制全量复制

####### 从节点拉取buffer数据，并反馈偏移量

###### 问题

####### buffer 过小就会导致指令被覆盖

####### buffer过大可能导致OOM

######## 设置 client-output-buffer-limit，超过则主库强制断开从库的连接

####### 从库过多时，主库生成RDB会很频繁

######## 主从级联模式

##### 两个buffer

###### replication buffer

####### 用于全量复制

####### 每个 client 对应一个 replication buffer

###### repl_backlog_buffer

####### 用于增量复制

####### 环形缓冲区

####### 所有从库共享

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

####### wait 指令

####### 等待wait之前的所有写操作 同步到 N 个从节点

###### 从节点故障

##### 主从配置不一致

###### maxmemory配置不一致

####### 可能丢失数据

master: maxmemory=4g
slave: maxmemory=2g

当数据>2g，slave会使用maxmemory-policy删除数据，failover之后的表现就是丢失数据。

###### 数据结构优化参数不一致

####### 内存不一致

##### 规避“全量复制”

###### 第一次全量复制

####### 不可避免

####### 优化：小主节点（小分片），低峰

###### 节点runId不匹配导致复制

####### 主节点重启后runId变化

####### 优化：故障转移（哨兵、集群）

###### 复制积压缓冲区不足

####### 网络中断后无法进行部分复制

####### 优化：rel_backlog_size（默认1m）

##### 规避“复制风暴”

###### 主节点重启，多个从节点复制

###### 优化：更换复制拓扑

### sentinel

#### 原理

##### 思路

###### sentinel持续监控主从节点状态；主挂掉，自动选择最优的从节点切换成主

###### 客户端连接集群时，先向sentinel询问主节点地址

##### 三个定时任务

###### 每1秒：心跳检测

####### sentinel 对其他 sentinel 和 redis 执行ping 

####### 失败判定依据

###### 每2秒：交换信息

####### sentinel 通过 master 的 channel 交换信息

######## master频道：__sentinel__:hello

####### 交换对节点的看法、以及自身信息

###### 每10秒：发现slave

####### sentinel 对 m/s 执行info

######## 发现slave节点

sentinel初始配置只关心master节点

####### 确认主从关系

##### 哨兵集群

######  sentinel 集群可看成是一个 ZooKeeper 集群

###### 集群内部如何通讯？

####### 利用 pub / sub 机制

######## 频道：__sentinel__:hello

####### 每个哨兵在主库上发布消息、订阅消息

###### 哨兵如何知道从库IP？

####### 向主库发送 info 命令

###### 哨兵如何与客户端同步信息？

####### 哨兵自身提供了 pub/sub 频道

`+sdown`: 实例进入“主观下线”
`-sdown`：实例退出“主观下线”
`+odown`：进入“客观下线”
`-down`：退出“客观下线”

`+slave-reconf-sent`：哨兵发送slaveof命令重新配置从库
`+slave-reconf-inprog`：从库配置了新主库，但尚未进行同步
`+slave-reconf-done`：从库配置了新主库，且完成同步

`+switch-master`：主库地址发生变化

#### 流程

##### 客户端流程

###### 【0. sentinel集合】 预先知道sentinel节点集合、masterName

###### 【1. 获取sentinel】遍历sentinel节点，获取一个可用节点

###### 【2. 获取master节点】get-master-addr-by-name masterName

###### 【3. role replication】获取master节点角色信息

###### 【4. 变动通知】当节点有变动，sentinel会通知给客户端 （发布订阅）

####### JedisSentinelPool -> MasterListener --> sub "+switch-master"

####### sentinel是配置中心，而非代理！

##### 故障转移流程

###### 【1. 故障发现】多个sentinel发现并确认master有问题

####### 主观下线

相关配置
```
sentinel monitor myMaster <ip> <port> <quorum>
sentinel down-after-milliseconds myMaster <timeout>
```

######## ping 响应超时

####### 客观下线

######## 超半数哨兵判断为 主观下线

######## 为了防止误判，防止无谓的主从切换

###### 【2. 选主】选举出一个sentinel作为领导

####### 原因

######## 只有一个sentinel节点完成故障转移

####### 实现

######## 通过sentinel is-master-down-by-addr命令竞争领导者

###### 【3. 选master】选出一个slave作为master, 并通知其余slave

####### 筛选

######## 从库当前在线状态

######## 从库之前的网络连接状态

####### 打分

######## 选slave-priority最高的

######## 选复制偏移量最大的

主库会用 master_repl_offset 记录当前的最新写操作在 repl_backlog_buffer 中的位置，而从库会用 slave_repl_offset 这个值记录当前的复制进度。

######### slave_repl_offset

######## 选runId最小的

####### 对这个slave执行slaveof no one

###### 【4. 通知】通知客户端主从变化

###### 【5. 老master】等待老的master复活成为新master的slave

####### sentinel会保持对其关注

#### 消息丢失

##### 场景

###### 主从切换后，从节点的数据不是最新的

##### 缓解

###### min-slaves-to-write 1

主节点必须至少有一个从节点在进行正常复制，否则就停止对外写服务，丧失可用性

####### 至少要有一个从节点正常复制，才返回结果

###### min-slaves-max-lag 10

如果 10s 没有收到从节点的反馈，就意味着从节点同步不正常

####### 如果10s未收到从节点反馈，则认为复制不正常

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

#### 原理

##### proxy 转发

##### 存储槽位关系

###### zookeeper

###### etcd

##### 扩容迁移

###### 增加 SlotScan 命令，扫描出制定槽位下的所有key，然后挨个迁移

###### 如果get key正好在迁移中的槽位

####### 则强制对当前key先完成迁移

####### 迁移后访问新redis实例

#### 代价

##### 不支持事务

##### rename 操作也很危险

###### rename 前后可能放到不同的redis实例

##### 为了支持扩容，单个 key 对应的 value 不宜过大

##### 网络开销更大

##### 需要运维zk

### cluster

#### 原理

##### 复制

###### 与 Master / Slave 一样

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

###### Slave默认是热备，不提供读写服务

##### 分片

###### slots

####### 16384个

######## 定位：crc16(key) % 16384

######## 计算槽位

######### cluster keyslot k

####### 槽位信息存储于每个节点中

######## Rax

`Rax slots_to_keys` 用来记录槽位和key的对应关系
- Radix Tree 基数树


######## 通过 Gossip 协议传播配置信息

######## 问题

######### 不适合大规模集群

######### gossip 协议传播速度慢

######## 解决

######### 代理

- 转发请求
- 监控集群状态、负责主从切换
- 维护集群元数据（槽位映射xin'x）

########## Codis/ twemproxy

######### 客户端维护元数据

####### 每个节点通过meet命令交换槽位信息

###### 伸缩

####### 迁移slot (同步)

- Redis 迁移的单位是槽，当一个槽正在迁移时，这个槽就处于中间过渡状态。这个槽在原节点的状态为`migrating`，在目标节点的状态为`importing`，  


- 迁移过程是同步的，在目标节点执行`restore指令`到原节点删除key之间，原节点的主线程会处于阻塞状态，直到key被成功删除。 >> 要尽可能避免大key

原理：
- 以原节点作为目标节点的「客户端」
- 原节点对当前的key执行dump指令得到序列化内容，然后发送指令restore携带序列化的内容作为参数
- 目标节点再进行反序列化就可以将内容恢复到目标节点的内存中
- 原节点收到后再把当前节点的key删除


######## 步骤

######### dump

######### restore

######### remove

######## 状态

####### 迁移slot过程中如何同时提供服务？

######## - ASK

###### 故障转移

####### 故障发现

######## 通过ping/pong发现故障

######## PFAIL 主观下线

- node1 发送ping消息
- node2 回复pong消息
- node1 收到pong，并更新与node2的 `最后通信时间`
- cron定时任务：如果最后通信时间超过node-timeout，则标记为 `pfail`

######## FAIL 客观下线

- 接受ping
- 消息解析：其他pfail节点 + 主节点发送消息
- 维护故障链表
- 尝试客观下线：计算有效下线报告数量
- if > 槽节点总数一半，则更新为客观下线；
-并向集群广播下线节点的fail消息。

######### 当半数以上主节点都标记其为pfail

####### 故障恢复

######## 资格检查

每个从节点：检查与主节点断线时间；

如果大于 
`cluster-node-timeout` * 
`cluster-slave-validity-factor`，则取消资格

######## 准备选举时间

offset越大，则延迟选举时间越短


- slave通过向其他master发送FAILOVER_AUTH_REQUEST消息发起竞选，master回复FAILOVER_AUTH_ACK告知是否同意。

######## 选举投票

收集选票，如果大于 N/2 + 1，则可替换主节点

######## 替换主节点

1. slaveof no one
2. clusterDelSlot撤销故障主节点负责的槽；
3. clusterAddSlot把这些槽分配给自己；
4. 向集群广播pong消息，表明已经替换了故障节点

##### 一致性: 保证朝着epoch值更大的信息收敛

保证朝着epoch值更大的信息收敛: 每一个Node都保存了集群的配置信息`clusterState`。

- `currentEpoch`表示集群中的最大版本号，集群信息每更新一次，版本号自增。
- nodes列表，表示集群所有节点信息。包括该信息的版本epoch、slots、slave列表

配置信息通过Redis Cluster Bus交互(PING / PONG, Gossip)。
- 当某个节点率先知道信息变更时，将currentEpoch自增使之成为集群中的最大值。
- 当收到比自己大的currentEpoch，则更新自己的currentEpoch使之保持最新。
- 当收到的Node epoch大于自身内部的值，说明自己的信息太旧、则更新为收到的消息。


#### 客户端

##### 连接

###### 可只连接一个节点地址，其他地址可自动由这个节点来发现

###### 连接多个安全性更好：否则如果挂了，需要修改地址

###### redis-cli -c 会自动跳转到新节点

##### 槽位迁移感知

###### ASK

####### 流程

######## 0.先尝试访问源节点

######## 1.源节点返回ASK转向

######## 2.向新节点发送 ASKING 命令

在迁移没有完成之前，这个槽位还是不归新节点管理的，它会向客户端返回一个`-MOVED`重定向指令告诉它去源节点去执行。如此就会形成 `重定向循环`。
asking指令的目标就是打开目标节点的选项，告诉它下一条指令不能不理，而要当成自己的槽位来处理。

######### 否则不处理而返回MOVED。避免循环重定向

######## 3.向新节点发送命令

####### 作用

######## 迁移中的slot

###### MOVED

####### 流程

######## 1.向任意节点发送命令

######## 2.节点计算槽和对应节点

######## 3.如果指向自身，则执行命令并返回结果

######## 4.如果不指向自身，则回复 -moved (moved slot ip:port)

######## 5.客户端重定向发送命令，并更新本地槽位信息缓存

####### 作用

######## 表示 slot 不在当前节点，重定向到正确节点

###### moved vs. ask

####### 都是客户端重定向

####### MOVED: 表示slot确实不在当前节点（或已确定迁移）

######## 会更新客户端缓存

####### ASK: 表示slot在迁移中

######## 不会更新客户端缓存

##### 集群变更感知

###### ConnectionError --> 目标节点挂了

####### 抛出 ConnectionError

####### 随机选节点重试，返回 MOVED 指令告知新节点地址

###### ClusterDownError --> 手动主从切换

####### 访问旧节点会返回 ClusterDownError

####### 客户端关闭连接、清空槽位映射关系表

####### 待下一条指令，重新初始化节点信息

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

######### why?

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

####### READONLY命令：强制slave读

- 默认情况下，某个slot对应的节点一定是一个master节点。客户端通过`MOVED`消息得知的集群拓扑结构也只会将请求路由到各个master中。

- 即便客户端将读请求直接发送到slave上，slave也会回复MOVED到master的响应。

- 客户端向slave发送 READONLY 命令后，slave对于读操作将不再返回moved，而是直接处理。

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

#### 命令

##### 创建

###### 原生

####### 配置文件：cluster-enabled yes

cluster-enabled yes

cluster-config-file "xx.conf"
cluster-require-full-coverage no
cluster-node-timeout 15000

####### 启动: redis-server *.conf

####### gossip通讯：cluster meet ip port

####### 分配槽(仅对master)：cluster addslots {0...5461}

####### 配置从节点：cluster replicate node-id

###### 脚本

####### 安装ruby

####### 安装rubygem redis

####### 安装redis-trib.rb

###### 验证

####### cluster nodes

####### cluster info

####### cluster slot

####### redis-trib.rb info ip:port

##### 扩容

###### 准备新节点

###### 加入集群

####### meet

####### redis-trib.rb add-node

redis-trib.rb add-node new_host:new_port existing_host:existing_port --slave --master-id

###### 迁移槽和数据

####### 手工

######## 1_对目标节点：cluster setslot {slot} importing {sourceNodeId}

######## 2_对源节点：cluster setslot {slot} migrating {targetNodeId}

######## 3_对源节点循环执行：cluster getkeysinslot {slot} {count}，每次获取count个键

######## 4_对源节点循环执行：migrate {targetIp} {targetPort} key 0 {timeout}

0: db0


######## 5_对所有主节点：cluster setslot {slot} node {targetNodeId}

####### pipeline migrate

####### redis-trib.rb reshard

redis-trib.rb reshard host:port
--from
--to
--slots

host:port是任一个节点的

##### 收缩

###### 迁移槽

###### 忘记节点

####### cluster forget {downNodeId}

####### redis-trib.rb del-node

redis-trib.rb del-node ip:port {downNodeId}

###### 关闭节点

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

## 运维

### 内部结构

#### 查看元素内部编码

##### debug object key

##### 查看 encoding 字段

### 内存

#### 内存回收

##### 无法保证立即回收已经删除的 key 的内存

##### 但会重新使用未回收的空闲内存

##### flushdb

###### 删除所有key

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

####### + maxmemory_policy

###### config rewrite

##### 动态调整内存上限

##### 序列化与压缩

###### 拒绝Java原生

###### 推荐protobuf, kryo, snappy

### info 命令

#### 分类

##### server

##### clients

##### memory

##### persistence

##### stats

##### replication

##### cpu

##### cluster

##### keyspace

#### 用例

##### info stats | grep ops

###### 查询每秒执行指令次数

###### 然后用 `monitor` 命令观察哪些key被频繁访问

##### info stats | grep sync

###### 查询同步失败次数

###### 如果 sync_partial_err 失败次数过多，考虑扩大backlog缓冲区

##### info clients

###### 查询连接了多少客户端

###### 然后用 `client list` 命令列出客户端地址

##### info memory

###### 查询内存占用

##### info replication | grep backlog

###### 查询复制积压缓冲区大小

### 保护

####  spiped: SSL代理

##### 加密

#### 设置密码

##### server: requirepass / masterauth

##### client: auth命令 、 -a参数

#### rename-command flushall ""

##### 不支持config set动态配置

#### bind 内网IP

### 性能

#### slowlog 慢查询

##### 配置

###### slowlog-max-len

####### 先进先出队列、固定长度、内存

####### 默认10ms, 建议1ms

###### slowlog-log-slower-than

####### 建议1000

##### 命令

###### slowlog get [n]

###### slowlog len

###### slowlog reset

#### latency-monitor

https://redis.io/topics/latency-monitor

##### 配置

###### latency-monitor-threshold=5ms

##### 命令

###### LATENCY LATEST

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

### 客户端

#### 连接池

##### 连接数

###### maxTotal

####### 如何预估

例如：
- 一次命令耗时1ms，所以一个连接QPS=1000;
- 业务期望QPS = 50000;

> 则 maxTotal = 50000 / 1000 = 50

######## 业务希望的 redis 并发量

######## 客户端执行命令时间

######## redis 资源：应用个数 * maxTotal < redis最大连接数

###### maxIdle

####### 有坑

####### 应该 == maxTotal，如果太小 空闲连接会被丢弃

###### minIdle

##### 等待

###### blockWhenExhausted

资源用尽后，是否要等待；建议设成true


###### maxWaitMillis

##### 有效性检测

###### testOnBorrow 

###### testOnReturn

##### 监控

###### jmxEnabled

##### 空闲资源监测

###### testWhileIdle

###### timeBetweenEvictionRunsMillis

监测周期


###### numTestsPerEvictionRun

每次监测的采样数


#### RedisTemplate

##### 序列化

###### serializeValuesWith

###### 存储类型信息

####### ObjectMapper#enableDefaultTyping()
