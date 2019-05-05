# MySQL知识图谱

## 原理

### 更新

#### update

##### WAL: Write-Ahead-Logging

先写日志，再写磁盘。

InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里

###### 【执行器】先找【引擎】取出行

引擎逻辑：

- 如果这行本来就在内存中，直接返回给执行器；
- 否则，先从磁盘读入内存，再返回。

###### 【执行器】执行更新，【引擎】将新数据更新到内存

###### 【引擎】同时将更新操作记录到 redo log （此时处于prepare状态）

###### 【执行器】生成 bin log，并写入磁盘

###### 【执行器】调用引擎的提交事务接口，【引擎】提交redo log

####### 两阶段提交

redo log 先 prepare， 
再写 binlog，
最后commit redo log

##### redo log

###### InnoDB特有

###### 循环写

####### 默认4个文件，每个1G

####### write pos

######## 当前记录的位置

####### checkpoint

######## 当前要擦除的位置

###### 写入机制

####### innodb_flush_log_at_trx_commit

######## 0: 事务提交时，log写入redo log buffer

######## 1: 事务提交时，log写入disk 【推荐】

######## 2: 事务提交时，log写入FS page cache

####### 后台线程，每1秒将redo log buffer， write到FS page cache，然后fsync持久化到磁盘

##### binlog

###### Server共有

###### 追加写：不会覆盖旧日志

###### 格式

####### statement

####### row

###### 写入机制

####### 事务执行中：把日志写到binlog cache

####### 事务提交时：把binlog cache写到binlog 文件

######## write: 写入文件系统的page cache

######## fsync: 持久化到硬盘

######## sysnc_binlog

######### 0: 每次提交事务，只write不fsync

######### 1: 每次提交事务，都fsync 【推荐】

######### N: 每次提交事务，都write，累积N个后fsync

##### 脏页：内存数据页跟磁盘数据页内容不一致

###### flush: 写回磁盘

###### flush过程会造成抖动

###### 何时触发flush

####### redo log写满（此时系统不再接受更新，更新数跌成0）

####### 系统内存不足

####### 系统空闲时

####### mysql关闭时

#### insert

##### insert ... select

###### 常用于两个表拷贝

###### 对源表加锁 

###### 如果insert和select是同一个表，则可能造成循环写入

##### insert into t ... on duplicate key update d=100

###### 插入主键冲突的话，执行update d=100

### 查询

#### count(*)

##### MyISAM把总行数存在磁盘上，count(*)效率奇高

##### 效率：count(字段) < count(id) < count(1) = count(*)

count(字段)：满足条件的数据行里，字段不为null的总个数；

count(主键id)：引擎遍历表，取出id返回给server；server判断是否为空，按行累加；

count(1)：引擎遍历表，但不取值，server层对于返回的每一行 放入数字1；

count(*)：并不会取出全部字段，*肯定不是null,直接按行累加。



#### join

##### 分类

###### NLJ

####### Index Nested-Loop Join

####### 先遍历t1 --> 对于每一行，去t2走树搜索

######## 驱动表t1: 全表扫描

######## 被驱动表t2: 树搜索，使用索引！

####### 原则：应该让小表来做驱动表

####### 什么是小表？

######## 考虑条件过滤后、参与join的各个字段总数据量小

###### SNJ

####### Simple Nested-Loop Join

####### 当被驱动表t2不能使用索引时，扫描行数很大

###### BNL

####### Block Nested-Loop Join

####### 读取t1存入join_buffer --> 扫描t2，跟join_buffer数据作对比

######## 如果join_buffer小，则分段存入，驱动表应该选小表

######## 如果join_buffer足够，则驱动表大小无所谓

####### 原则：要尽量避免BNL

##### 优化

###### MRR: Multi-Range Read

####### 按照主键递增顺序查询，则近似顺序读，能提升读性能

###### BKA: Batched Key Access

####### 用于优化 NLJ 

####### 读取t1一批数据 --> 放到临时内存(join_buffer) --> 一起传给t2

###### BNL性能差

####### 多次扫描被驱动表，占用磁盘IO

####### 判断Join条件需要执行M*N次对比，占用CPU

####### BNL 大表Join后对Buffer Pool的影响是持续性的

######## 会占据Buffer Pool Yong区

######## 导致业务正常访问的数据页 没有机会进入Yong区

####### BNL转成BKA

######## 加索引

######## 使用临时表

### 聚合排序

#### order by

##### sort_buffer

如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助 --> sort_buffer_size

##### 分类

###### 全字段排序

无需回表


###### rowid排序

SET max_length_for_sort_data = 16;
只在buffer中放入id和待排序字段，需要按照id回表


##### 索引

###### 如果有索引：不需要临时表，也不需要排序

###### 进一步优化：覆盖索引

##### 随机

###### order by rand()

###### where id >= @X limit 1

#### group by

##### 机制

###### 内容少：内存临时表

###### 内容多：磁盘临时表

##### 优化

###### 索引

如果对列做运算，可用generated column机制实现列数据的关联更新。

alter table t1 add column z int generated always as(id0), add index(z)


###### 直接排序

select SQL_BIG_RESULT .. from ..
告知mysql结果很大，直接走磁盘临时表。

#### 临时表

##### 例子

###### union

(select 1000 as f)
union
(select id from t1 order by id desc limit 2);

- 创建内存临时表，字段f,是主键；
- 执行第一个子查询，放入1000；
- 执行第二个子查询，获取的数据插入临时表 （主键去重）
- 从临时表按行取出数据。

NOTE: 如果是Union ALL则不会用临时表，因为无需去重，直接依次执行子查询即可。


###### group by

select id as m, 
  count(*) as c,
from t1 
group by m;

创建临时表：字段m, 

## 索引

### 基础

#### 常见模型

##### 哈希表

###### value存在数组中，用哈希函数把key换算成index

###### 适用于等值查询，范围查询需要全部扫描

##### 有序数组

###### 查询方便，支持范围查询

###### 更新成本高

###### 适用于静态存储

##### 跳表

##### LSM树

##### 搜索树

###### B Tree

M阶B Tree: 
- 每个非叶子结点至多有M个儿子，至少有M/2个儿子；
- 根节点至少有2个儿子；
- 所有叶子节点在同一层。

###### B+ Tree

叶子节点才是真正的原始数据


###### 与二叉树的区别

- 二叉树：优化比较次数
- B/B+树：优化磁盘读写次数

#### 分类

##### 簇索引 -主键索引

每个表至多一个，一般为主键索引。
- B Tree上存储的就是数据本身

##### 非簇索引 -二级索引

- B Tree上存储的是主键
- 需要两次查找才能获取数据本身

##### 唯一索引 vs. 普通索引

###### 性能

####### 读取：类似

####### 更新：普通索引好

###### change buffer

当需要更新一个数据页时，
- 如果数据页在内存中就直接更新
- 而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB 会将这些更新操作缓存在 change buffer 中

这样就不需要从磁盘中读入这个数据页了。在

下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。

####### merge

######## 将change buffer中的操作应用到原数据页

######## 触发时机

######### 访问该数据页时

######### 后台线程定期merge

######### 数据库关闭时

####### 条件：不适用于唯一索引，只限于普通索引

####### 适用场景：`写多读少`

因为 merge 的时候是真正进行数据更新的时刻，而 change buffer 的主要目的就是将记录的变更动作缓存下来，

所以在一个数据页做 merge 之前，change buffer 记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。

####### vs. redo log

######## change buffer 主要节省的是随机读磁盘的 IO 消耗

######## redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写）

######## 写入change buffer后 也是要写redo log的

### 主键

#### 自增主键

##### 优点

###### 性能好

每次插入一条新记录，都是追加操作，都不涉及到挪动其他记录，也不会触发叶子节点的分裂

####### 插入是追加操作

####### 不挪动其他记录，不触发页分裂

###### 存储空间小

##### 机制

###### show create table命令，AUTO_INCREMENT=2 列出下一个自增值 (Y)

###### 如果插入值为X

####### 当X<Y，则自增值不变

####### 当X>=Y，则自增值更新

##### 不保证连续

插入数据时，是先更新自增值，再执行插入。
如果插入异常，自增值不回退。		

###### 由于主键冲突导致

###### 由于回滚导致

###### 自增值为何不能回退？

假设有并行事务A/B, A申请到id=2，B申请到id=3；
A回退，如果把id也回退为2，那么后续继续插入数据id=3是会冲突。

##### 自增锁

###### innodb_autoinc_lock_mode = 0

####### 语句执行结束才放锁

###### innodb_autoinc_lock_mode = 1

####### 普通insert: 申请自增锁后马上释放

####### insert...select: 语句执行结束才放锁

因为不知道会插入多少行，无法一次性申请id


###### innodb_autoinc_lock_mode = 2

####### 申请自增锁后马上释放

若此时binlog_format = statement，z对于insert...select语句会有数据一致性问题：
- insert into t2 select from t1, 的同时t1表在插入新数据。
- t2 id可能不连续
- 但同步到备库，id连续

###### 建议

####### innodb_autoinc_lock_mode = 2

####### binlog_format = row

####### 既能提升并发性，又不会出现数据一致性问题

#### 业务主键

##### 适用场景

###### 只有一个索引

###### 例如KV

### 操作

#### 覆盖索引

select primaryKey（而不是*） from xx where indexColumn=
非簇索引节点值为“primaryKey”，而非真实数据记录。避免回表



##### 1. select 主键

##### 2. 返回字段、查询条件 作为联合索引

#### 前缀索引

联合索引(a, b), 只查a也会索引

##### 字符串索引的最左M个字符

###### 字符串类型：用前缀索引，并定义好长度，即可既节省空间、又不用额外增加太多查询成本

如何定义长度:
alter table t add index i_2(email(6))


###### 如何找到合适的长度：区分度

select 
count(distinct email) as L,
count(distinct left(email, 4)) as L4,
count(distinct left(email, 5)) as L5,
...
from User;

假设可接受的损失比是5%，则找出不小于L*95%的值

##### 联合索引的最左N个字段

##### 联合索引的顺序：如果调整顺序可以少维护一个索引，则用该顺序；如不能，则考虑空间优先

空间有效：查询name+age, name, age
- 策略A: (name, age) (age)
- 策略B: (age, name) (name)

应选A, 因为age占用空间geng'x

##### 缺点

###### 前缀索引可能增加扫描行数

###### 无法利用覆盖索引

#### 索引下推

(name, age)
可以适用where name like `张%` AND age=10

直接过滤掉不满足条件age!=10的记录，减少`回表次数`。


### 索引失效的情况

#### 优化器选择索引

##### 标准

###### 扫描行数

####### 判断扫描行数的依据：区分度cardinality

`show index`可查

####### 区分度通过采样统计得出

####### analyze table t 重新统计索引信息

###### 是否使用临时表

###### 是否排序

###### 是否回表：主键索引无需回表

##### 选错索引的情况

###### 为了避免回表，使用了主键索引；而实际上用where条件中的索引会更快

##### 索引选择非最优时如何补救

###### force index强行选择一个索引

select * from t force index(a) where...

###### 修改语句，引导使用希望的索引

order by b limit 1 --> index b
order by b, a limit 1 --> index a

###### 新建合适的索引

#### 查询语句导致索引失效

##### 条件字段函数操作: where month(t_date) = 7

##### 隐式类型转换: where tradeid = 123 , tradeid is varchar

##### 隐式字符编码转换:  utf8 --> utf8mb4

## 锁

### 全局锁

#### 命令：Flush tables with read lock (FTWRL)

#### 场景：全库逻辑备份

当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（增删改）、数据定义语句（建表、修改表结构等）和更新类事务的提交语句

##### innodb一般用 `mysqldump -single-transaction`

此命令会在导数据之前启动一个事务，确保拿到一致性视图。
由于MVCC的支持，导出数据过程中可以正常进行更新cao'z。

#### 区别与 set global readonly=true

##### 一些系统会用readonly来判断是否从库

##### 如果客户端异常断开，readonly状态不会重置，风险高

#### 优化：innoDB可在可重复读隔离级别下开启一个事务

### 表级锁

#### 表锁（lock tables xx read/write）

#### 元数据锁（MDL, metadata lock)

##### 自动加上，无需显式使用

###### 当对一个表做增删改查操作的时候，加 MDL 读锁

###### 当要对表做做结构变更操作的时候，加 MDL 写锁；

##### 增加字段需要MDL写锁，会block业务

###### ALTER +timeout

ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 


###### 执行之前杀掉长事务

### 行锁

#### MyISAM不支持行锁

#### 两阶段锁协议

##### 行锁是在需要的时候才加上的

##### 但并不是不需要了就立刻释放，而是等到事务结束

#### 最佳实践

##### 要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放

#### 死锁

##### 等待超时

###### innodb_lock_wait_timeout

##### 死锁检测

 

###### innodb_deadlock_detect=on

###### 发现死锁后，主动回滚死锁链条中的某一个事务

###### 问题：耗费 CPU

当更新热点行时，每个新来的线程都要判断会不会由于自己的加入导致死锁。

###### 解决：服务端并发控制

对于相同行的更新，在进入引擎之前排队


## 事务

### select xx for update: 锁住行

### where stock=xx: 乐观锁

### 事务隔离

#### 实现原理

##### MVCC：多版本并发控制

###### 不同时刻启动的事务会有不同的read-view

###### 回滚日志: undo log

##### 查询数据：一致性读（consistent read view）

###### 秒级创建快照

####### 基于row trx_id实现`快照`

######## 事务transaction id 严格递增

######## 事务更新数据时，会把transaction id赋值给数据版本 row trx_id

####### 数据表中的一行记录，其实可能有多个版本（row），每个版本都有自己的row trx_id

###### 用于实现可重复读

####### 如果数据版本是我启动以后才生成的，我就不认，我必须要找到它的上一个版本

###### 根据 row trx_id 、一致性视图（read-view）确定数据版本的可见性

####### 每个事务有一个对应的数组，记录事务启动瞬间所有活跃的事务ID

######## 数组里事务ID最小值：低水位

######## 数组里事务ID最大值+1：高水位

####### 判断可见性

######## row trx_id < 低水位：表示是已提交的事务生成的，对当前事务可见

######## row trx_id > 高水位：表示是有将来启动的事务生成的，不可见

######## row trx_id 在两者之间：

######### 如果row trx_id在数组中，表示是有未提交的事务生成的，不可见

######### 如果row trx_id不在数组中，表示是已提交的事务生成的，可见

####### 判断可见性(简化)

######## 版本未提交：不可见

######## 版本已提交，但在视图创建后提交：不可见

######## 版本已提交，且在视图创建前提交：可见

##### 更新数据：当前读（current read ）

###### 更新数据都是先读后写的

####### 而这个读，只能读`当前的值`，即当前读current read

####### 并非一致性读！！不会去判断row trx_id

###### 当前读必须要读最新版本

####### 所以必须加锁

####### 如果此时有其他事务更新此行 但未提交，则行锁被占用，当前事务会等待锁

###### 除了 update 语句外，select 语句如果加锁，也是`当前读`

####### 共享锁：select xx lock in share mode;

####### 排它锁：select xx for update;

#### 隔离级别

##### Read Uncommitted

##### Read Committed

###### 查询只承认在`语句`启动前就已经提交完成的数据

###### 解决：脏读

##### Repeatable Read

###### 查询只承认在`事务`启动前就已经提交完成的数据

###### 解决：脏读、不可重复读

##### Serialized

###### 对相关记录加读写锁

###### 解决：脏读、不可重复读、幻读

## 运维

### 逻辑架构

#### Server层

##### 连接器

管理连接，权限验证


###### 推荐用长连接

####### 问题：长期积累可能OOM

因为执行过程中临时使用的内存是管理在连接对象里，直到连接断开才释放

####### 优化

######## 定期断开长连接

######## 每次执行比较大的操作后，执行mysql_reset_connection 重新初始化连接

##### 查询缓存

###### 表更新后，缓存即失效

###### 8.0 不再支持查询缓存

##### 分析器

###### 词法分析

###### 语法分析

##### 优化器

###### 生成执行计划

###### 选择索引

###### 选择表连接顺序

##### 执行器

###### 判断权限

###### 操作引擎

#### 存储引擎

##### InnoDB

###### 事务

InnoDB每一条SQL语言都默认封装成事务，自动提交，这样会影响速度，所以最好把多条SQL语言放在begin和commit之间，组成一个事务；



###### 不保存表的具体行数

##### MyISAM

###### 不支持事务

###### 不支持行锁

###### 不支持外键

###### 保存了整个表的行数

###### 支持全文索引

##### Memory

### 连接池

#### 配置参数

##### maxLive (100)

##### maxIdle (100)

##### minIdle (10)

##### initialSize (10)

##### maxWait (30s)

当第101个请求过来时候，等待30s; 30s后如果还没有空闲链接，则报错

#### 由客户端维护

### 收缩表空间

#### 空洞

##### 删除数据导致空洞：只是标记记录为可复用，磁盘文件大小不会变

##### 插入数据导致空洞：索引数据页分裂

#### 重建表

##### alter table t engine=innodb, ALGORITHM=copy;

###### 原表不能同时接受更新

###### temp table

##### alter table t engine=innodb, ALGORITHM=inplace; 

###### Online DDL: 同时接受更新

###### temp file

###### row log

##### analyze table t 其实不是重建表，只是对表的索引信息进行重新统计

##### optimize table = alter + analyze

### 误删恢复

#### 误删行

##### binlog_format=row + Flashback工具

##### 预防

###### sql_safe_updates=on

保证delete语句必须有where, where里包含索引字段


#### 误删库/表

##### Flashback不起作用

truncate /drop table 和 drop database, binlog里记录的是statement，所以无法通过Flashback恢复


##### 全量备份 + 恢复实时binlog

##### 延迟复制备库

CHANGE MASTER TO MASTER_DELAY = N (秒)

#### rm删除数据

##### 重新选主

### 复制表数据

#### 物理拷贝

- create table r like t;
- alter table r discard tablespace;
- flush table t for export;
- cp t.cfg r.cfg;
  cp t.idb r.idb;
- unlock tables;
- alter table r import tablespaces;

#### mysqldump

mysqldump -h$host -P$port -u$user 
--add-locks=0 
--no-create-info 
--single-transaction  
--set-gtid-purged=OFF 
db1 t 
--where="a>900" 
--result-file=/client_tmp/t.sql


mysql -h127.0.0.1 -P13000  -uroot
db2 
-e "source /client_tmp/t.sql"

执行INSERT语句，binlog记录的也是INSERT

#### select ... into outfile

select * from db1.t where a>900 into outfile '/server_tmp/t.csv';

load data infile '/server_tmp/t.csv' into table db2.t;

### 主备

#### 主备延迟

##### 时间点：1. 主库完成一个事务写入binlog，2. 传给备库，3. 备库执行

##### show slave status --> seconds_behind_master

##### 原因

###### 备库机器性能差

###### 备库压力大：运营分析脚本 （用一主多从解决，或者输出到hadoop提供统计类查询的能力）

###### 大事务

####### 一次性用delete语句删除太多数据

####### 大表DDL

###### 备库的并行复制能力

##### 解决方案

###### 强制走主库

###### sleep

###### 判断主备无延迟

####### 判断seconds_behind_master

每次查询从库前，判断seconds_behind_master是否为0

####### 对比位点

show slave status
- Master_Log_File <-> Read_Master_Log_Pos
- Relay_Master_Log_File <-> Exec_Master_Log_Pos

####### 对比GTID

- Retrieved_Gtid_Set 备库收到的所有日志GTID
- Executed_Gtid_Set 备库所有已执行完的GTID
二者相同则表示备库接收到的日志都已经同步完成。

###### 配合 semi-sync

semi-sync
- 事务提交时，主库把binlog发给从库；
- 从库收到binlog后，返回ack;
- 主库收到ack后，才给客户端返回事务完成的确认。

问题：
- 如果有多个从库，则还是可能有过期读。
- 在持续延迟情况下，可能出现过度等待。

###### 等主库位点

- 更新完成后，执行 `show master status`获取当前主库执行到的file/pos.

- 查询从库前， `select master_pos_wait(file, pos, timeout)` 查询从命令开始执行，到应用完file/pos表示的binlog位置，执行了多少事务。

- 如果返回 >=0, 则在此从库查询；否则查询主库

###### 等GTID

- 主库更新后，返回GTID;
- 从库查询 `select wait_for_executed_gtid_set(gtid_set, 1)`
- 返回0则表示从库已执行该GTID.

#### 主备切换

##### 主备切换策略

###### 可靠性优先策略

保证数据一致。
- 判断备库seconds_behind_master, 如果小于阈值则继续；
- 主库改成readonly；
- 判断备库seconds_behind_master，如果为0，则把备库改成可读写 readonly=false.

问题：步骤2之后，主备都是readonly，系统处于不可写状态。

###### 可用性优先策略

保证系统任意时刻都可写。

- 直接把连接切到备库，并让备库可读写。
- 再把主库改成readonly

问题：
若binlog_format=mixed，可能出现数据不一致；
若binlog_format=row，主备同步可能报错，例如duplicate key error.

##### 如何判断主库出问题

###### select 1

####### 问题场景：innodb_thread_concurrency小，新请求都处于等待状态

innodb_thread_concurrency一般设为64~128.
线程等待行锁时，并发线程计数会-1.

###### 查表判断

mysql> select * from mysql.health_check; 

####### 问题场景：空间满了后，查询可以，但无法写入

###### 更新判断

update mysql.health_check set t_modified=now()

####### 问题场景：主备库做相同更新，可能导致行冲突

解决：多行数据 (server_id, now())

###### 内部统计

假定认为IO请求时间查过200ms则认为异常：
```
mysql> select event_name,MAX_TIMER_WAIT  FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;
```

####### 打开统计有性能损耗

### 安全

#### privilege

##### create user

create user 'user1'@'%' identified by 'pwd';

###### 会插入mysql.user表

##### grant privilege

grant all privileges on *.* to 'ua'@'%' with grant option;

- *.* 全局权限
- db1.* 库权限
- db1.t1 表权限
- grant select(id), .. 列权限

##### revoke privilege

revoke all privileges on *.* from 'ua'@'%';

##### flush privileges

###### 作用: 清空acl_users数组，从mysql.user表重新读取数据

###### grant/revoke后没必要执行flush

### 分库分表

#### 分区表

CREATE TABLE `t` (
  `ftime` datetime NOT NULL,
  `c` int(11) DEFAULT NULL,
  KEY (`ftime`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1

PARTITION BY RANGE (YEAR(ftime))
(PARTITION p_2017 VALUES LESS THAN (2017) ENGINE = InnoDB,
 PARTITION p_2018 VALUES LESS THAN (2018) ENGINE = InnoDB,
 PARTITION p_2019 VALUES LESS THAN (2019) ENGINE = InnoDB,
PARTITION p_others VALUES LESS THAN MAXVALUE ENGINE = InnoDB);


insert into t values('2017-4-1',1),('2018-4-1',1);


##### 特点

###### 缺点1：MySQL第一次打开分区表时，需要访问所有分区

###### 缺点2：server层认为是同一张表，所有分区共用同一个 MDL 锁

###### 引擎层认为是不同的表

##### 场景

###### 历史数据，按时间分区

###### alter table drop partition清理历史数据

#### mycat
