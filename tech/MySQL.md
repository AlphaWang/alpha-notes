[toc]





# | 基础

Vs. Oracle

- 共享池 Shared Pool

  - 库缓存 Library Cache
    - 缓存执行计划：命中 --> 软解析，不命中 --> 硬解析。
    - 用 `绑定变量` 提升命中率

  - 数据字典缓冲区



## || 逻辑架构

**Server 层**

- **连接器**

  - 管理连接、权限验证

  - 推荐用长连接
    - 问题：长期积累可能OOM。因为执行过程中临时使用的内存是管理在连接对象里，直到连接断开才释放
    - 优化：定期断开长连接。
    - 优化：每次执行比较大的操作后，执行 mysql_reset_connection 重新初始化连接

- **查询缓存**
  - 表更新后，缓存即失效
  - 8.0 不再支持查询缓存

- **分析器**

  - 词法分析

  - 语法分析

- **优化器**

  - 找到最佳执行计划

  - 选择索引
  - 选择表连接顺序

- **执行器**

  - 判断权限

  - 操作引擎



**存储引擎**

- **InnoDB**
  - 支持事务：InnoDB每一条SQL语言都默认封装成事务，自动提交，这样会影响速度，所以最好把多条SQL语言放在begin和commit之间，组成一个事务；
  - 支持外键、行锁
  - 不保存表的具体行数
  - redo log 支持崩溃恢复

- **MyISAM**
  - 不支持事务
  - 不支持行锁
  - 不支持外键
  - 保存了整个表的行数
  - 支持全文索引

- **Memory**

- **NDB**

- **Archive**



## || 存储结构

**行 Row**



**页 Page**

- 存储的基本单位，默认16K

  ```sql
  mysql> show variables like '%innodb_page_size%';
  ```

- 页结构

  - 文件头 File Header

    > 包含两个指针，串联起数据页双向链表：
    > FIL_PAGE_PREV
    > FIL_PAGE_NEXT

  - 页头 Page Header

  - 最大最小记录 Infimum+Supremum

  - 用户记录 User Records

  - 空闲空间 Free Space

  - 页目录 Page Directory

  - 文件尾 File Tailer

  

**区 Extent**

- 包含64个连续的Page



**段 Segment**

- 创建表时 --> 表段

- 创建索引时 --> 索引段：B+树上的非叶子节点

- 数据段：B+树上的叶子节点

- 回滚段



**表空间 Tablespace**



# | 原理

## || Update

**WAL: Write-Ahead-Logging**

- 原理

  - 先写日志，再写磁盘。
  - InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里。

- 执行流程

  - **Step-1 读到内存**：【执行器】先找【引擎】取出行

    > 引擎逻辑：
    >
    > - 如果这行本来就在内存中，直接返回给执行器；
    > - 否则，先从磁盘读入内存，再返回。

  - **Step-2 更新内存**：【执行器】执行更新，【引擎】将新数据更新到内存

  - **Step-3 更新 Redo log**：【引擎】同时将更新操作记录到 redo log （此时处于prepare状态）

  - **Step-4 写入 Bin log**：【执行器】生成 bin log，并写入磁盘

  - **Step-5 提交 Redo log**：【执行器】调用引擎的提交事务接口，【引擎】提交redo log

    > 两阶段提交 2PC
    >
    > redo log 先 prepare， 
    > 再写 binlog，
    > 最后commit redo log

    

**Redo log**

- 目的：InnoDB特有，为了崩溃时可以恢复更改

- 原理：循环写

  - 默认4个文件，每个1G
  - write pos：当前记录的位置
  - checkpoint：当前要擦除的位置

- 写入机制

  - innodb_flush_log_at_trx_commit

    > 0：事务提交时，log写入redo log buffer
    >
    > 1：事务提交时，log写入disk 【推荐】
    >
    > 2：事务提交时，log写入FS page cache

  - 后台线程，每1秒将redo log buffer， write 到 FS page cache，然后fsync持久化到磁盘

- 配置参数
  - `innodb_log_file_size`：默认1G
  - `innodb_log_buffer_size`：默认8M
  - `innodb_flush_log_at_trx_commit`



**binlog**

- Server共有

- 追加写：不会覆盖旧日志

- 格式

  - statement
  - row

- 写入机制

  - 事务执行中：把日志写到 binlog cache

  - 事务提交时：把 binlog cache 写到 binlog 文件

    - write: 写入文件系统的page cache

    - fsync: 持久化到硬盘

    - sysnc_binlog

      > 0: 每次提交事务，只 write 不 fsync
      >
      > 1: 每次提交事务，都 fsync 【推荐】
      >
      > N: 每次提交事务，都write，累积 N 个后 fsync



**脏页**：内存数据页跟磁盘数据页内容不一致

- flush: 写回磁盘
- flush 过程会造成抖动
- 何时触发 flush
  - redo log写满（此时系统不再接受更新，更新数跌成0）
  - 系统内存不足
  - 系统空闲时
  - mysql 关闭时

## || Insert

- **insert ... select**
  - 常用于两个表拷贝
  - 对源表加锁 
  - 如果insert和select是同一个表，则可能造成循环写入

- **insert into t ... on duplicate key update d=100**
  - 插入主键冲突的话，执行update d=100



## || Select

**步骤**

- FROM / ON

- WHERE

- GROUP BY 分组

- 聚合函数计算

- HAVING 筛选分组

- 计算所有表达式

- SELECT 字段

- ORDER BY 排序

- LIMIT 筛选



## || Count(*)

- MyISAM把总行数存在磁盘上，count(*)效率奇高

- 效率：count(字段) < count(id) < count(1) = count(*)

  - `count(字段)`：满足条件的数据行里，字段不为null的总个数；
  - `count(主键id)`：引擎遍历表，取出id返回给server；server判断是否为空，按行累加；

  - `count(1)`：引擎遍历表，但不取值，server层对于返回的每一行 放入数字1；

  - `count(*)`：并不会取出全部字段，*肯定不是null,直接按行累加。

- count(*) / count(1) 时尽量采用二级索引
  - 聚簇索引包含信息太多，而`count(*)`只是统计行

## || Join

**分类**

- **NLJ**：Index Nested-Loop Join
  - 先遍历t1 --> 对于每一行，去t2走树搜索
    - 驱动表t1：全表扫描
    - 被驱动表t2：树搜索，使用索引！
  - 原则：应该让小表来做驱动表
    - 什么是小表？考虑条件过滤后、参与 join 的各个字段总数据量小

- **SNJ**：Simple Nested-Loop Join
  - 当被驱动表 t2 不能使用索引时，扫描行数很大

- **BNL**：Block Nested-Loop Join
  - 读取 t1 存入 join_buffer --> 扫描t2，跟join_buffer数据作对比
    - 如果join_buffer小，则分段存入，驱动表应该选小表
    - 如果join_buffer足够，则驱动表大小无所谓
  - 原则：要尽量避免BNL



**优化**

- **MRR: Multi-Range Read**
  - 按照主键递增顺序查询，则近似顺序读，能提升读性能

- **BKA: Batched Key Access**
  - 用于优化 NLJ 
  - 读取t1一批数据 --> 放到临时内存(join_buffer) --> 一起传给t2

- **BNL性能差**

  - 多次扫描被驱动表，占用磁盘 IO

  - 判断 Join 条件需要执行 M*N 次对比，占用 CPU

  - BNL 大表 Join 后对 Buffer Pool 的影响是持续性的

    - 会占据 Buffer Pool Yong 区
    - 导致业务正常访问的数据页 没有机会进入 Yong 区

  - BNL 转成 BKA

    - 加索引
    - 使用临时表

    

## || 聚合排序

**order by**

- **sort_buffer**
  - 如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助 
    --> sort_buffer_size

- 分类

  - **全字段排序**：无需回表

  - **rowid 排序**

    > SET max_length_for_sort_data = 16;
    > 只在buffer中放入id和待排序字段，需要按照id回表

- 索引对排序的影响

  - 如果有索引
    - 不需要临时表，也不需要排序
    - FileSort：内存排序，可能产生临时文件到磁盘
    - Index排序：通过索引保证顺序性，效率高

  - 进一步优化：覆盖索引

  

- 如何随机排序

  - order by rand()

  - where id >= @X limit 1



**group by**

- 机制

  - 内容少：内存临时表

  - 内容多：磁盘临时表

- 优化

  - **加索引**
    - 如果对列做运算，可用 generated column 机制实现列数据的关联更新。
    - alter table t1 add column z int generated always as(id0), add index(z)

  - **直接排序**
    - select SQL_BIG_RESULT .. from ..
    - 告知mysql结果很大，直接走磁盘临时表



**临时表**

- 例子：**union**
  ```sql
  (select 1000 as f)
  union
  (select id from t1 order by id desc limit 2);
  ```

  - 创建内存临时表，字段f,是主键；

  - 执行第一个子查询，放入1000；

  - 执行第二个子查询，获取的数据插入临时表 （主键去重）

  - 从临时表按行取出数据。

    > NOTE: 如果是 Union ALL 则不会用临时表，因为无需去重，直接依次执行子查询即可。


- 例子：**group by**
  ```sql
  select id as m, 
    count(*) as c,
  from t1 
  group by m;
  ```

  - 创建临时表：字段m, 



## || 缓冲池
innodb_buffer_pool

缓存内容

- 数据页

- 插入缓存

- 自适应哈希索引

- 索引页

- 锁信息

- 数据字典信息



# | 索引

## || 基础

**常见索引模型**

- **哈希表**

  - value 存在数组中，用哈希函数把 key 换算成 index

  - 适用于等值查询，而范围查询需要全部扫描

  - 不支持order by排序、范围查询

- **有序数组**

  - 查询方便，支持范围查询

  - 更新成本高；适用于静态存储

- 跳表

- LSM树

- 搜索树

  - **B Tree**：平衡的多路搜索树

    > M阶B Tree: 
    >
    > - 每个非叶子结点至多有M个儿子，至少有M/2个儿子；
    > - 根节点至少有2个儿子；
    > - 所有叶子节点在同一层。

  - **B+ Tree**：叶子节点才是真正的原始数据
    - 叶子节点存储数据
    - 范围查询更有优势
    - 查询效率更稳定
    - 磁盘读写代价更低

  - 与二叉树的区别
    - 二叉树：优化比较次数
    - B/B+树：优化磁盘读写次数




**索引的场景**

- 适合场景
  - 字段有唯一性限制
  - 频繁作为 where 条件
  - 频繁作为 groupBy, orderBy
  - DISTINCT字段

- 不适合的场景
  - 区分度低，重复记录多：男/女
  - 表记录太少
  - 频繁更新的字段
  - 要对字段进行表达式计算时
  - 联合索引顺序

- 索引的代价

  - 维护代价：新增数据时要修改聚簇索引 + N 个二级索引

  - 空间代价：二级索引虽不存储原始数据，但要存储索引列数据
  - 回表代价





## || 簇索引 vs. 非簇索引

- **簇索引** -主键索引
  - 每个表至多一个，一般为主键索引。
  - 索引上存储的是数据本身
  - 增删改效率不高

- **非簇索引** -二级索引

  - 索引上存储的是主键

  - 需要两次查找才能获取数据本身 --> 回表

  

## || 唯一索引 vs. 普通索引

- **唯一索引** = 普通索引 + UNIQUE

- **主键索引** = 普通索引 + UNIQUE  + NOT NULL

- **普通索引**：**change buffer**

  > 当需要更新一个数据页时，
  >
  > - 如果数据页在内存中就直接更新；
  > - 而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB 会将这些更新操作缓存在 change buffer 中，这样就不需要从磁盘中读入这个数据页了。
  > - 在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。
  >

  - **merge**：将change buffer中的操作应用到原数据页，触发时机：
    - 访问该数据页时
    - 后台线程定期 merge
    -  数据库关闭时
  - 条件：不适用于唯一索引，只限于普通索引
  - 适用场景：`写多读少`
    - 因为 merge 的时候是真正进行数据更新的时刻，而 change buffer 的主要目的就是将记录的变更动作缓存下来，所以在一个数据页做 merge 之前，change buffer 记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。 
  - 与 redo log 的区别
    - change buffer 主要节省的是随机读磁盘的 IO 消耗
    - redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写）
    - 写入 change buffer 后也是要写redo log的

- 性能对比

  - 读取：类似
  - 更新：普通索引好





## || 主键

**自增主键**

- 优点

  - 性能好
    - 插入是追加操作
    - 不挪动其他记录，不触发页分裂；因为主键是聚簇索引 

  - 存储空间小

- 机制

  - show create table命令，AUTO_INCREMENT=2 列出下一个自增值 (Y)

  - 假设插入值为 X
    - 当 X<Y，则自增值不变
    - 当 X>=Y，则自增值更新

- **不保证连续**：插入数据时，是先更新自增值，再执行插入。如果插入异常，自增值不回退。		

  - 由于主键冲突导致

  - 由于回滚导致

  > 自增值为何设计为不能回退？
  > 假设有并行事务A/B，A申请到id=2，B申请到 id=3；
  > A回退，如果把id也回退为2，那么后续继续插入数据 id=3 是会冲突。

- 自增锁

  - innodb_autoinc_lock_mode

    - = 0，语句执行结束才放锁。

    - = 1，普通insert: 申请自增锁后马上释放；insert...select: 语句执行结束才放锁。——因为不知道会插入多少行，无法一次性申请id

    - = 2，申请自增锁后马上释放。

      > 若此时binlog_format = statement，对于 insert...select 语句会有数据一致性问题：
      >
      > - insert into t2 select from t1, 的同时t1表在插入新数据。
      > - t2 id 可能不连续
      > - 但同步到备库，id连续

  - 建议配置
    - innodb_autoinc_lock_mode = 2
    - binlog_format = row
    - 既能提升并发性，又不会出现数据一致性问题



**业务主键**

- 适用场景
  - 只有一个索引，例如KV





## || 覆盖索引

>  select primaryKey（而不是*） from xx where indexColumn=

- 非簇索引节点值为“primaryKey”，而非真实数据记录。
- 避免回表



实现

- 1. select 主键

- 2. 把返回字段、查询条件 作为联合索引。——**宽索引**



## || 前缀索引

联合索引(a, b), 只查a也会索引 

- 好处
  - 减少索引大小，每页存储更多的索引，提高查询效率

- 缺点
  - 前缀索引可能增加扫描行数
  - 无法利用覆盖索引



**概念**

- **字符串索引的最左M个字符**

  - 字符串类型：用前缀索引，并定义好长度，即可既节省空间、又不用额外增加太多查询成本。如何定义长度:
    ```sql
    alter table t add index i_2(email(6))
    ```

  - 如何找到合适的长度：**区分度**
    ```sql
    select 
    count(distinct email) as L,
    count(distinct left(email, 4)) as L4,
    count(distinct left(email, 5)) as L5,
    ...
    from User;
    
    -- 假设可接受的损失比是5%，则找出不小于 L*95% 的值
    ```

    

- **联合索引的最左N个字段**

  - 限制：遇到范围查询，则后续的列就无法使用索引了

    > 索引：(x, y, z)
    > 查询：`x=9 AND y>8 AND z=7` ，则匹配前缀索引 xy。——z 无法使用索引！



**联合索引的顺序**

- 如果调整顺序可以少维护一个索引，则用该顺序

- 如不能，则考虑空间优先

  > 空间有效：查询name+age, name, age
  >
  > - 策略A: (name, age) (age)
  > - 策略B: (age, name) (name)
  >
  > 应选 A，因为 age 占用空间更小



## || 索引下推

(name, age)

可以适用于 where name like `张%` AND age=10

直接过滤掉不满足条件 age!=10 的记录，减少`回表次数`。 




## || 三星索引标准

**定义**

- 一：WHERE 列作为初始。——最小化索引片
- 二：groupBy orderBy 列。——避免file sort
- 三：select 列。——避免回表



**反范式**

- 宽索引减少了回表，但会造成页分裂、页合并。——每个页存储的索引数据变少了。

- 添加记录的索引维护成本高



## || 索引失效

**优化器选择索引**

- **正常优化**

  - **扫描行数**
    - 判断扫描行数的依据：区分度cardinality。——`show index`可查区分度。
    - 区分度通过采样统计得出。
    - analyze table t 重新统计索引信息。

  - **是否使用临时表**

  - **是否排序**
  - **是否回表**。——主键索引无需回表

- **选错索引的情况**

  - 例如为了避免回表，使用了主键索引；而实际上用 where 条件中的索引会更快。



**索引选择非最优时如何补救**

- force index 强行选择一个索引
  ```sql
  select * from t force index(a) where...
  ```

- 修改语句，引导使用希望的索引
  ```sql
  order by b limit 1 --> index b
  order by b, a limit 1 --> index a
  ```

- 新建合适的索引



**optimizer_trace** 调试优化器

- 打开后，查询 information_schema.OPTIMIZER_TRACE
  ```sql
  SET optimizer_trace="enabled=on";
  
  SELECT * FROM person WHERE NAME >'name84059' AND create_time>'2020-01-24 05:00:00';
  
  SELECT * FROM information_schema.OPTIMIZER_TRACE;
  
  SET optimizer_trace="enabled=off";
  ```

  

**查询语句导致索引失效**

- 函数操作
  ```sql
  where month(t_date) = 7
  ```

- 表达式操作
  ```sql
  where uid + 1 = 9
  ```

- OR 某个条件缺索引

- 条件字段 判断 NULL, NOT NULL

- 联合索引不包含第一列

- 联合索引中间有列范围查询，则后续列无法使用索引

- 转换

  - 隐式字符编码转换，例如 utf8 --> utf8mb4

  - 隐式类型转换，例如 `where tradeid = 123` , tradeid is varchar

# | 锁

## || 锁的粒度

粒度越小，越容易出现死锁。 

**1. 全局锁**

- 命令：Flush tables with read lock (FTWRL)

- 场景：全库逻辑备份 

  > 当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：
  >
  > - 数据更新语句（增删改）
  > - 数据定义语句（建表、修改表结构等）
  > - 更新类事务的提交语句

  - innodb 一般用 `mysqldump -single-transaction`。
    ——此命令会在导数据之前启动一个事务，确保拿到一致性视图。由于MVCC的支持，导出数据过程中可以正常进行更新操作。

  - innodb MVCC 支持

  

- 区别与 `set global readonly=true`
  - 一些系统会用readonly来判断是否从库。
  - 如果客户端异常断开，readonly 状态不会重置，风险高。

- 优化：innoDB可在可重复读隔离级别下开启一个事务



**2. 表锁**

- 表锁（lock tables xx read/write）

- 元数据锁（MDL, metadata lock)

  - 自动加上，无需显式使用
    - 当对一个表做增删改查操作的时候，加 MDL 读锁；
    - 当要对表做做结构变更操作的时候，加 MDL 写锁；

  - **增加字段需要 MDL 写锁，会block业务**

    - ALTER +timeout : ALTER .. WAIT N
      ```sql
      ALTER TABLE tbl_name NOWAIT add column ...
      ALTER TABLE tbl_name WAIT N add column ... 
      ```

    - 执行之前杀掉长事务



**3. 行锁**

- MyISAM 不支持行锁

- **两阶段锁协议**
  - 行锁是在需要的时候才加上的
  - 但并不是不需要了就立刻释放，而是等到事务结束

- 最佳实践
  - 要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放

- **死锁**：只会发生在共享锁情况下

  - **死锁检测**：innodb_deadlock_detect=on
    - 发现死锁后，主动回滚死锁链条中的某一个事务
    - 问题：耗费 CPU。当更新热点行时，每个新来的线程都要判断会不会由于自己的加入导致死锁。
    - 解决：服务端并发控制。对于相同行的更新，在进入引擎之前排队

  - **死锁条件**
    - 互斥
    - 占有且等待
    - 不可强占用
    - 循环等待

  - **死锁解决**
    - 等待超时：innodb_lock_wait_timeout
    - 一次性锁定所有资源
    - 锁升级：行锁--> 表锁



## || 共享锁 vs. 排它锁

**共享锁 / S锁 / 读锁**

- 其他事务可以读取，但不能修改。——保证数据在读取时不被修改。
- **共享锁表**：`LOCK TABLE xx READ`

- **共享锁行**：`SELECT ** LOCK IN SHARE MODE`



**排它锁 / X锁 / 写锁**

- 其他事务无法进行查询或修改

- 当进行数据更新时（insert, delete, update），自动使用排它锁

- **意向锁 Intent Lock**：给更大一级的空间示意里面已经有锁

  > 排他锁行时，同时在表上加意向锁。
  > 当其他事务尝试获取排他表锁时，无需再遍历看是否有行锁。

- **排他锁表**：`LOCK TABLE xx WRITE`

- **排他锁行**：`SELECT ** FOR UPDATE`



## || 乐观锁 vs. 悲观锁

**乐观锁**

- where stock=xx
- 将某一字段作为版本号

**悲观锁**

- SELECT ** FOR UPDATE
- 锁住行



# | 事务

## || 事务特性ACID

- **1. 原子性 Atomicity**

  - 不可分割

  - 要么都成功，要么都失败

  - 是基础

- **2. 一致性 Consistency**

  - 事务操作后，会由原来的一致状态，变成另一种一致状态

  - 是约束条件

- **3. 隔离性 Isolation**
  - 事务彼此独立，不受其他事务影响
  - 是手段

- **4. 持久性 Duration**

  - 事务提交之后对数据的修改是持久性的

  - 即使后续宕机，也不会改变事务的结果
  - 是目的

  

## || 事务隔离级别

**读未提交 -RU，Read Uncommitted**

- 加共享锁
- 完全不隔离
  - 会出现脏读
  - 基本不用



**读提交 -RC，Read Committed**

- 查询承认在“语句”启动前就已经提交完成的数据

- 解决

  - 脏读

    > 脏读：读到另一个事务尚未提交的修改。



**可重复读 -RR，Repeatable Read**

- 查询只承认在“事务”启动前就已经提交完成的数据

- 解决

  - 脏读

  - 不可重复读

    > 不可重复读：同一事务中两次相同查询返回值不一样。



**序列化 -Serialized**

- 对相关记录加读写锁

- 解决

  - 脏读

  - 不可重复读

  - 幻读

    > 幻读：同一事务两次相同查询返回的记录数量不一致。
    >
    > 例如事务A 插入记录R，报主键冲突，但是查询 R 却查不到。——因为另一个事务B新插入了R。######## 因为另一个事务B新插入了R。



**事务隔离编码**

- 查询隔离级别

  ```sql
  SHOW VARIABLES LIKE 'transaction_isolation';
  ```

- 设置隔离级别

  ```sql
  SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
  ```

- 关闭/开启自动提交

  ```sql
  set autocommit = 0 | 1
  ```

- SET @@completion_type = 

  > 0：commit 提交事务后，下一个事务还要用 begin 开始
  >
  > 1：commit 提交事务后，自动开启一个相同隔离级别的事务
  >
  > 2：commit 提交事务后，自动与服务器断开连接

- BEGIN / COMMIT / ROLLBACK





## || 事务隔离原理

**MVCC：多版本并发控制**

> Multi Version Concurrency Control
>
> https://juejin.im/post/5c68a4056fb9a049e063e0ab 

- 思想
  - 不同时刻启动的事务会有不同的 read-view。
  - 保存数据的历史版本，通过比较版本号决定数据是否显示。
  - 不用加锁即可保证事务的隔离效果

- 好处
  - 解决读写之间阻塞的问题
  - 解决一致性读的问题：快照读
  - 降低死锁概率：乐观锁

- 原理

  - **事务ID：自增长**。通过ID大小即可判断事务的时间顺序。
  - **行记录的隐藏列**
    - `db_row_id`：行ID
    - `db_trx_id`：最后操作这个数据的事务ID。—— = “创建时间”
    - `db_roll_ptr`：指向这个记录的 undo log 信息，即是回滚指针。通过回滚指针可以遍历到 历史快照。——= “过期时间”

  - **undo log**：回滚日志
  - **通过 Undo Log + Read View 进行数据读取**
    - **MV**:  Undo log 保存多版本的历史快照
    - **CC**: Read View 帮助判断当前版本的数据是否可见



**快照读 vs. 当前读**

**查询数据：一致性读（consistent read view）**

- **秒级创建快照**

  - 基于 row trx_id实现`快照`

    > 事务 transaction id 严格递增；
    >
    > 事务更新数据时，会把 transaction id 赋值给数据版本 row trx_id；
    >
    > db_trx_id: 最后操作这个数据的事务ID；

  - 数据表中的一行记录，其实可能有多个版本（row），每个版本都有自己的 row trx_id

- **用于实现可重复读**
  - 如果数据版本是我启动以后才生成的，我就不认，我必须要找到它的上一个版本。

- **数据版本的可见性**

  - 根据 row trx_id 、一致性视图（read-view）确定数据版本的可见性。

  - 每个事务有一个对应的数组，记录事务启动瞬间所有活跃的事务 ID。

    > 数组里事务ID最小值：低水位；
    >
    > 数组里事务ID最大值+1：高水位

  - 判断可见性

    > row trx_id < 低水位：表示是已提交的事务生成的，对当前事务可见
    >
    > row trx_id > 高水位：表示是有将来启动的事务生成的，不可见
    >
    > row trx_id 在两者之间：遍历数组。如果 row trx_id 在数组中，表示是由未提交的事务生成的，不可见；如果row trx_id不在数组中，表示是已提交的事务生成的，可见。

  - 判断可见性(简化)

    > 版本未提交：不可见
    >
    > 版本已提交，但在视图创建后提交：不可见
    >
    > 版本已提交，且在视图创建前提交：可见

- **Read View重要属性**
  - trx_ids：当前活跃的事务 ID 集合。活跃事务：未提交。
  - low_limit_id：最大活跃事务 ID。
  - up_limit_id：最小活跃事务 ID。
  - creator_trx_id：创建当前读视图的事务 ID。



**更新数据：当前读（current read ）**

- **更新数据都是先读后写的**
  - 而这个读，只能读`当前的值`，即当前读current read；
  - 并非一致性读！！不会去判断row trx_id
- **当前读必须要读最新版本**
  - 所以必须加锁
  - 如果此时有其他事务更新此行 但未提交，则行锁被占用，当前事务会等待锁

- 除了 update 语句外，select 语句如果加锁，也是`当前读`
  - 共享锁：select xx lock in share mode;
  - 排它锁：select xx for update;





# | 高可用

## || 主从

**原理**

- 组件

  - 主库：binlog dump thread。

    > 等待从库连接，主库将binlog发送给从库。

  - 从库：IO thread -> Relay log

    > 向主库发送请求更新binlog；拷贝到本地 Relay Log

  - 从库：SQL thread

    > 读取Relay Log，执行日志中的事件

- 复制方式

  - 方式一：**异步复制**
    - 客户端提交commit后，不需要等从库返回结果；一致性最弱

  - 方式二：**半同步复制**
    - 客户端提交commit后，至少等 {N} 个从库接收到binlog，并写入中继日志；
    - 提升了一致性，降低了主库写的效率。
    - N = rpl_semi_sync_master_wait_no_slave

  - 方式三：**组复制**

    - 基于Paxos，强一致性

    

## || 读写分离

Tips：

- 若有多个从库，可用 HAProxy + Keepalived 实现从库的负载均衡和高可用



## || 主备延迟

时间点：1. 主库完成一个事务写入binlog，2. 传给备库，3. 备库执行

查看延时：`show slave status` --> seconds_behind_master

**主备延迟的原因**

- 备库机器性能差

- 备库压力大：运营分析脚本 （用一主多从解决，或者输出到hadoop提供统计类查询的能力）

- 大事务
  - 一次性用 delete 语句删除太多数据
  - 大表 DDL

- 备库的并行复制能力



**解决方案**

- **强制走主库**
  - 写+读放到同一个事务，则读也会走主库

- **sleep**

- **判断主备无延迟**

  - 判断 seconds_behind_master

    > 每次查询从库前，判断seconds_behind_master 是否为 0。

  - 对比位点：

    > show slave status
    >
    > - Master_Log_File <-> Read_Master_Log_Pos
    > - Relay_Master_Log_File <-> Exec_Master_Log_Pos

  - 对比GTID

    > - Retrieved_Gtid_Set 备库收到的所有日志GTID
    > - Executed_Gtid_Set 备库所有已执行完的GTID
    >
    > 二者相同则表示备库接收到的日志都已经同步完成。


- **配合 semi-sync**

  > semi-sync
  >
  > - 事务提交时，主库把binlog发给从库；
  > - 从库收到binlog后，返回ack;
  > - 主库收到ack后，才给客户端返回事务完成的确认。
  >
  > 问题：
  >
  > - 如果有多个从库，则还是可能有过期读。
  > - 在持续延迟情况下，可能出现过度等待。

- **等主库位点**

  - 更新完成后，执行 `show master status`获取当前主库执行到的file/pos.


  - 查询从库前， `select master_pos_wait(file, pos, timeout)` 查询从命令开始执行，到应用完file/pos表示的binlog位置，执行了多少事务。


  - 如果返回 >=0, 则在此从库查询；否则查询主库

- **等GTID**

  - 主库更新后，返回GTID;

  - 从库查询 `select wait_for_executed_gtid_set(gtid_set, 1)`

  - 返回0则表示从库已执行该GTID.


- **调整业务**

  - 例如付款完成后，不会自动跳到订单页

  

## || 主备切换

**主备切换策略**

- **可靠性优先策略**：保证数据一致。

  - Step-1：判断备库 seconds_behind_master，如果小于阈值则继续；

  - Step-2：主库改成readonly；

  - Step-3：判断备库seconds_behind_master，如果为0，则把备库改成可读写 readonly=false.

    > 问题：步骤2之后，主备都是readonly，系统处于不可写状态。


- **可用性优先策略**：保证系统任意时刻都可写。

  - Step-1：直接把连接切到备库，并让备库可读写。

  - Step-2：再把主库改成readonly

    > 问题：
    > 若binlog_format=mixed，可能出现数据不一致；
    > 若binlog_format=row，主备同步可能报错，例如duplicate key error.




**如何判断主库出问题**

- **select 1**

  - 问题场景：innodb_thread_concurrency 小，新请求都处于等待状态

    > innodb_thread_concurrency一般设为64~128.
    > 线程等待行锁时，并发线程计数会-1.

- **查表判断**

  ```sql
  mysql> select * from mysql.health_check; 
  ```

  - 问题场景：空间满了后，查询可以，但无法写入

- **更新判断**

  ```sql
  update mysql.health_check set t_modified=now()
  ```

  - 问题场景：主备库做相同更新，可能导致行冲突
  - 解决：多行数据 (server_id, now())

- **内部统计**

  - 假定认为IO请求时间查过200ms则认为异常：
    ```sql
    mysql> select event_name,MAX_TIMER_WAIT  FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;
    ```

  - 打开统计有性能损耗



## || 分库分表

**分区表**

```sql
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
```



- **特点**

  - 缺点1：MySQL 第一次打开分区表时，需要访问所有分区

  - 缺点2：server 层认为是同一张表，所有分区共用同一个 MDL 锁

  - 引擎层认为是不同的表

- 场景

  - 历史数据，按时间分区

  - alter table drop partition清理历史数据

  

**mycat** 

- TBD



## || 设置

- 查看 binlog设置：show variables like '%log_bin%'
  ```sql
  mysql> show variables like '%log_bin%';
  
  log_bin: ON 
  log_bin_basename: /usr/local/var/mysql/binlog  
  ```

- 数据恢复：mysqlbinlog --start-datetime --stop-datetime
  ```sql
  $mysqlbinlog --start-datetime "2020-02-20 00:00:00" --stop-datetime "2020-02-20 15:09:00" /usr/local/var/mysql/binlog.000001 | mysql -uroot
  ```

  


# | 运维

## || 连接池

配置参数

- maxLive (100)

- maxIdle (100)

- minIdle (10)

- initialSize (10)

- maxWait (30s)

  - 当第101个请求过来时候，等待30s; 30s后如果还没有空闲链接，则报错。

  

由客户端维护



## || 收缩表空间

**空洞**

- 删除数据导致空洞：只是标记记录为可复用，磁盘文件大小不会变
- 插入数据导致空洞：索引数据页分裂



**重建表**

- `alter table t engine=innodb, ALGORITHM=copy;`
  - 原表不能同时接受更新
  - temp table

- `alter table t engine=innodb, ALGORITHM=inplace; `
  - Online DDL: 同时接受更新
  - temp file
  - row log

- `analyze table t` 
  - 其实不是重建表，只是对表的索引信息进行重新统计

- `optimize table` = alter + analyze



## || 误删恢复

**误删行**

- `binlog_format=row` + Flashback工具
- 预防
  - sql_safe_updates=on。
  - 保证delete语句必须有where, where里包含索引字段



**误删库/表**

- 此时 Flashback不起作用
  - truncate /drop table 和 drop database, binlog里记录的是statement，所以无法通过Flashback恢复

- 全量备份 + 恢复实时binlog
- 延迟复制备库
  - CHANGE MASTER TO MASTER_DELAY = N (秒)



**rm 删除数据**

- 重新选主



## || 复制表数据

**物理拷贝**

- create table r like t;
- alter table r discard tablespace;
- flush table t for export;
- cp t.cfg r.cfg;
  cp t.idb r.idb;
- unlock tables;
- alter table r import tablespaces;



**mysqldump**

```sql
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

-- 执行INSERT语句，binlog记录的也是INSERT
```



**select ... into outfile**

```sql
select * from db1.t where a>900 into outfile '/server_tmp/t.csv';

load data infile '/server_tmp/t.csv' into table db2.t;
```



## || 安全

**privilege**

- create user：会插入mysql.user表
  ```sql
  create user 'user1'@'%' identified by 'pwd';
  ```

  

- grant privilege
  ```sql
  grant all privileges on *.* to 'ua'@'%' with grant option;
  ```

  - *.* 全局权限
  - db1.* 库权限
  - db1.t1 表权限
  - grant select(id), .. 列权限

  

- revoke privilege
  ```sql
  revoke all privileges on *.* from 'ua'@'%';
  ```

  

- flush privileges

  - 作用: 清空acl_users数组，从mysql.user表重新读取数据

  - grant/revoke后没必要执行flush

  

# | 开发

## || 设计

**||| 范式** 

> **超键**：能唯一标识元组的属性集叫做超键。`id + 姓名`
>
> **候选键**：如果超键不包括多余的属性，那么这个超键就是候选键。 `id`
>
> **主键**：用户可以从候选键中选择一个作为主键。
>
> **外键**：如果数据表 R1 中的某属性集不是 R1 的主键，而是另一个数据表 R2 的主键，那么这个属性集就是数据表 R1 的外键。
>
> **主属性**：包含在任一候选键中的属性称为主属性。
>
> **非主属性**：与主属性相对，指的是不包含在任何一个候选键中的属性。



- 1NF：数据表中的任何属性都是**原子性**的，不可再分

- 2NF：数据表里的非主属性都要和该表的候选键有**完全依赖关系**
  - SRP: 一张表就是一个独立对象，不要包含多个职责
  - 否则
    - 数据冗余
    - 插入异常
    - 删除异常

- 3NF：任何非主属性都**不传递依赖**于候选键
  - e.g. 球员编号 --> 球队名称 --> 球队教练
  - vs. 2NF：更进一步，必须直接完全依赖

- BCNF 巴斯范式：主属性对候选键**不存在部分依赖**或传递依赖

- 反范式：冗余字段

  - 优点：提升查询效率
  - 缺点：增删改不便
  - 场景：数据仓库

  

**||| 数据迁移**

要求

- 在线迁移，不断线

- 新旧数据一致

- 可回滚



**双写方案**

- 步骤
  - Step-1. 旧库 --> Binlog --> 新库（从）
  - Step-2. 业务代码：同时写新旧库
    - 可异步
    - 写入新库失败时，要单独记录日志，以便补写
  - Step-3. 校验
    - 重要！
  - Step-4. 切到新库（灰度）



- 回滚	
  - 新旧数据一致，随时回滚



**级联同步方案**

- 步骤
  - Step-1. 旧库 --> Binlog --> 新库（从）
  - Step-2. 新库 --> Binlog --> 备库（从）
  - Step-3. 等三个库的写入一致后，切换读流量到新库
  - Step-4. 暂停写入，将写流量切换到新库

- 回滚
  - 切换到备库



**||| DateTime**

https://dev.mysql.com/doc/refman/8.0/en/datetime.html

日期数据类型

- Date
  - 支持范围： `1000-01-01` to `9999-12-31`

- Datetime
  - 支持范围：`1000-01-01 00:00:00` to `9999-12-31 23:59:59`

- Timestamp

  - 支持范围：`1970-01-01 00:00:01 UTC` to `2038-01-19 03:14:07 UTC`

  - 存储格式
    - 写：当前时区 --》转换为UTC存储
    - 读：UTC格式 --》当前时区

## || 规范

**建表**

- decimal
  - 小数类型为 decimal，禁用 float / double，会有精度损失

- char
  - 定长

- varchar
  - 可变长
  - 长度超过5000是用text，独立建表，避免影响其他字段的索引效率



**索引**

- **唯一索引**
  - 业务上具有唯一性的字段，必须建成唯一索引；即便是组合

- **varchar上的索引必须指定索引长度**

  - 根据区分度来选择

  - select (distinct left(列名, 索引长度)) / count (*)

- **order by字段，设为组合索引的一部分**
  - where a=x b=x order by c
  - 索引：a_b_c

- **组合索引：区分度高的放左边**



## || SQL

**||| select**

- select 执行顺序

  > WHERE 
  > GROUP BY 
  > HAVING 
  > SELECT 的字段 
  > DISTINCT 
  > ORDER BY 
  > LIMIT

- 去除重复行：select distinct

- limit 1 提高查询速度



**||| 函数**

- 算术函数

  - ABS()

  - MOD()

  - ROUND()

- 字符串函数

  - CONCAT()

  - SUBSTRING()

  - REPLACE()

- 日期函数

  - CURRENT_DATE()

  - DATE(), YEAR(), MONTH(), ...

    > SQL： SELECT * FROM heros WHERE DATE(birthdate)>'2016-10-01'
    >
    > 如果不用DATE函数，则不安全

  - EXTRACT()

    > SELECT EXTRACT(YEAR FROM '2019')

- 转换函数
  - CAST(xx AS INT)
  - COALESCE() 返回第一个非空数



**||| 子查询**

- 关联子查询 vs. 非关联子查询

- 关键字

  - EXIST, IN
    ```sql
    SELECT * FROM A WHERE cc IN (SELECT cc FROM B)
    SELECT * FROM A WHERE EXIST (SELECT cc FROM B WHERE B.cc=A.cc)
    ```

  - ANY, ALL
    ```sql
    SELECT player_id, player_name, height 
    FROM player 
    WHERE height > ALL (SELECT height FROM player WHERE team_id = 1002)
    ```

    

- 子查询作为计算字段
  ```sql
  SELECT team_name, 
   (SELECT count(*) FROM player WHERE player.team_id = team.team_id) AS player_num 
  FROM team
  ```

  

**||| 存储过程**

- 编写

```sql
DELIMITER //
CREATE PROCEDURE `add_num`(IN n INT)
BEGIN
       DECLARE i INT;
       DECLARE sum INT;
       
       SET i = 1;
       SET sum = 0;
       WHILE i <= n DO
              SET sum = sum + i;
              SET i = i +1;
       END WHILE;
       SELECT sum;
END //
DELIMITER ;
```

- 控制流语句
  - BEGIN...END
  - DECLARE / SET / SELECT INTO
  - IF...THEN...ENDIF
  - CASE...WHEN...THEN...
  - LOOP / LEAVE / ITERATE
  - REPEAT...UNTIL...END
  - WHILE...DO...END

- 优点

  - 一次编译多次使用，提高执行效率

  - 减少网络传输量

- 缺点
  - 可以执行差
  - 调试困难



**||| 游标**

- 编写
  - 定义游标

  - 打开游标
  - 从游标中获取数据
  - 关闭游标
  - 释放游标

```sql
CREATE PROCEDURE `calc_hp_max`()
BEGIN
  -- 创建接收游标的变量
  DECLARE hp INT;  

  -- 创建总数变量 
  DECLARE hp_sum INT DEFAULT 0;
  
  -- 创建结束标志变量  
  DECLARE done INT DEFAULT false;
  
  -- 1. 定义游标     
  DECLARE cur_hero CURSOR FOR SELECT hp_max FROM heros;
  -- 指定游标循环结束时的返回值  
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = true;  
  
  -- 2. 打开游标     
  OPEN cur_hero;
  read_loop:LOOP 
  
  -- 3. 从游标获取数据
  FETCH cur_hero INTO hp;
  
  -- 判断游标的循环是否结束  
  IF done THEN  
     LEAVE read_loop;
  END IF; 
              
  SET hp_sum = hp_sum + hp;
  END LOOP;
  -- 4. 关闭游标
  CLOSE cur_hero;
  SELECT hp_sum;
  -- 5. 释放游标
  DEALLOCATE PREPARE cur_hero;
END

```



- 好处
  - 灵活，对数据进行逐行扫描处理

- 不足

  - 要对数据行加锁，并发量大时会影响效率

  - 消耗内存



# | 调优

## || SQL优化

**表设计优化**

- 尽量遵循第三范式

- 如果经常要进行多表联查，考虑反范式优化，增加冗余字段

- 数据类型：优先数值类型，字符串长度尽可能短



**查询优化**

- 小表驱动大表

- select * 慎用；索引覆盖

- 子查询优化为 join
  - 因为子查询的结果集无法使用索引

- 优化分页查询

  ```sql
  -- as-is 
  select * from `demo`.`order` order by order_no limit 10000, 20;
  -- to-be
  select * from `demo`.`order` where id > (select id from `demo`.`order` order by order_no limit 10000, 1)  limit 20;
  ```

  

**删除优化**

- 问题：删除记录过多
  ```sql
  delete from xx where 
  ```

- 优化1：分批删除
  ```sql
  delete from xx where .. order by id limit 1000
  ```

- 优化2：按主键删除
  ```sql
  select max(id from xx where)
  delete from xx where id > ? order by id limit 1000
  ```

  

## || 运维优化

**profile 命令**

- 打开profiling
  ```sql
  select @@profiling;
  set profiling=1;
  ```

- 查看执行时间
  ```sql
  show profiles; -- 查看query id
  show profile all for query 2;
  ```

  

**Optimize Table**

- 释放空间、重建索引

- 执行时会锁表

- 更快：》复制到新表
  ```sql
  -- 新建一个临时订单表
  create table orders_temp like orders;
  
  -- 把当前订单复制到临时订单表中
  insert into orders_temp
    select * from orders
    where timestamp >= SUBDATE(CURDATE(),INTERVAL 3 month);
  
  -- 修改替换表名
  rename table orders to orders_to_be_droppd, orders_temp to orders;
  
  -- 删除旧表
  drop table orders_to_be_dropp
  ```

  

## || 索引

- 不适合的场景
  - 区分度低，重复记录多：男/女
  - 表记录太少
  - 频繁更新的字段
  - 要对字段进行表达式计算时
  - 联合索引顺序

- 联合索引场景
  - 同时有groupBy + orderBy

## || NOSQL

种类

- KV存储
  - LevelDB
  - Redis

- 列式数据库

  - Hbase

  - Cassandra

- 文档数据库

  - MongoDB

  - CouchDB



作用

- 提升性能：随机 IO --> 顺序 IO

- 场景补充：ES 倒排索引

- 提升扩展性：replica；shard 扩容



## || SQL语句优化思路

**||| 开启慢查询日志**

```sql
show variables like 'slow_query%';
show variables like 'long_query_time';

set global slow_query_log='ON'; // 开启慢 SQL 日志
set global slow_query_log_file='/var/lib/mysql/test-slow.log';
-- 记录日志地址
set global long_query_time=1;
-- 最大执行时间
```



**||| EXPLAIN 执行计划**

执行计划输出：

- **key**
  - 实际用到的索引

- **possible_keys**
  - 可能用到的索引

- **key_len**
  - 实际用到的索引的长度

- **type**：查询方式

  > 效率排行：all < index < range < index_merge < ref < eq_ref < const/system

  - **system**： 只有一行数据匹配 （MyISAM或Memory表）
  - **const**：只有一行数据匹配
    - 主键
    - 唯一索引
  - **eq_ref**：使用唯一索引扫描。例如多表连接中使用 主键/唯一索引 作为关联条件
  - **ref**：使用“非唯一索引”扫描
  - **index_merge**：合并索引，使用多个单列索引
  - **range**：“索引列”范围扫描
  - **index**：索引全表扫描。很慢的。遍历整个索引树
  - **ALL**：全表扫描 

- **ref**：关联ID

- **partitions**：分区表信息

- **rows**： 所扫描的行数

- **table**：查询的表

- **filtered**：查找到所需记录占总扫描记录数的比例

- **select_type**：查询类型
  - SIMPLE
  - PRIMARY
  - UNION
  - SUBQUERY

- **extra**：额外信息
  - Using Index：表示直接查二级索引，避免了回表



**调优**

- rows 越少越好



**||| SHOW PROFILE 分析sql执行性能**

```sql
show variables like 'profiling';
set profiling = 'ON';
```

- SHOW PROFILES
  - 列出最近查询的时间

- SHOW PROFILE for query xx
  - 展开某个查询具体时间