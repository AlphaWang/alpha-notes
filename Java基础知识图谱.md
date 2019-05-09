# Java基础知识图谱

## JVM

### 内存区域

#### 运行时数据区

##### 程序计数器

###### 线程私有

###### 当前线程所执行的字节码的行号指示器

###### OOM: 无

##### JVM 栈

###### 线程私有

###### 存储局部变量表、操作数栈、方法出口

###### StackOverflowError

###### OOM: 动态扩展时无法申请到足够内存

如果`-Xss`设置过大，当创建大量线程时可能OOM.
（每个线程都会创建一个栈）

###### 栈桢 Stack Frame

####### 局部变量表

######## 存放方法参数、方法内定义的局部变量

######## 容量以slot为最小单位

######## 为了尽可能节省栈桢空间，局部变量表中的slot可以重用

######## 局部变量无“准备阶段”，无初始值

####### 操作数栈

######## 存放算术运算操作数、调用其他方法的参数

####### 动态连接

####### 返回地址

######## 正常完成出口 -Normal Method Invocation Completion

######## 异常完成出口 -Abrupt Method Invocation Completion

######## 方法退出时的操作

######### 回复上层方法的局部变量表、操作数栈

######### 把返回值压入调用者栈桢的操作数栈

######### 调整PC寄存器的值，指向方法调用指令后面的一条指令

##### 本地方法栈

##### 方法区

###### 线程共享

###### 存储类信息、常量、静态变量、运行时常量(String.intern)

###### GC: 回收常量池、卸载类型

###### OOM: String.intern，CGLib

- 例如大量String.intern()
- 用CGLib生成大量动态类
- OSGi应用
- 大量JSP的应用 （JSP第一次运行需要便以为Javale）
`-XX:PermSize` `-XX:MaxPermSize`

##### 堆

###### 线程共享

###### 存储对象实例、数组

###### OOM

`-Xms` `-Xmx`

##### 直接内存(堆外内存)

###### NIO DirectByteBuffer

###### OOM

`-XX:MaxDirectMemorySize`

#### 其他

##### Direct Memory

###### NIO可能会操作堆外内存

###### -XX:MaxDirectMemorySize

###### 回收：FullGC时顺便清理，不能主动触发

###### 异常OutOfMemoryError: Direct buffer memory

##### 线程堆栈

###### 纵向异常：无法分配新的栈桢：StackOverflowError

###### 横向异常：无法建立新的线程：OutOfMemoryError: unable to create new native thread

##### Socket缓存区

##### JNI代码

##### 虚拟机和GC

### 对象

#### 创建

##### 在常量池找类的符号引用

##### 类加载

加载，是指查找字节流，并且据此创建类的过程

###### 启动类加载器

###### 扩展类加载器

Java9改名为平台类jia'zai'qi

###### 应用类加载器

##### 链接

###### 验证

确保被加载类能够满足 Java 虚拟机的约束条件

###### 准备

为被加载类的静态字段分配内存

###### 解析

将符号引用解析成为实际引用

##### 初始化

为标记为常量值的字段赋值，以及执行 < clinit > 方法
类的初始化仅会被执行一次，这个特性被用来实现单例的延迟初始化

###### 常量值赋值

###### 执行clinit方法（静态代码块）

###### 初始化仅会被执行一次：单例

##### 分配内存

###### 指针碰撞：适用于Compact GC，例如Serial, ParNew

当内存规整时适用


###### 空闲列表：适用于Mark-Sweep GC，例如CMS

当内存不连续时适用


###### 并发问题

####### CAS

####### TLAB: Thread Local Allocation Buffer

本地线程分配缓冲：每个线程在堆中预先分配一小块内存。

#### 编译：将字节码翻译成机器码

##### 编译器

###### 解释执行

###### JIT (Just in Time Compilation)

####### 即时编译，在运行时将热点代码编译成机器码

###### AOT (Ahead of Time Compilation) 

####### 避免JIT预热开销

##### 参数

###### -Xint: 仅解释执行

###### -Xcomp: 关闭解释器，最大优化级别；会导致启动变慢

#### 结构

##### Header

###### Mark Word

- HashCode
- GC分代年龄
- 锁状态标志
- 线程持有的锁
- 偏向线程ID
- 偏向时间戳

###### 类型指针

JVM通过这个指针来确定对象是哪个类的实例


##### Instance Data

##### Padding

##### 类文件结构

###### 魔数、版本号

###### 常量池

####### 字面量

####### 符号引用

- 类和接口的 全限定名
- 字段名和描述符
- 方法名和描述符

###### 访问标志

- 类 or 接口
- public?
- abstract?
- 
final?

###### 类的继承关系：this_class, super_class, interfaces

###### 字段表集合

- access_flag
- name_index: 简单名称，指向常量池
- descriptor_index: 描述符。字段数据类型，方法参数列表、返回值
- attributes: 属性表


###### 方法表集合

- 类似字段表集合；
- 方法体存放在`Code`shu'xing'l

#### 字节码指令

##### Opcode 操作码  + Operands 操作数

##### load 加载指令：将一个局部变量加载到操作栈

##### store 存储指令：将一个数值从操作数栈 存储到局部变量表

##### 运算指令：iadd, isub, ladd, lsub

##### 类型转换指令：i2b, d2f

##### 对象创建与访问指令：new, newarray, getfield

##### 操作数栈管理指令：pop, swap

##### 控制转移指令

##### 方法调用和返回指令

###### invokestatic

####### 调用静态方法

###### invokespecial

####### 调用 实例构造器、私有方法、父类方法

###### invokevirtual

####### 调用所有的虚方法

###### invokeinterface

####### 调用接口方法

###### invokedynamic

####### 在运行时动态解析出调用点限定符所引用的方法

###### ireturn

##### 异常处理指令：athrow

##### 同步指令：monitorenter, monitorexit

#### 方法调用

##### 变量类型

###### 静态类型 Father f =

###### 实际类型 f = new Son()

##### 静态分派

###### 典型应用：方法重载

###### 依赖静态类型来定位方法

##### 动态分派

###### 典型应用：方法重写

###### 依赖实际类型来定位方法

#### 类加载

##### 生命周期

###### 加载 Loading

- 根据全限定名来获取二进制字节流 `Class Loader`；
- 将字节流所代表的静态存储结构转化为方法区的运行时数据结构；
- 生成java.lang.Class对象

注意：数组类本身不通过类加载器创建，而是有JVM直接创建。

###### 连接 Linking

####### 验证 Verification

- 文件格式验证：
魔数开头、版本号、可能抛出`VerifyError`

- 元数据验证：
是否有父类、是否实现了接口中要求的方法

- 字节码验证：
通过数据流和控制流分析，确定程序语义是否合法

- 符号引用验证：
在讲符号引用转化为直接引用时进行（解析阶段），可能抛出`IllegalAccessError`, `NoSuchFieldError`, `NoSuchMethodError`

####### 准备 Preparation

在方法区中，为类变量分配内存，并设置初始值。

####### 解析 Resolution

将常量池中的`符号引用` 替换为 `直接引用`

###### 初始化 Initialization

初始化是执行类构造器<clinit>()方法的过程。
- clinit包括类变量赋值动作、静态语句块。
- clinit保证多线程环境中被加锁、同步 >> 可用来实现单例  

5中情况必须立即对类进行初始化：
- new, getstatic
- 反射调用
- 先触发父类初始化
- main
- java7 REF_getStatic ??

反例：
- `SubClass.staticValue` 通过子类引用父类静态字段，子类不会被初始化
- `MyClass[]` 通过数组定义来引用类，不会触发类初始化
- `MyCalss.CONSTANT_VALUE` 引用常量，不会触发类初始化


###### 使用 Using

###### 卸载 Unloading

##### 双亲委派模型

###### 类加载器

####### Bootstrap ClassLoader

/li

####### Extension ClassLoader

/lib/ext

####### Application ClassLoader

classpath



####### 自定义：重写findClass, 而非loadClass

###### 被破坏

####### jdk1.2之前

####### JNDI等SPI框架需要加载classpath代码

####### OSGi: 网状

##### 异常

###### ClassNotFoundException

####### 当动态加载Class的时候找不到类会抛出该异常

####### 一般在执行Class.forName()、ClassLoader.loadClass()或ClassLoader.findSystemClass()的时候抛出

###### NoClassDefFoundError

####### 编译成功以后，执行过程中Class找不到

####### 由JVM的运行时系统抛出

### GC

#### 引用类型

##### 种类

###### 强引用 Strong Reference

###### 软引用 SoftReference

####### 发生内存溢出之前，会尝试回收

####### 应用场景：可用于实现内存敏感的缓存

###### 弱引用 WeakReference

####### 下一次垃圾回收之前，一定会被回收

####### 可用来构建一种没有特定约束的关系，例如维护非强制性的映射关系

####### 应用场景：缓存

###### 虚引用 PhantomReference 

####### 不能通过它访问对象

######## 需要配合ReferenceQueue使用

```java
Object counter = new Object();
ReferenceQueue refQueue = new ReferenceQueue<>();

PhantomReference<Object> p = new PhantomReference<>(counter, refQueue);

counter = null;
System.gc();

try {
    // Remove获取对象。
    // 可以指定 timeout，或者选择一直阻塞
    Reference<Object> ref = refQueue.remove(1000L);
    if (ref != null) {
        // do something
    }
} catch (InterruptedException e) { }

```

####### 虚引用仅提供一种确保对象被finalize后做某些事的机制

####### finalize 之后会变成虚引用

####### 应用场景：跟踪对象被垃圾回收器回收的活动

##### 实践

###### -XX:+PrintReferenceGC 打印各种引用数量

###### Reference.reachabilityFence(this)

####### 声明对象强可达

```java
class Resource {

 public void action() {
 try {
     // 需要被保护的代码
     int i = myIndex;
     Resource.update(externalResourceArray[i]);
 } finally {
     // 调用 reachbilityFence，明确保障对象 strongly reachable
     Reference.reachabilityFence(this);
 }
 
 
 // 调用
 new Resource().action();

```

#### GC回收判断

##### 引用计数法

###### 无法解决循环引用问题

##### 可达性分析

###### GC Roots

####### 虚拟机栈中，本地变量表引用的对象

####### 本地方法栈中，JNI引用的对象

####### 方法区中，类静态属性引用的对象

####### 方法区中，常量引用的对象

#### GC算法

##### 标记清除 Mark-Sweep

- 标记出所有需要回收的对象
- 标记完成后统一回收被标记的对象


###### 效率问题

###### 空间问题

##### 复制算法 Copying

- 将可用内存划分为两块；
- 当一块用完时，将存活对象复制到另一块内存


###### 问题：内存使用率

###### 适用：存活率低的场景

###### 例子：Eden + 2 Suvivor

##### 标记整理 Mark-Compact

- 标记出所有需要回收的对象
- 让所有存活对象都向一端移动	

###### 例子：老年代

#### GC收集器

##### 实现

###### 枚举根节点

- 会发生停顿
- OopMap: 虚拟机知道哪些地方存放着对象引用

###### 安全点

程序执行时，只有达到安全点时才能暂停：
- 方法调用
- 循环跳转
- 异常跳转


####### 抢先式中断

少用。
- 先中断全部线程
- 如果发现有线程中断的地方不在安全点上，则恢复线程


####### 主动式中断

- GC简单地设置一个标志
- 线程执行时主动轮询这个标志，当为真时则自己中断挂起

###### 安全区域

在一段代码片段中，引用关系不会发生变化，在这个区域中任意地方开始GC都是安全的。

##### Serial (Serial Old)

###### Stop The World

###### 新生代：复制算法

###### 老年代：标记整理

##### ParNew

###### 新生代：多个GC线程

###### 其余和Serial一致，STW

##### Parallel Scavenge (Parallel Old)

###### 目标：吞吐量；而其他算法关注缩短停顿时间

###### GC停顿时间会牺牲吞吐量和新生代空间

###### 适合后台运算任务，不需要太多的交互

###### 可设置MaxGCPauseMillis, GCTimeRatio

###### 其余类似ParNew

##### CMS, Concurrent Mark Sweep

###### 目标：减少回收停顿时间

###### 标记清除算法（其他收集器用标记整理）

###### 步骤

####### 1.初始标记 (单线程，STW)

- Stop The World
- 标记GC roots直接关联到的对象

####### 2.并发标记 (单线程，并发)

- 耗时长，但与用户线程一起工作
- GC Roots Tracing 可达性分析

####### 3.重新标记 (多线程，STW)

- Stop The World
- 修正上一步并发标记期间发生变动的对象

####### 4.并发清除 (单线程，并发)

- 与用户线程一起工作

###### 缺点

####### CPU资源敏感，吞吐量会降低

并发阶段会占用线程

####### 无法处理浮动垃圾，可能Concurrent Mode Failure

- CMS不能等到老年代几乎满时再收集，要预留空间给浮动垃圾；
- 否则会导致预留的内存无法满足程序需要，出现Concurrent Mode Failure，临时启用Serial Old收集，停顿时间长。  

`CMSInitiatingOccupancyFraction`不宜过高

####### 标记清除，产生内存碎片

##### G1

###### 特点

####### 并行与并发

####### 分代收集

G1将堆划分为大小相等的`Region`，新生代和老年代不再是物理隔离的

####### 空间整合 -标记整理

####### 可预测的停顿

允许使用者明确指定M毫秒内GC时间不得超过N毫秒

###### 步骤

####### 1.初始标记 (单线程，STW)

####### 2.并发标记 (单线程，并发)

- 耗时长，但与用户线程一起工作
- GC Roots Tracing 可达性分析

####### 3.最终标记 (多线程，STW)

####### 4.筛选回收 (多线程)

#### 内存分配策略

##### 对象优先在Eden分配

如果Eden空间不足，则出发MinorGC


##### 大对象直接进入老年代

PretenureSizeThreshold

##### 长期存活的对象进入老年代

MaxTenuringThreshold

##### 动态对象年龄判断

如果Survivor中相同年龄对象的总大小 > Survivor的一半；则该年龄及更老的对象 直接进入lao'nian'dai

##### 空间分配担保: 可能触发FullGC

- MinorGC之前，检查老年代最大可用的连续空间，是否大于新生代所有对象总空间。如果大于，则MinorGC是安全的。
- 否则要进行一次FullGCC.

### 工具

#### 命令行工具

##### jps -l

##### jstat -class/-gc

jstat -gc vmid [interval] [count]

- class
- gc
- gccause



##### jinfo: 查看修改VM参数

修改参数：
- jinfo -flag name=value
- jinfo -flag[+|-] name

##### jmap: 生成heap dump

jmap -dump:live,format=b,file=/pang/logs/tomcat/heapdump.bin 1

##### jhat: 简单分析heap dump 

##### jstack -l: 生成thread dump

##### jcmd

`jcmd 1 GC.heap_dump ${dump_file_name}`

JFR.stop
JFR.start
JFR.dump
JFR.check
VM.native_memory
VM.check_commercial_features
VM.unlock_commercial_features
ManagementAgent.stop
ManagementAgent.start_local
ManagementAgent.start
GC.rotate_log
Thread.print
GC.class_stats
GC.class_histogram
GC.heap_dump
GC.run_finalization
GC.run
VM.uptime
VM.flags
VM.system_properties
VM.command_line
VM.version
help

##### javap: 分析class文件字节码

`javap -verbose TestClass`

#### GUI工具

##### jconsole

查看MBean


###### 内存：jstat

###### 线程：jstack

##### VisualVM

###### 插件

###### 监视：dump

###### Profiler

####### CPU

####### 内存

###### BTrace

#### instrument

##### ClassFileTransformer

##### Instrumentation

### 参数

#### 运维

##### -XX:+HeapDumpOnOutOfMemoryError

##### -Xverify:none -忽略字节码校验

##### -Xint -禁止JIT编译器

#### 内存

##### -XX: SurvivorRatio

##### -XX:MaxTenuringThreshold

##### -XX:+AlwaysTenure -去掉Suvivor空间

##### -Xms -最小堆

##### -Xmx -最大堆

##### -Xmn -新生代

##### -Xss -栈内存大小

##### -XX:PermSize  -XX:MaxPermSize

##### -XX:MaxDirectMemorySize -直接内存

#### GC

##### -XX:+DisableExplicitGC -忽略System.gc()

##### -XX:+PrintGCApplicationStoppedTime

##### -XX:+PrintGCDateStamps

##### -XX:+PrintReferenceGC 打印各种引用数量

##### -Xloggc:xx.log

## 并发

### 基础

#### 并发的问题

##### 微观

###### 可见性问题：缓存导致

###### 原子性问题：线程切换导致 （e.g. i++）

###### 有序性问题：编译优化导致

##### 宏观

###### 安全性问题

####### Data Race -数据竞争

######## 同时修改共享数据

####### Race Condition -竞态条件

######## 程序的执行结果依赖线程执行的顺序

####### 解决：互斥（锁）

###### 活跃性问题

####### 概念：指某个操作无法执行下去

####### 死锁

####### 活锁

######## 张猫

######## 解决：等待随机时间

####### 饥饿

######## 线程因为无法访问所需资源而无法执行下去

######## 解决：保证资源充足、公平分配资源（公平锁）、避免持有锁的线程长时间执行

###### 性能问题

####### 串行比不能过高

####### 解决

######## 无锁

######### Thread Local Storage

######### Copy on write

######### 乐观锁

######### 原子类

######## 减少锁持有的时间

######### ConcurrentHashMap

######### 读写锁

#### happens-before

##### 含义：前一个操作的结果对后一个操作是可见的。

##### 1. 程序顺序性规则

与编译器重排序 冲突？

###### 前面的操作 happens-before后续的任意操作

##### 2. volatile变量规则

###### 对volatile的写操作 happens-before后续对这个变量的读操作

##### 3. 传递性

##### 4. 管程中锁的规则

- synchronized是Java对管程的实现。

###### 对一个锁的解锁 happens-before 后续对这个锁的加锁

##### 5. 线程start()规则

###### thread-A call thread-B.start()，start()执行前A的操作 happens-before B中的任意操作

##### 6. 线程join()规则

###### thread-A call thread-B.join()，B中的任意操作 happens-before join方法操作的返回

#### 死锁

##### 发生条件

###### 1. 互斥

####### 共享资源X/Y只能被一个线程占用

####### 避免：不可避免

###### 2. 占有且等待

####### 线程取得X的锁后等待Y的过程中，不会释放X的锁

####### 避免：一次性申请所有资源

###### 3. 不可抢占

####### 占有的资源不可被其他线程抢占

####### 避免：如果等待不到，则主动释放已占有的锁 （Lock  timeout）

###### 4. 循环等待

####### 互相等待

####### 避免：按照资源序号申请

##### 避免死锁

###### 其实就是破坏发生条件

###### 1. 互斥

#### Thread

##### 线程状态

###### 初始

####### New

###### 可运行

####### Runnable

###### 运行

####### Runnable

###### 休眠

####### Blocked

######## 等待锁

####### Waiting

######## wait()

######## join()

######## LockSupport.park()

####### Timed_Waiting

######## sleep(ms)

######## wait(timeout)

######## join(ms)

########  parkNanos/parkUntil

###### 终止

####### Terminated

######## 线程执行结束

######## 线程异常

######### interrupt()

########## 异常

########### 如果处于Waiting/Timed_Waiting，interrupt()则会触发InterruptedException

所以 wait(), join(), sleep()方法声明都有throws InterruptedException

########### 如果处于Runnable状态，并阻塞在InterruptibleChannel上；则会触发ClosedByInterruptException

########### 如果处于Runnable状态，并阻塞在Selector上；则Selector会立即返回

########## 主动监测

########### if isInterrupted()

######### 不可用stop()!! 其不会释放锁

### 分工

#### Executor / 线程池

##### execute() vs. submit()

###### execute不需要返回值，无法判断任务是否执行成功

###### submit 返回 Future

##### ThreadPoolExecutor

###### corePoolSize

####### 最小线程数

###### maximumPoolSize

####### 最大线程数

###### keepAliveTime & unit

####### 判断是否空闲的依据

###### workQueue

####### 工作队列

####### 慎用无界队列，避免OOM

###### threadFactory

###### handler: 拒绝策略

####### CallerRunsPolicy

######## 提交任务的线程去执行

####### AbortPolicy

######## 抛出 RejectedExecutionException，默认值

######## 慎用，因为并不强制要catch exception，很容易被忽略

######## 一般与降级策略配合

####### DiscardOldestPolicy

######## 丢弃最老的任务

##### 线程池大小设置

###### CPU密集型任务：线程池尽可能小

####### CPU核数 + 1 

###### IO密集型任务：线程池尽可能大

####### CPU核数 * (1 + IO耗时/CPU耗时 )

##### 与一般池化资源的区别

###### 本质是 生产者-消费者 模式

####### 生产者：线程使用方

####### 消费者：线程池本身

###### 并非 acquire / release

#### Future

##### 作用：类比提货单

##### ThreadPoolExecutor.submit(xx)

###### Runnable

###### Callable

###### Runnable, T

```java
ExecutorService executor  = Executors.newFixedThreadPool(1);

// 创建 Result 对象 r
Result r = new Result();
r.setAAA(a);

// 提交任务
Future<Result> future = 
  executor.submit(new Task(r), r);  
Result fr = future.get();

// 下面等式成立
fr === r;
fr.getAAA() === a;
fr.getXXX() === x

class Task implements Runnable {
  Result r;
  // 通过构造函数传入 result
  Task(Result r){
    this.r = r;
  }
  
  void run() {
    // 可以操作 result
    a = r.getAAA();
    r.setXXX(x);
  }
}

```

##### 方法

// 取消任务
boolean cancel(boolean mayInterruptIfRunning);

// 判断任务是否已取消  
boolean isCancelled();

// 判断任务是否已结束
boolean isDone();

// 获得任务执行结果
get();

// 获得任务执行结果，支持超时
get(long timeout, TimeUnit unit);


###### cancel

###### isCancelled

###### isDone

###### get, get(timeout)

##### FutureTask 工具类

###### 实现接口

####### Runnable

######## 所以可以将FutureTask提交给Executor去执行

####### Future

######## 所以可以获取任务的执行结果

```java
// 创建 FutureTask
FutureTask<Integer> futureTask = new FutureTask<>(()-> 1+2);

// 创建线程池
ExecutorService es = 
  Executors.newCachedThreadPool();
  
// 提交 FutureTask 
es.submit(futureTask);

// 获取计算结果
Integer result = futureTask.get();

```

###### RunnableAdapter

####### 创建：Executors#callable(Runnable, T)

####### 原理：适配器模式

```java
class RunnableAdapter<T> implements Callable<T> {

        final Runnable task;
        final T result;
        
        public T call() {
            task.run();
            return result;
        }
    }
 ```

#### CompletableFuture

##### 创建

###### 四个静态方法

####### runAsync(Runnable, [Executor])

####### supplyAsync(Supplier, [Executor])

###### 默认使用公共的ForkJoinPool线程池

####### 默认线程数 == CPU核数

####### 强烈建议不同业务用不同线程池

##### 实现接口

###### Future

####### 解决两个问题

######## 异步操作什么时候结束

######## 如何获取异步操作的执行结果

###### CompletionStage

####### 描述串行关系

```java
CompletableFuture<String> f0 = 
  CompletableFuture.supplyAsync(
    () -> "Hello World")      //①
  .thenApply(s -> s + " QQ")  //②
  .thenApply(String::toUpperCase);//③

System.out.println(f0.join());
```

// 输出结果: 异步流程，但是顺序执行
HELLO WORLD QQ

######## thenApply(Function..), xxAsync

######### 返回值CompletionStage<R>

######## thenAccept(Consumer..), xxAsync

######### 返回值CompletionStage<Void>

######## thenRun(Runnable..), xxAsync

######### 返回值CompletionStage<Void>

######## thenCompose(Function..), xxAsync

######### 返回值CompletionStage<R>

######### 新创建一个子流程，最终结果和 thenApply 相同

####### 描述 AND 聚合关系

######## thenCombine(other, Function), xxAsync

######### R

######## thenAcceptBoth(other, Consumer), xxAsync

######### Void

######## runAfterBoth(other, Runnable), xxAsync

######### Void

####### 描述 OR 聚合关系

```java
CompletableFuture<String> f1 = 
  CompletableFuture.supplyAsync(()->{
    int t = getRandom(5, 10);
    sleep(t, TimeUnit.SECONDS);
    return String.valueOf(t);
});

CompletableFuture<String> f2 = 
  CompletableFuture.supplyAsync(()->{
    int t = getRandom(5, 10);
    sleep(t, TimeUnit.SECONDS);
    return String.valueOf(t);
});

CompletableFuture<String> f3 = 
  f1.applyToEither(f2,s -> s);

System.out.println(f3.join());

```

######## applyToEither(other, Function), xxAsync

######### R

######## acceptEither(other, Consumer), xxAsync

######### Void

######## runAfterEither(other, Runnable), xxAsync

######### Void

####### 异常处理

```java
CompletableFuture<Integer> f0 = CompletableFuture
    .supplyAsync(()->7/0))
    .thenApply(r->r*10)
    .exceptionally(e->0);
    
System.out.println(f0.join());

```

######## exceptionally(Function)

######### 类似catch

######## whenComplete(Consumer), xxAsync

######### 类似finally

######### 不支持返回结果

######## handle(Function), xxAsync

######### 类似finally

######### 支持返回结果

#### CompletionService

##### 线程池 + Future任务 + 阻塞队列

CompletionService 内部维护了一个阻塞队列，当任务执行结束就把任务的执行结果加Future加入到阻塞队列中

##### 示例

```java
// 创建线程池
ExecutorService executor = Executors.newFixedThreadPool(3);
  
// 创建 CompletionService
CompletionService<Integer> cs = new ExecutorCompletionService<>(executor);

// 异步向电商询价
cs.submit(()->getPriceByS1());
cs.submit(()->getPriceByS2());
cs.submit(()->getPriceByS3());

// 将询价结果异步保存到数据库
for (int i=0; i<3; i++) {
  Integer r = cs.take().get();
  executor.execute(()->save(r));
}

```

##### 创建

###### ExecutorCompletionService(Executor)

####### 默认用无界 LinkedBlockingQueue

###### ExecutorCompletionService(Executor, BlockingQueue)

##### 方法

###### submit(Callable) / submit(Runnable, V)

###### take()

####### 如果队列为空，则阻塞

###### poll()

####### 如果队列为空，则返回null

#### Fork / Join

##### 相当于单机版的MapReduce

##### 分治任务模型

###### 任务分解：Fork

###### 结果合并：Join

##### ForkJoin 计算框架

###### 线程池 ForkJoinPool

####### 方法

######## invoke(ForkJoinTask)

######## submit(Callable)

####### 原理

######## ForJoinPool内部有多个任务队列

######### （而ThreadPoolExecutor只有一个队列）

######## 任务窃取

######### 双端队列

###### 分治任务 ForkJoinTask

####### 方法

######## fork()

######### 异步执行一个子任务

######## join()

######### 阻塞当前线程等待子任务执行结束

####### 子类

######## RecursiveAction

######### compute() 无返回值

######## RecursiveTask

######### compute() 有返回值

###### 示例

####### 计算斐波那契数列

```java
static void main(String[] args){
  // 创建分治任务线程池  
  ForkJoinPool fjp = new ForkJoinPool(4);
    
  // 创建分治任务
  Fibonacci fib = new Fibonacci(30);
    
  // 启动分治任务  
  Integer result = fjp.invoke(fib);
  
  // 输出结果  
  System.out.println(result);
}

// 递归任务
static class Fibonacci extends RecursiveTask<Integer> {
  final int n;
  
  protected Integer compute(){
    if (n <= 1)
      return n;
    Fibonacci f1 = new Fibonacci(n - 1);
    
    // 创建子任务  
    f1.fork();
    
    Fibonacci f2 = new Fibonacci(n - 2);
    
    // 等待子任务结果，并合并结果  
    return f2.compute() + f1.join();
  }
}

```

####### 统计单词数量

```java
static void main(String[] args){
  String[] fc = {"hello world",
          "hello me",
          "hello fork",
          "hello join",
          "fork join in world"};
          
  // 创建 ForkJoin 线程池    
  ForkJoinPool fjp = new ForkJoinPool(3);
  
  // 创建任务    
  MR mr = new MR(fc, 0, fc.length);  
  // 启动任务    
  Map<String, Long> result = fjp.invoke(mr);
  
  // 输出结果    
 
}

//MR 模拟类
static class MR extends RecursiveTask<Map<String, Long>> {
  private String[] fc;
  private int start, end;
  
  @Override protected Map<String, Long> compute(){
  
    if (end - start == 1) {
      return calc(fc[start]);
    } else {
      int mid = (start+end)/2;
      MR mr1 = new MR(fc, start, mid);
      
      mr1.fork();
      MR mr2 = new MR(fc, mid, end);
      // 计算子任务，并返回合并的结果    
      return merge(mr2.compute(),  mr1.join());
    }
  }
  
  // 合并结果
  private Map<String, Long> merge(
      Map<String, Long> r1, 
      Map<String, Long> r2) {
    Map<String, Long> result = new HashMap<>();
    result.putAll(r1);
    // 合并结果
    r2.forEach((k, v) -> {
      Long c = result.get(k);
      if (c != null)
        result.put(k, c+v);
      else 
        result.put(k, v);
    });
    return result;
  }
  
  // 统计单词数量
  private Map<String, Long> calc(String line) {
    Map<String, Long> result =  new HashMap<>();
    // 分割单词    
    String [] words = line.split("\\s+");
    // 统计单词数量    
    for (String w : words) {
      Long v = result.get(w);
      if (v != null) 
        result.put(w, v+1);
      else
        result.put(w, 1L);
    }
    return result;
  }
}

```

#### 模式

##### Guarded Suspension模式

##### Balking模式

##### Thread-Per-Message模式

##### 生产者消费者模式

##### Worker Thread模式

##### 两阶段终止模式

### 协作

#### Monitor 管程

##### 概念

###### 管程：管理共享变量、以及对共享变量的操作过程，让他们支持并发

###### synchronized, wait(), notify(), notifyAll()都是管程的组成部分

##### 实现

###### MESA模型

###### 互斥

####### 讲共享变量及其操作封装起来

###### 同步

####### 入口等待队列

####### 条件变量等待队列

######## wait(): 加入条件变量等待队列

######## notify(): 从条件变量等待队列，进入入口等待队列

#### CountDownLatch

##### 场景：一个线程等待多个线程

##### 计数器不能循环再利用

##### 方法

```java

// 创建 2 个线程的线程池
Executor executor = Executors.newFixedThreadPool(2);

while(存在未对账订单){
  // 计数器初始化为 2
  CountDownLatch latch = new CountDownLatch(2);
    
  // 查询未对账订单
  executor.execute(()-> {
    pos = getPOrders();
    latch.countDown();
  });
  
  // 查询派送单
  executor.execute(()-> {
    dos = getDOrders();
    latch.countDown();
  });
  
  // 等待两个查询操作结束
  latch.await();
  
  // 执行对账操作
  diff = check(pos, dos);
  // 差异写入差异库
  save(diff);
}


```

###### countDown()

###### await()

#### CyclicBarrier

##### 场景：一组线程之间互相等待

##### 计数器会循环利用：当减到0后，会自动重置到初始值

##### 可以设置回调函数

##### 方法

```java

// 订单队列
Vector<P> pos;
// 派送单队列
Vector<D> dos;

// 执行回调的线程池 
Executor executor = Executors.newFixedThreadPool(1);
final CyclicBarrier barrier =
  new CyclicBarrier(2, ()->{
    executor.execute(()-> check());
  });
  
void check(){
  P p = pos.remove(0);
  D d = dos.remove(0);
  // 执行对账操作
  diff = check(p, d);
  // 差异写入差异库
  save(diff);
}
  
void checkAll(){

  // 循环查询订单库
  Thread T1 = new Thread(()->{
    while(存在未对账订单){
      // 查询订单库
      pos.add(getPOrders());
      // 等待
      barrier.await();
    }
  }
  T1.start(); 
  
  // 循环查询运单库
  Thread T2 = new Thread(()->{
    while(存在未对账订单){
      // 查询运单库
      dos.add(getDOrders());
      // 等待
      barrier.await();
    }
  }
  T2.start();
}

```

###### await()

#### Semaphore

##### 场景：限流器，不允许多于N个线程同时进入临界区

##### 方法

###### acquire()

###### release()

#### Phaser

#### Exchanger

#### wait / notify

##### 机制

###### 线程首先获取互斥锁；

###### 当线程要求的条件不满足时，释放互斥锁，进入等待状态；

###### 当条件满足后，通知等待的线程，重新获取互斥锁

##### 实现

###### 前提是先获取锁，所以wait/notify/notifyAll必须在synchronized内部调用

否则会IllegalMonitorStateException

###### wait 要在while循环内部调用

因为被唤醒时表示`条件曾经满足`，但wait返回时条件可能出现变化。

###### 尽量用notifyAll

####### 何时可用notify?

######## 所有等待线程拥有相同的等待条件

######## 所有等待线程被唤醒后，执行相同的操作

######## 只需要唤醒一个线程

##### wait vs. sleep

###### wait会释放锁

###### wait必须在同步块中使用

###### wait不抛异常

#### Condition

##### Condition实现了管程模型里的条件变量

##### await() / signal() / signalAll()

##### 示例

###### 使用Condition实现阻塞队列

``` java
public class BlockedQueue<T>{
  final Lock lock =
    new ReentrantLock();
  // 条件变量：队列不满  
  final Condition notFull =
    lock.newCondition();
  // 条件变量：队列不空  
  final Condition notEmpty =
    lock.newCondition();

  // 入队
  void enq(T x) {
    lock.lock();
    try {
      while (队列已满){
        // 等待队列不满
        notFull.await();
      }  
      // 省略入队操作...
      // 入队后, 通知可出队
      notEmpty.signal();
    }finally {
      lock.unlock();
    }
  }
  // 出队
  void deq(){
    lock.lock();
    try {
      while (队列已空){
        // 等待队列不空
        notEmpty.await();
      }  
      // 省略出队操作...
      // 出队后，通知可入队
      notFull.signal();
    }finally {
      lock.unlock();
    }  
  }
}
```


###### 使用Condition实现异步转同步

``` java
// 创建锁与条件变量
private final Lock lock 
    = new ReentrantLock();
private final Condition done 
    = lock.newCondition();

// 调用方通过该方法等待结果
Object get(int timeout){
  long start = System.nanoTime();
  lock.lock();
  try {
	while (!isDone()) {
	  done.await(timeout);
      long cur=System.nanoTime();
	  if (isDone() || 
          cur-start > timeout){
	    break;
	  }
	}
  } finally {
	lock.unlock();
  }
  if (!isDone()) {
	throw new TimeoutException();
  }
  return returnFromResponse();
}
// RPC 结果是否已经返回
boolean isDone() {
  return response != null;
}
// RPC 结果返回时调用该方法   
private void doReceived(Response res) {
  lock.lock();
  try {
    response = res;
    if (done != null) {
      done.signal();
    }
  } finally {
    lock.unlock();
  }
}

```

### 互斥

#### 无锁

##### 不变模式

###### 对象创建后状态不再发生变化

####### 实现

######## final 类

######## final 字段

######## 所有方法均是只读的

####### 保护性拷贝

####### 避免创建重复对象：享元模式 Flyweight Pattern

######## 本质是对象池

######## Long 缓存[-127, 127]

###### 无状态

####### 没有属性，只有方法

##### 线程封闭

###### 不和其他线程共享变量

##### 线程本地存储 ThreadLocal

###### 例子

####### 线程ID生成器

```java
static class ThreadId {
  static final AtomicLong nextId=new AtomicLong(0);
 
  static final ThreadLocal<Long> tl=ThreadLocal.withInitial(
    ()->nextId.getAndIncrement());
  // 为每个线程分配一个唯一的 Id
  static long get(){
    return tl.get();
  }
}

```

###### 原理

####### 方案一：ThreadLocal 引用Map<Thread, T>

####### 方案二：Thread引用Map<ThreadLocal, T>

```java
class Thread {
  ThreadLocal.ThreadLocalMap threadLocals;
}

class ThreadLocal<T>{
  public T get() {
    ThreadLocalMap map = Thread.currentThread().threadLocals;

    Entry e = map.getEntry(this);
    return e.value;  
  }
  
  static class ThreadLocalMap{
    // 内部是数组而不是 Map
    Entry[] table;
    
    static class Entry extends WeakReference<ThreadLocal>{
      Object value;
    }
  }
}

```

####### 方案二的好处

######## 更合理：所有和线程相关的数据都存储在 Thread 里面

######## 不容易内存泄露

方案一，只要ThreadLocal对象存在，那么Map中的Thread对象就永远不会回收

###### 注意

####### 线程池中使用ThreadLocal可能导致内存泄露

- 线程池中线程存活时间长；
- 所以Thread.ThreadLocalMap不会被回收；


####### 手动释放：ThreadLocal.remove()

##### CAS

##### Copy-on-Write

###### 场景

####### 读多写少

####### 弱一致性

####### 对读的性能要求很高

####### 函数式编程

###### 案例

####### RPC 客户端存储 路由哈希表

####### ConcurrentHashMap<String, CopyOnWriteArraySet<Router>>

```java
// 路由信息： Immutable
public final class Router{
  private final String  ip;
  private final Integer port;
  private final String  iface
}

// 路由表信息
public class RouterTable {
  //Key: 接口名
  //Value: 路由集合
  ConcurrentHashMap<String, CopyOnWriteArraySet<Router>> 
    rt = new ConcurrentHashMap<>();
    
  // 根据接口名获取路由表
  public Set<Router> get(String iface){
    return rt.get(iface);
  }
  
  // 删除路由
  public void remove(Router router) {
    Set<Router> set=rt.get(router.iface);
    if (set != null) {
      set.remove(router);
    }
  }
  
  // 增加路由
  public void add(Router router) {
    Set<Router> set = rt.computeIfAbsent(
      route.iface, r -> 
        new CopyOnWriteArraySet<>());
    set.add(router);
  }
}

```

####### CopyOnWriteArraySet, CopyOnWriteArrayList

###### 缺点

####### 消耗内存 (并非按需复制)

##### Atomic原子类

###### 类型

####### 基本类型 

######## AtomicLong

######## AtomicInteger

######## AutomicBoolean

####### 数组类型

AtomicIntegerArray, AtomicLongArray, Atomic ReferenceArray

######## AtomicIntegerArray

######## AtomicReferenceArray

####### 引用类型

######## AtomicReference

######### 例如 原子性设置upper/lower

```java
public class SafeWM {
  class WMRange{
    final int upper;
    final int lower;
  }
  
  final AtomicReference<WMRange>
    rf = new AtomicReference<>(
      new WMRange(0,0)
    );
    
  // 设置库存上限
  void setUpper(int v){
    while(true){
      WMRange or = rf.get();
      // 检查参数合法性
      if(v < or.lower){
        throw new IllegalArgumentException();
      }
      WMRange nr = new WMRange(v, or.lower);
      if(rf.compareAndSet(or, nr)){
        return;
      }
    }
  }
}

```

######## AtomicStampedReference

boolean compareAndSet(
  V expectedReference,
  V newReference,
  int expectedStamp,
  int newStamp) 


######### 解决ABA问题

######### 增加版本号stamp

######## AtomicMarkableReference

boolean compareAndSet(
  V expectedReference,
  V newReference,
  boolean expectedMark,
  boolean newMark)


######### 解决ABA问题

######### 将版本号简化为boolean

####### 对象属性修改

######## AtomicIntegerFieldUpdater

######## AtomicReferenceFieldUpdater

######## 注意

######### 对象属性必须是volatile

######### 示例

```java
public static <U>
AtomicXXXFieldUpdater<U> 
newUpdater(
  Class<U> tclass, //类信息
  String fieldName)
  

boolean compareAndSet(
  T obj, //对象信息
  int expect, 
  int update)
```

####### 累加器

######## LongAdder

######## LongAccumulator

######## 注意

######### 不支持compareAndSet

######### 性能更好

###### 原理

####### CAS + volatile

- 利用 `CAS` (compare and swap) + `volatile` 和 native 方法来保证原子操作，从而避免 `synchronized` 的高开销，执行效率大为提升。

- CAS的原理是拿期望的值和原本的一个值作比较，如果相同则更新成新的值。


`UnSafe.objectFieldOffset()` 方法是一个本地方法，这个方法是用来拿到“原来的值”的内存地址，返回值是 valueOffset。另外 value 是一个volatile变量，在内存中可见，因此 JVM 可以保证任何时刻任何线程总能拿到该变量的最新值。

######## 只有 currentValue == expectedValue，才会将其更新为newValue

######## 简单示例代码：自旋


```java
class SimulatedCAS {
  volatile int count;
  // 实现 count+=1
  addOne() {
    do {
      newValue = count+1; //①
    } while (count !=
      cas(count,newValue) //②
  }
  
  // 模拟实现 CAS，仅用来帮助理解
  synchronized int cas (
    int expect, int newValue){

    int curValue = count;
    if(curValue == expect){
      count= newValue;
    }
    return curValue;
  }
}
``


######## 真实示例代码：Unsafe.getAndAddLong()

```java
public final long getAndAddLong(
  Object o, long offset, long delta){
  long v;
  do {
    // 读取内存中的值
    v = getLongVolatile(o, offset);
  } while (!compareAndSwapLong(
      o, offset, v, v + delta));
  return v;
}

// 原子性地将变量更新为 x
// 条件是内存中的值等于 expected
// 更新成功则返回 true
native boolean compareAndSwapLong(
  Object o, long offset, 
  long expected,
  long x);

```

####### ABA 问题

##### 并发容器

###### 同步容器

####### Collections.synchronizedList / Set / Map

####### Vector / Stack / Hashtable

###### List

####### CopyOnWriteArrayList

###### Map

####### ConcurrentHashMap

######## 无序

####### ConcurrentSkipListMap

######## 有序

######## 跳表，性能高

###### Set

####### CopyOnWriteArraySet

####### ConcurrentSkipListSet

###### Queue

####### 单端阻塞队列

######## ArrayBlockingQueue

######## LinkedBlockingQueue

####### 双端阻塞队列

######## LinkedBlockingDeque

####### 单端非阻塞队列

######## ConcurrentLinkedQueue

####### 双端非阻塞队列

######## ConcurrentLinkedDeque

#### 互斥锁

##### synchronized

###### 三种使用方式：修饰实例方法、静态方法、代码块

###### 加锁对象

####### synchronized(obj)

####### 如果要保护多个相关的资源，则要选择一个粒度更大的锁，使其覆盖所有资源

###### 原理

####### 同步块：Java对象头中有monitor对象；
monitorenter指令：计数器+1
monitorexit指令：计数器-1

####### 同步方法：ACC_SYNCHRONIZED 标识

###### Java6的优化

####### 偏向锁，轻量级锁，自旋锁；
锁消除，锁粗化

###### 对比ReentrantLock

####### 依赖于JVM vs. 依赖于API

####### Lock增加了高级功能

######## 能够响应中断

######### lockInterruptibly()

######## 支持超时

######### tryLock(timeout)

######## 非阻塞地获取锁

######### tryLock()

######## 可实现公平锁

######### ReentrantLock(boolean fair)

####### 等待通知机制：wait/notify vs. condition

####### 性能已不是选择标准

###### 对比volatile

####### volatile是线程同步的轻量级实现，只能修饰变量

####### volatile只保证可见性，不保证原子性

####### 对线程访问volatile字段不会发生阻塞，而sync会阻塞

##### Lock

###### ReentrantLock

####### ReentrantLock vs. synchronized

######## see above

###### ReadWriteLock

####### 方法：writeLock / readLock

####### 实例：缓存

```java
class CachedData {
  Object data;
  volatile boolean cacheValid;
  final ReadWriteLock rwl =
    new ReentrantReadWriteLock();
  // 读锁  
  final Lock r = rwl.readLock();
  // 写锁
  final Lock w = rwl.writeLock();
  
  void processCachedData() {
    // 获取读锁
    r.lock();
    if (!cacheValid) {
      // 释放读锁，因为不允许读锁的升级
      r.unlock();
      // 获取写锁
      w.lock();
      try {
        // 再次检查状态  
        if (!cacheValid) {
          data = ...
          cacheValid = true;
        }
        // 释放写锁前，降级为读锁
        // 降级是可以的
        r.lock(); ①
      } finally {
        // 释放写锁
        w.unlock(); 
      }
    }
    // 此处仍然持有读锁
    try {use(data);} 
    finally {r.unlock();}
  }
}

```

####### 允许多个线程同时`读`共享变量

######## 适合读多写少场景

####### 升级vs降级

######## 不支持锁的升级：获取读锁后，不能再获取写锁

会导致写锁永久等待。反例：

```java
// 读缓存
r.lock();         ①
try {
  v = m.get(key); ②
  if (v == null) {
    w.lock();
    try {
      // 再次验证并更新缓存
      // 省略详细代码
    } finally{
      w.unlock();
    }
  }
} finally{
  r.unlock();     ③
}

```



######## 但支持锁的降级

####### 读锁不能newCondition 

###### StampedLock

####### 加锁解锁

######## 加锁后返回stamp，解锁时需要传入这个stamp

######## writeLock() / unlockWrite(stamp)

######## readLock() / unlockRead(stamp)

######## validate(stamp)

######### 判断是否已被修改

####### 锁的降级升级

######## 降级：tryConvertToReadLock()

######## 升级：tryConvertToWriteLock()

####### 支持三种锁模式

######## 写锁

######### 类似WriteLock

######## 悲观读锁

######### 类似ReadLock

######## 乐观读

######### tryOptimisticRead()


```java
class Point {

  private int x, y;
  
  final StampedLock sl = new StampedLock();
    
  // 计算到原点的距离  
  int distanceFromOrigin() {
    // 乐观读
    long stamp =  sl.tryOptimisticRead();
    // 读入局部变量，
    // 读的过程数据可能被修改
    int curX = x, curY = y;
    
    // 判断执行读操作期间，
    // 是否存在写操作，如果存在，
    // 则 sl.validate 返回 false
    if (!sl.validate(stamp)){
      // 升级为悲观读锁
      stamp = sl.readLock();
      try {
        curX = x;
        curY = y;
      } finally {
        // 释放悲观读锁
        sl.unlockRead(stamp);
      }
    }
    return Math.sqrt(
      curX * curX + curY * curY);
  }
}

```

######### 允许一个线程同时获取写锁

########## 而在ReadWriteLock中，当多个线程同时读的同时，写操作会被阻塞

######### stamp 类似数据库乐观锁中的version

####### 注意事项

######## 不支持重入

######## 不支持newCondition

####### 示例

######## 读模板


```java
final StampedLock sl = 
  new StampedLock();

// 乐观读
long stamp = 
  sl.tryOptimisticRead();
// 读入方法局部变量
......
// 校验 stamp
if (!sl.validate(stamp)){

  // 升级为悲观读锁
  stamp = sl.readLock();
  try {
    // 读入方法局部变量
    .....
  } finally {
    // 释放悲观读锁
    sl.unlockRead(stamp);
  }
}
// 使用方法局部变量执行业务操作
......

```

######## 写模板


```java
long stamp = sl.writeLock();
try {
  // 写共享变量
  ......
} finally {
  sl.unlockWrite(stamp);
}

```

###### 最佳实践

####### 永远只在更新对象的成员变量时加锁

####### 永远只在访问可变的成员变量时加锁

####### 永远不在调用其他对象的方法时加锁

##### condition（条件变量）

###### 必须在排它锁中使用

###### ReentrantLock

###### 通知机制

### 模式

#### Immutability模式

#### Copy-on-Write模式

#### 线程本地存储模式

#### Guarded Suspension模式

##### 保护性性暂停，又称Guarded Wait, Spin Lock

##### 将异步转换为同步

##### 示例

###### Dubbo

https://github.com/apache/incubator-dubbo/blob/master/dubbo-remoting/dubbo-remoting-api/src/main/java/org/apache/dubbo/remoting/exchange/support/DefaultFuture.java#L168

###### MQ异步返回结果

#### Balking模式

#### Thread-per-Message模式

#### Worker Thread模式

#### 两阶段终止模式

#### 生产者-消费者模式

## Core Java

### Exception

#### 分类

##### Throwable

##### Error

###### 无需捕获

###### 例如OutOfMemoryError

##### Exception

###### Checked Exception

####### 可检查异常，强制要求捕获

####### 例如 IOException

###### Unchecked Exception

####### 运行时异常，不强制要求捕获

####### 例如NPE, ArrayIndexOutOfBoundsException

#### 最佳实践

##### try-with-resources

##### throw early, catch late

###### 对入参判空

###### 重新抛出，在更高层面 有了清晰业务逻辑后，决定合适的处理方式

#### 问题

##### try-catch 会产生额外的性能开销

##### 创建Exception对象会对栈进行快照，耗时

#### 典型题目

##### NoClassDefFoundError vs. ClassNotFoundException

###### ClassNotFoundException

####### 动态加载类时（反射），classpath中未找到

####### 当一个类已经某个类加载器加载到内存中了，此时另一个类加载器又尝试着动态地从同一个包中加载这个类

###### NoClassDefFoundError

####### 编译的时候存在，但运行时却找不到

### Immutable Class

#### final class

#### private final field

#### setter: 不提供

#### getter: copy-on-write

#### 构造对象时，成员变量使用深度拷贝来初始化

### 反射与动态代理

#### 动态代理

##### 实现方式

###### 反射: JDK Proxy

####### implements InvocationHandler

```java
class MyInvocationHandler implements InvocationHandler {
  private Object target;
    
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    System.out.println("Invoking sayHello");
    Object result = method.invoke(target, args);
    return result;
  }
}


```

####### 实现接口

```java
interface Hello {
    void sayHello();
}
class HelloImpl implements  Hello {
    @Override
    public void sayHello() {
        System.out.println("Hello World");
    }
}

```



####### Proxy.newProxyInstance

```java
HelloImpl hello = new HelloImpl();

MyInvocationHandler handler = new MyInvocationHandler(hello);
        
Hello proxyHello = (Hello) Proxy.newProxyInstance(
HelloImpl.class.getClassLoader(), 
HelloImpl.class.getInterfaces(), 
handler);

// 调用代理方法
proxyHello.sayHello();

}
```

###### 字节码操作机制：ASM, cglib

##### 设计模式

###### 代理模式

###### 装饰器模式

### 函数式编程

#### Lambda表达式

##### 闭包

###### lambda表达式引用的是值，而非变量

###### 给变量多次赋值，然后再lambda中引用它，会编译报错

##### 函数接口

###### 即lambda表达式的类型

#### 流

##### 实现机制

###### 惰性求值方法

####### 返回Stream

###### 及早求值方法

####### 返回另一个值

##### 常用流操作

###### collect(toList())

####### 根据Stream里的值生成一个列表

###### map

####### 转换：将Stream中的值转换成一个新的流

###### filter

###### flatMap

####### 用Stream替换值，然后将多个Stream连接成一个Stream

###### min / max(Comparator)

###### reduce

####### 从一组值中生成一个值

## 网络编程

### Netty

#### 通讯方式

##### BIO: 同步阻塞

##### NIO: 同步非阻塞

##### AIO: 异步非阻塞

### Tomcat

#### 配置

##### acceptCount:100

accept队列的长度；默认值是100。

- 类比取号。
- 若超过，进来的连接请求一律被拒绝。

- acceptCount的设置，与应用在连接过高情况下希望做出什么反应有关系。如果设置过大，后面进入的请求等待时间会很长；如果设置过小，后面进入的请求立马返回connection refused。

##### maxConnections

Tomcat在任意时刻接收和处理的最大连接数。-1表示不受限制。

BIO: =maxThreads
NIO: 10000

- 类比买票

- 当接收的连接数达到maxConnections时，Acceptor线程不会读取accept队列中的连接；这时accept队列中的线程会一直阻塞着。



##### maxThreads: 200

请求处理线程的最大数量。默认200。

- 类比观影

### nginx

#### 应用场景

##### 静态资源服务

##### 反向代理服务

##### API服务 -OpenResty

#### tips

##### `cp -r contrib/vim/* ~/.vim` 将vim语法高亮

## 例题

### 基础

#### Exception,  Error

#### final,  finally, finalize

#### 四种引用类型

#### String, StringBuffer, StringBuilder

##### Immutable

##### intern() 缓存，永久代 --> MetaSpace

##### Java9: Compact String, char[] --> byte[]

#### int 与 Integer

##### 自动装箱：Integer.valueOf()

###### 缓存 [-128, 127] 

##### 自动拆箱：Integer.intValue()

##### 避免无意的装箱拆箱行为

##### 局限

###### 非线程安全

###### 原始数据类型与泛型并不能配合

###### 对象数组  数据操作低效，无法利用CPU缓存机制
