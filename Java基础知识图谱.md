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

##### 解释执行

##### 即时编译JIT：针对热点代码

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

##### 强引用

##### 软引用

发生内存溢出之前，会尝试回收

##### 弱引用

下一次垃圾回收之前，一定会被回收


##### 虚引用

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

##### 线程本地存储

###### ThreadLocal

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

## MySql

### 原理

#### 更新

##### update

###### WAL: Write-Ahead-Logging

先写日志，再写磁盘。

InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里

####### 【执行器】先找【引擎】取出行

引擎逻辑：

- 如果这行本来就在内存中，直接返回给执行器；
- 否则，先从磁盘读入内存，再返回。

####### 【执行器】执行更新，【引擎】将新数据更新到内存

####### 【引擎】同时将更新操作记录到 redo log （此时处于prepare状态）

####### 【执行器】生成 bin log，并写入磁盘

####### 【执行器】调用引擎的提交事务接口，【引擎】提交redo log

######## 两阶段提交

redo log 先 prepare， 
再写 binlog，
最后commit redo log

###### redo log

####### InnoDB特有

####### 循环写

######## 默认4个文件，每个1G

######## write pos

######### 当前记录的位置

######## checkpoint

######### 当前要擦除的位置

####### 写入机制

######## innodb_flush_log_at_trx_commit

######### 0: 事务提交时，log写入redo log buffer

######### 1: 事务提交时，log写入disk 【推荐】

######### 2: 事务提交时，log写入FS page cache

######## 后台线程，每1秒将redo log buffer， write到FS page cache，然后fsync持久化到磁盘

###### binlog

####### Server共有

####### 追加写：不会覆盖旧日志

####### 格式

######## statement

######## row

####### 写入机制

######## 事务执行中：把日志写到binlog cache

######## 事务提交时：把binlog cache写到binlog 文件

######### write: 写入文件系统的page cache

######### fsync: 持久化到硬盘

######### sysnc_binlog

########## 0: 每次提交事务，只write不fsync

########## 1: 每次提交事务，都fsync 【推荐】

########## N: 每次提交事务，都write，累积N个后fsync

###### 脏页：内存数据页跟磁盘数据页内容不一致

####### flush: 写回磁盘

####### flush过程会造成抖动

####### 何时触发flush

######## redo log写满（此时系统不再接受更新，更新数跌成0）

######## 系统内存不足

######## 系统空闲时

######## mysql关闭时

##### insert

###### insert ... select

####### 常用于两个表拷贝

####### 对源表加锁 

####### 如果insert和select是同一个表，则可能造成循环写入

###### insert into t ... on duplicate key update d=100

####### 插入主键冲突的话，执行update d=100

#### 查询

##### count(*)

###### MyISAM把总行数存在磁盘上，count(*)效率奇高

###### 效率：count(字段) < count(id) < count(1) = count(*)

count(字段)：满足条件的数据行里，字段不为null的总个数；

count(主键id)：引擎遍历表，取出id返回给server；server判断是否为空，按行累加；

count(1)：引擎遍历表，但不取值，server层对于返回的每一行 放入数字1；

count(*)：并不会取出全部字段，*肯定不是null,直接按行累加。



##### join

###### 分类

####### NLJ

######## Index Nested-Loop Join

######## 先遍历t1 --> 对于每一行，去t2走树搜索

######### 驱动表t1: 全表扫描

######### 被驱动表t2: 树搜索，使用索引！

######## 原则：应该让小表来做驱动表

######## 什么是小表？

######### 考虑条件过滤后、参与join的各个字段总数据量小

####### SNJ

######## Simple Nested-Loop Join

######## 当被驱动表t2不能使用索引时，扫描行数很大

####### BNL

######## Block Nested-Loop Join

######## 读取t1存入join_buffer --> 扫描t2，跟join_buffer数据作对比

######### 如果join_buffer小，则分段存入，驱动表应该选小表

######### 如果join_buffer足够，则驱动表大小无所谓

######## 原则：要尽量避免BNL

###### 优化

####### MRR: Multi-Range Read

######## 按照主键递增顺序查询，则近似顺序读，能提升读性能

####### BKA: Batched Key Access

######## 用于优化 NLJ 

######## 读取t1一批数据 --> 放到临时内存(join_buffer) --> 一起传给t2

####### BNL性能差

######## 多次扫描被驱动表，占用磁盘IO

######## 判断Join条件需要执行M*N次对比，占用CPU

######## BNL 大表Join后对Buffer Pool的影响是持续性的

######### 会占据Buffer Pool Yong区

######### 导致业务正常访问的数据页 没有机会进入Yong区

######## BNL转成BKA

######### 加索引

######### 使用临时表

#### 聚合排序

##### order by

###### sort_buffer

如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助 --> sort_buffer_size

###### 分类

####### 全字段排序

无需回表


####### rowid排序

SET max_length_for_sort_data = 16;
只在buffer中放入id和待排序字段，需要按照id回表


###### 索引

####### 如果有索引：不需要临时表，也不需要排序

####### 进一步优化：覆盖索引

###### 随机

####### order by rand()

####### where id >= @X limit 1

##### group by

###### 机制

####### 内容少：内存临时表

####### 内容多：磁盘临时表

###### 优化

####### 索引

如果对列做运算，可用generated column机制实现列数据的关联更新。

alter table t1 add column z int generated always as(id0), add index(z)


####### 直接排序

select SQL_BIG_RESULT .. from ..
告知mysql结果很大，直接走磁盘临时表。

##### 临时表

###### 例子

####### union

(select 1000 as f)
union
(select id from t1 order by id desc limit 2);

- 创建内存临时表，字段f,是主键；
- 执行第一个子查询，放入1000；
- 执行第二个子查询，获取的数据插入临时表 （主键去重）
- 从临时表按行取出数据。

NOTE: 如果是Union ALL则不会用临时表，因为无需去重，直接依次执行子查询即可。


####### group by

select id as m, 
  count(*) as c,
from t1 
group by m;

创建临时表：字段m, 

### 索引

#### 基础

##### 常见模型

###### 哈希表

####### value存在数组中，用哈希函数把key换算成index

####### 适用于等值查询，范围查询需要全部扫描

###### 有序数组

####### 查询方便，支持范围查询

####### 更新成本高

####### 适用于静态存储

###### 跳表

###### LSM树

###### 搜索树

####### B Tree

M阶B Tree: 
- 每个非叶子结点至多有M个儿子，至少有M/2个儿子；
- 根节点至少有2个儿子；
- 所有叶子节点在同一层。

####### B+ Tree

叶子节点才是真正的原始数据


####### 与二叉树的区别

- 二叉树：优化比较次数
- B/B+树：优化磁盘读写次数

##### 分类

###### 簇索引 -主键索引

每个表至多一个，一般为主键索引。
- B Tree上存储的就是数据本身

###### 非簇索引 -二级索引

- B Tree上存储的是主键
- 需要两次查找才能获取数据本身

###### 唯一索引 vs. 普通索引

####### 性能

######## 读取：类似

######## 更新：普通索引好

####### change buffer

当需要更新一个数据页时，
- 如果数据页在内存中就直接更新
- 而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB 会将这些更新操作缓存在 change buffer 中

这样就不需要从磁盘中读入这个数据页了。在

下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。

######## merge

######### 将change buffer中的操作应用到原数据页

######### 触发时机

########## 访问该数据页时

########## 后台线程定期merge

########## 数据库关闭时

######## 条件：不适用于唯一索引，只限于普通索引

######## 适用场景：`写多读少`

因为 merge 的时候是真正进行数据更新的时刻，而 change buffer 的主要目的就是将记录的变更动作缓存下来，

所以在一个数据页做 merge 之前，change buffer 记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。

######## vs. redo log

######### change buffer 主要节省的是随机读磁盘的 IO 消耗

######### redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写）

######### 写入change buffer后 也是要写redo log的

#### 主键

##### 自增主键

###### 优点

####### 性能好

每次插入一条新记录，都是追加操作，都不涉及到挪动其他记录，也不会触发叶子节点的分裂

######## 插入是追加操作

######## 不挪动其他记录，不触发页分裂

####### 存储空间小

###### 机制

####### show create table命令，AUTO_INCREMENT=2 列出下一个自增值 (Y)

####### 如果插入值为X

######## 当X<Y，则自增值不变

######## 当X>=Y，则自增值更新

###### 不保证连续

插入数据时，是先更新自增值，再执行插入。
如果插入异常，自增值不回退。		

####### 由于主键冲突导致

####### 由于回滚导致

####### 自增值为何不能回退？

假设有并行事务A/B, A申请到id=2，B申请到id=3；
A回退，如果把id也回退为2，那么后续继续插入数据id=3是会冲突。

###### 自增锁

####### innodb_autoinc_lock_mode = 0

######## 语句执行结束才放锁

####### innodb_autoinc_lock_mode = 1

######## 普通insert: 申请自增锁后马上释放

######## insert...select: 语句执行结束才放锁

因为不知道会插入多少行，无法一次性申请id


####### innodb_autoinc_lock_mode = 2

######## 申请自增锁后马上释放

若此时binlog_format = statement，z对于insert...select语句会有数据一致性问题：
- insert into t2 select from t1, 的同时t1表在插入新数据。
- t2 id可能不连续
- 但同步到备库，id连续

####### 建议

######## innodb_autoinc_lock_mode = 2

######## binlog_format = row

######## 既能提升并发性，又不会出现数据一致性问题

##### 业务主键

###### 适用场景

####### 只有一个索引

####### 例如KV

#### 操作

##### 覆盖索引

select primaryKey（而不是*） from xx where indexColumn=
非簇索引节点值为“primaryKey”，而非真实数据记录。避免回表



###### 1. select 主键

###### 2. 返回字段、查询条件 作为联合索引

##### 前缀索引

联合索引(a, b), 只查a也会索引

###### 字符串索引的最左M个字符

####### 字符串类型：用前缀索引，并定义好长度，即可既节省空间、又不用额外增加太多查询成本

如何定义长度:
alter table t add index i_2(email(6))


####### 如何找到合适的长度：区分度

select 
count(distinct email) as L,
count(distinct left(email, 4)) as L4,
count(distinct left(email, 5)) as L5,
...
from User;

假设可接受的损失比是5%，则找出不小于L*95%的值

###### 联合索引的最左N个字段

###### 联合索引的顺序：如果调整顺序可以少维护一个索引，则用该顺序；如不能，则考虑空间优先

空间有效：查询name+age, name, age
- 策略A: (name, age) (age)
- 策略B: (age, name) (name)

应选A, 因为age占用空间geng'x

###### 缺点

####### 前缀索引可能增加扫描行数

####### 无法利用覆盖索引

##### 索引下推

(name, age)
可以适用where name like `张%` AND age=10

直接过滤掉不满足条件age!=10的记录，减少`回表次数`。


#### 索引失效的情况

##### 优化器选择索引

###### 标准

####### 扫描行数

######## 判断扫描行数的依据：区分度cardinality

`show index`可查

######## 区分度通过采样统计得出

######## analyze table t 重新统计索引信息

####### 是否使用临时表

####### 是否排序

####### 是否回表：主键索引无需回表

###### 选错索引的情况

####### 为了避免回表，使用了主键索引；而实际上用where条件中的索引会更快

###### 索引选择非最优时如何补救

####### force index强行选择一个索引

select * from t force index(a) where...

####### 修改语句，引导使用希望的索引

order by b limit 1 --> index b
order by b, a limit 1 --> index a

####### 新建合适的索引

##### 查询语句导致索引失效

###### 条件字段函数操作: where month(t_date) = 7

###### 隐式类型转换: where tradeid = 123 , tradeid is varchar

###### 隐式字符编码转换:  utf8 --> utf8mb4

### 锁

#### 全局锁

##### 命令：Flush tables with read lock (FTWRL)

##### 场景：全库逻辑备份

当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（增删改）、数据定义语句（建表、修改表结构等）和更新类事务的提交语句

###### innodb一般用 `mysqldump -single-transaction`

此命令会在导数据之前启动一个事务，确保拿到一致性视图。
由于MVCC的支持，导出数据过程中可以正常进行更新cao'z。

##### 区别与 set global readonly=true

###### 一些系统会用readonly来判断是否从库

###### 如果客户端异常断开，readonly状态不会重置，风险高

##### 优化：innoDB可在可重复读隔离级别下开启一个事务

#### 表级锁

##### 表锁（lock tables xx read/write）

##### 元数据锁（MDL, metadata lock)

###### 自动加上，无需显式使用

####### 当对一个表做增删改查操作的时候，加 MDL 读锁

####### 当要对表做做结构变更操作的时候，加 MDL 写锁；

###### 增加字段需要MDL写锁，会block业务

####### ALTER +timeout

ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 


####### 执行之前杀掉长事务

#### 行锁

##### MyISAM不支持行锁

##### 两阶段锁协议

###### 行锁是在需要的时候才加上的

###### 但并不是不需要了就立刻释放，而是等到事务结束

##### 最佳实践

###### 要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放

##### 死锁

###### 等待超时

####### innodb_lock_wait_timeout

###### 死锁检测

 

####### innodb_deadlock_detect=on

####### 发现死锁后，主动回滚死锁链条中的某一个事务

####### 问题：耗费 CPU

当更新热点行时，每个新来的线程都要判断会不会由于自己的加入导致死锁。

####### 解决：服务端并发控制

对于相同行的更新，在进入引擎之前排队


### 事务

#### select xx for update: 锁住行

#### where stock=xx: 乐观锁

#### 事务隔离

##### 实现原理

###### MVCC：多版本并发控制

####### 不同时刻启动的事务会有不同的read-view

####### 回滚日志: undo log

###### 查询数据：一致性读（consistent read view）

####### 秒级创建快照

######## 基于row trx_id实现`快照`

######### 事务transaction id 严格递增

######### 事务更新数据时，会把transaction id赋值给数据版本 row trx_id

######## 数据表中的一行记录，其实可能有多个版本（row），每个版本都有自己的row trx_id

####### 用于实现可重复读

######## 如果数据版本是我启动以后才生成的，我就不认，我必须要找到它的上一个版本

####### 根据 row trx_id 、一致性视图（read-view）确定数据版本的可见性

######## 每个事务有一个对应的数组，记录事务启动瞬间所有活跃的事务ID

######### 数组里事务ID最小值：低水位

######### 数组里事务ID最大值+1：高水位

######## 判断可见性

######### row trx_id < 低水位：表示是已提交的事务生成的，对当前事务可见

######### row trx_id > 高水位：表示是有将来启动的事务生成的，不可见

######### row trx_id 在两者之间：

########## 如果row trx_id在数组中，表示是有未提交的事务生成的，不可见

########## 如果row trx_id不在数组中，表示是已提交的事务生成的，可见

######## 判断可见性(简化)

######### 版本未提交：不可见

######### 版本已提交，但在视图创建后提交：不可见

######### 版本已提交，且在视图创建前提交：可见

###### 更新数据：当前读（current read ）

####### 更新数据都是先读后写的

######## 而这个读，只能读`当前的值`，即当前读current read

######## 并非一致性读！！不会去判断row trx_id

####### 当前读必须要读最新版本

######## 所以必须加锁

######## 如果此时有其他事务更新此行 但未提交，则行锁被占用，当前事务会等待锁

####### 除了 update 语句外，select 语句如果加锁，也是`当前读`

######## 共享锁：select xx lock in share mode;

######## 排它锁：select xx for update;

##### 隔离级别

###### Read Uncommitted

###### Read Committed

####### 查询只承认在`语句`启动前就已经提交完成的数据

####### 解决：脏读

###### Repeatable Read

####### 查询只承认在`事务`启动前就已经提交完成的数据

####### 解决：脏读、不可重复读

###### Serialized

####### 对相关记录加读写锁

####### 解决：脏读、不可重复读、幻读

### 运维

#### 逻辑架构

##### Server层

###### 连接器

管理连接，权限验证


####### 推荐用长连接

######## 问题：长期积累可能OOM

因为执行过程中临时使用的内存是管理在连接对象里，直到连接断开才释放

######## 优化

######### 定期断开长连接

######### 每次执行比较大的操作后，执行mysql_reset_connection 重新初始化连接

###### 查询缓存

####### 表更新后，缓存即失效

####### 8.0 不再支持查询缓存

###### 分析器

####### 词法分析

####### 语法分析

###### 优化器

####### 生成执行计划

####### 选择索引

####### 选择表连接顺序

###### 执行器

####### 判断权限

####### 操作引擎

##### 存储引擎

###### InnoDB

####### 事务

InnoDB每一条SQL语言都默认封装成事务，自动提交，这样会影响速度，所以最好把多条SQL语言放在begin和commit之间，组成一个事务；



####### 不保存表的具体行数

###### MyISAM

####### 不支持事务

####### 不支持行锁

####### 不支持外键

####### 保存了整个表的行数

####### 支持全文索引

###### Memory

#### 连接池

##### 配置参数

###### maxLive (100)

###### maxIdle (100)

###### minIdle (10)

###### initialSize (10)

###### maxWait (30s)

当第101个请求过来时候，等待30s; 30s后如果还没有空闲链接，则报错

##### 由客户端维护

#### 收缩表空间

##### 空洞

###### 删除数据导致空洞：只是标记记录为可复用，磁盘文件大小不会变

###### 插入数据导致空洞：索引数据页分裂

##### 重建表

###### alter table t engine=innodb, ALGORITHM=copy;

####### 原表不能同时接受更新

####### temp table

###### alter table t engine=innodb, ALGORITHM=inplace; 

####### Online DDL: 同时接受更新

####### temp file

####### row log

###### analyze table t 其实不是重建表，只是对表的索引信息进行重新统计

###### optimize table = alter + analyze

#### 误删恢复

##### 误删行

###### binlog_format=row + Flashback工具

###### 预防

####### sql_safe_updates=on

保证delete语句必须有where, where里包含索引字段


##### 误删库/表

###### Flashback不起作用

truncate /drop table 和 drop database, binlog里记录的是statement，所以无法通过Flashback恢复


###### 全量备份 + 恢复实时binlog

###### 延迟复制备库

CHANGE MASTER TO MASTER_DELAY = N (秒)

##### rm删除数据

###### 重新选主

#### 复制表数据

##### 物理拷贝

- create table r like t;
- alter table r discard tablespace;
- flush table t for export;
- cp t.cfg r.cfg;
  cp t.idb r.idb;
- unlock tables;
- alter table r import tablespaces;

##### mysqldump

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

##### select ... into outfile

select * from db1.t where a>900 into outfile '/server_tmp/t.csv';

load data infile '/server_tmp/t.csv' into table db2.t;

#### 主备

##### 主备延迟

###### 时间点：1. 主库完成一个事务写入binlog，2. 传给备库，3. 备库执行

###### show slave status --> seconds_behind_master

###### 原因

####### 备库机器性能差

####### 备库压力大：运营分析脚本 （用一主多从解决，或者输出到hadoop提供统计类查询的能力）

####### 大事务

######## 一次性用delete语句删除太多数据

######## 大表DDL

####### 备库的并行复制能力

###### 解决方案

####### 强制走主库

####### sleep

####### 判断主备无延迟

######## 判断seconds_behind_master

每次查询从库前，判断seconds_behind_master是否为0

######## 对比位点

show slave status
- Master_Log_File <-> Read_Master_Log_Pos
- Relay_Master_Log_File <-> Exec_Master_Log_Pos

######## 对比GTID

- Retrieved_Gtid_Set 备库收到的所有日志GTID
- Executed_Gtid_Set 备库所有已执行完的GTID
二者相同则表示备库接收到的日志都已经同步完成。

####### 配合 semi-sync

semi-sync
- 事务提交时，主库把binlog发给从库；
- 从库收到binlog后，返回ack;
- 主库收到ack后，才给客户端返回事务完成的确认。

问题：
- 如果有多个从库，则还是可能有过期读。
- 在持续延迟情况下，可能出现过度等待。

####### 等主库位点

- 更新完成后，执行 `show master status`获取当前主库执行到的file/pos.

- 查询从库前， `select master_pos_wait(file, pos, timeout)` 查询从命令开始执行，到应用完file/pos表示的binlog位置，执行了多少事务。

- 如果返回 >=0, 则在此从库查询；否则查询主库

####### 等GTID

- 主库更新后，返回GTID;
- 从库查询 `select wait_for_executed_gtid_set(gtid_set, 1)`
- 返回0则表示从库已执行该GTID.

##### 主备切换

###### 主备切换策略

####### 可靠性优先策略

保证数据一致。
- 判断备库seconds_behind_master, 如果小于阈值则继续；
- 主库改成readonly；
- 判断备库seconds_behind_master，如果为0，则把备库改成可读写 readonly=false.

问题：步骤2之后，主备都是readonly，系统处于不可写状态。

####### 可用性优先策略

保证系统任意时刻都可写。

- 直接把连接切到备库，并让备库可读写。
- 再把主库改成readonly

问题：
若binlog_format=mixed，可能出现数据不一致；
若binlog_format=row，主备同步可能报错，例如duplicate key error.

###### 如何判断主库出问题

####### select 1

######## 问题场景：innodb_thread_concurrency小，新请求都处于等待状态

innodb_thread_concurrency一般设为64~128.
线程等待行锁时，并发线程计数会-1.

####### 查表判断

mysql> select * from mysql.health_check; 

######## 问题场景：空间满了后，查询可以，但无法写入

####### 更新判断

update mysql.health_check set t_modified=now()

######## 问题场景：主备库做相同更新，可能导致行冲突

解决：多行数据 (server_id, now())

####### 内部统计

假定认为IO请求时间查过200ms则认为异常：
```
mysql> select event_name,MAX_TIMER_WAIT  FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;
```

######## 打开统计有性能损耗

#### 安全

##### privilege

###### create user

create user 'user1'@'%' identified by 'pwd';

####### 会插入mysql.user表

###### grant privilege

grant all privileges on *.* to 'ua'@'%' with grant option;

- *.* 全局权限
- db1.* 库权限
- db1.t1 表权限
- grant select(id), .. 列权限

###### revoke privilege

revoke all privileges on *.* from 'ua'@'%';

###### flush privileges

####### 作用: 清空acl_users数组，从mysql.user表重新读取数据

####### grant/revoke后没必要执行flush

#### 分库分表

##### 分区表

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


###### 特点

####### 缺点1：MySQL第一次打开分区表时，需要访问所有分区

####### 缺点2：server层认为是同一张表，所有分区共用同一个 MDL 锁

####### 引擎层认为是不同的表

###### 场景

####### 历史数据，按时间分区

####### alter table drop partition清理历史数据

##### mycat

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

## Spring

### 外部属性文件

#### PropertyPlaceholderConfigurer

##### 覆盖其convertProperty方法可实现加密属性值

#### <context:property-placeholder location=/>

### 国际化

#### Java支持

##### ResourceBundle.getBundle

##### MessageFormat.format

#### MessageResource

```
@Bean
	public MessageSourceAccessor messageSourceAccessor() {
		return new MessageSourceAccessor(messageSource());
	}

	@Bean
	public MessageSource messageSource() {
		ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
		messageSource.setDefaultEncoding("UTF-8");
		messageSource.setBasenames("classpath:vitamin/messages/message");
		return messageSource;
	}
```    

##### MessageSourceAccessor

##### ApplicationContext继承MessageResource

### 容器事件

#### 事件

##### ApplicationContextEvent

容器的启动、刷新、停止、关闭

##### RequestHandledEvent

HTTP请求被处理


#### 监听器

##### ApplicationListener

实现 `onApplicationEvent(E event)`

#### 事件源

#### 监听器注册表

##### ApplicationEventMulticaster

#### 事件广播器

##### ApplicationEventMulticaster

#### 事件发送者

##### 实现ApplicationContextAware

##### ctx.publishEvent()

### Spring Core

#### Bean

##### 生命周期

##### 作用域

###### singleton

###### prototype

每次getBean都会创建一个Bean，如果是cglib动态代理，则性能不佳


###### request

###### session

###### globalSession

###### 作用域依赖问题

prototype --> request, 动态代理

##### FactoryBean: 定制实例化Bean的逻辑

##### 配置方式

###### XML配置

###### Groovy配置

###### 注解配置

####### @Component

####### @Service

###### Java类配置

####### @Configuration

####### @Import

参考 `@EnableWebMvc`

######## 还可以@Import(ImportSelector.class)

更加灵活，可以增加条件分支，参考`@EnableCaching`

####### @Bean

##### 创建流程

###### ResourceLoader: 装载配置文件 --> Resouce

###### BeanDefinitionReader: 解析配置文件 --> BeanDefinition，并保存到BeanDefinitionRegistry

###### BeanFactoryPostProcessor: 对BeanDefinition进行加工

####### 对占位符<bean>进行解析

####### 找出实现PropertyEditor的Bean, 注册到PropertyEditorResistry

###### InstantiationStrategy: 进行Bean实例化

####### SimpleInstantiationStrategy

####### CglibSubclassingInstantiationStrategy

###### BeanWapper: 实例化时封装

####### Bean包裹器

####### 属性访问器：属性填充

####### 属性编辑器注册表

属性编辑器：将外部设置值转换为JVM内部对应类型


###### BeanPostProcessor

#### DI

##### Bean Factory

###### IoC容器

###### 子类

####### DefaultListableBeanFactory

##### ApplicationContext

###### 应用上下文，Spring容器

###### 结合POJO、Configuration Metadata

###### 子类

####### ClassPathXmlApplicationContext

####### FileSystemXmlApplicationContext

####### AnnotationConfigApplicationContext

##### WebApplicationContext

##### 依赖注入

###### 属性注入

###### 构造函数注入

###### 工厂方法注入

###### 注解默认采用byType自动装配策略

##### 条件装配

###### @Profile

###### @Conditional

例： OnPropertyCondition

#### AOP

##### 术语

###### JoinPoint 连接点

AOP黑客攻击的候选锚点
- 方法
- 相对位置


###### Pointcut 切点

定位到某个类的某个方法

###### Advice 增强

- AOP黑客准备的木马
- 以及方位信息: After, Before, Around


###### Target 目标对象

Advice增强逻辑的织入目标类

###### Introduction 引介

为类添加属性和方法，可继承 `DelegatingIntroductionInterceptor`

###### Weaving 织入

将Advice添加到目标类的具体连接点上的过程。
(连接切面与目标对象 创建代理的过程)

###### Aspect 切面

Aspect = Pointcut + Advice？


##### 原理

###### JDK动态代理

###### CGLib动态代理

####### 不要求实现接口

####### 不能代理final 或 private方法

####### 性能比JDK好，但是创建花费时间较长

##### 用法

###### 编程方式

####### ProxyFactory.addAdvice / addAdvisor

ProxyFactory.setTarget
ProxyFactory.addAdvice
ProxyFactory.getProxy() --> Target

```
public void addAdvice(int pos, Advice advice) {
  this.addAdvisor(pos, new DefaultPointcutAdvisor(advice));
}
```

####### 配置ProxyFactoryBean

<bean class="aop.ProxyFactoryBean"
p:target-ref="target"
p:interceptorNames="advice or adviso">
  
  

####### 自动创建代理

基于BeanPostProcessor实现，在容器实例化Bean时 自动为匹配的Bean生成代理实例。


######## BeanNameAutoProxyCreator

基于Bean配置名规则

######## DefaultAdvisorAutoProxyCreator

基于Advisor匹配机制

######## AnnotationAwareAspectJAutoPRoxyCreator

###### AspectJ

####### <aop:aspectj-autoproxy>

- 自动为匹配`@AspectJ`切面的Bean创建代理，完成切面织入。
- 底层通过 `AnnotationAwareAspectJAutoProxyCreator`实现。

### Spring MVC

#### 流程

##### web.xml

###### web.xml 匹配DispatcherServlet映射路径

##### DispatcherServlet

###### request.setAttribute

####### localResolver

####### themeResolver

##### HandlerMapping

###### 通过HandlerMapping找到处理请求的Handler

##### HanlderAdapter

###### 通过HandlerAdapter封装调用Handler

###### HttpMessageConverter

####### MappingJackson2HttpMessageConverter

####### 例如构造RequestBody / ResponseBody

##### ViewResolver

###### 通过ViewResolver解析视图名到真实视图对象

###### 接口

####### AbstractCachingViewResolver

####### UrlBasedViewResolver

####### FreeMarkerViewResolver

####### ContentNegotiatingViewResolver

####### InternalResourceViewResolver

###### 源码

####### DispatcherServlet

######## initStrategies()

######### initViewResolvers()

######### 初始化所有 ViewResolver.class 

######## doDispatch()

######### applyDefaultViewName()

######### processDispatchResult()

########## 异常处理

########## render

#### 自动装配

##### Spring SPI

###### 基础接口：WebApplicationInitializer

###### 编程驱动：AbstractDispatcherServletInitializer

###### 注解驱动：AbstractAnnotationConfigDispatcherServletInitializer

##### 流程

###### 1. 启动时查找ServletContainerInitializer实现类 

###### 2. 找到SpringServletContainerInitializer

###### 3.@HandlesTypes({WebApplicationInitializer.class})

#### 编码

##### 入参

###### 入参种类

####### ModelMap / Map

SpringMVC在调用方法前会创建一个隐含的模型对象。如果方法入参为Map/Model，则会将隐含模型的引用传递给这些入参。


####### WebRequest

####### HttpServletRequest / HttpSession

####### @MatrixVariable

####### @CookieValue

####### @RequestHeader

####### @RequestParam

####### @RequestParam MultipartFile

######## MultipartResolver

######## 支持类型 multipart/form-data

######### consumes = MediaType.MULTIPART_FORM_DATA_VALUE

###### 入参原理

####### HttpMessageConverter

- MappingJackson2HttpMessageConverter
- ByteArrayHttpMessageConverter

####### Converter

###### 校验

####### POJO配置

######## @NotEmpty

######## @NotNull

####### 入参配置

######## @Valid

######### 校验失败返回400 + errors

######## BindingResult

######### 调用bindingResult.hasErrors() 对校验失败情况进行处理

##### 返回值

###### 缩进设置

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer c {
  return builder -> builder.indentOutput(true);
}
```

##### 异常处理

###### @ExceptionHandler

####### 添加到@Controller中

####### 或添加到@ControllerAdvice中

###### 核心接口 HandlerExceptionResolver

####### SimpleMappingExceptionResovler

####### DefaultHandlerER

####### ResponseStatusER

####### ExceptionHandlerER

##### 拦截器

###### 接口

####### 同步：HandlerInterceptor

####### 异步：AsyncHandlerInterceptor  -afterConcurrentHandlingStarted

###### 源码：DispatcherSevlet

```java
doDispatch() {
 if (!applyPreHandle())
    return;
    
 handleAdapter.handle();
 
 if (async)
	return;
 
 applyPostHandle();
 
 
 triggerAfterCompletion();
 
 finally {
	if (asyn) 
       applyAfterConcurrentHandlingStarted();
				
}
```

###### 配置方式

####### WebMvcConfigurer.addInterceptors()

####### SpringBoot: @Configuration WebMvcConfigurer

#### 原理

##### WebApplicationContext

###### Servlet WebApplicationContext

WEB相关bean，继承自RootXxx

####### Controllers

####### ViewResolver

####### HandlerMapping

###### Root WebApplicationContext

middle-tier serv

####### Services

####### Repositories

###### AOP

####### 父子context的aop是独立的

####### 要想同时拦截父子：父Apspect @Bean, 子 <aop:aspectj-autoproxy/>

### Spring Data

#### spring-data-jpa

##### 连接池

###### Hikari

###### c3p0

###### alibaba druid

https://github.com/alibaba/druid

- 通过Filter, 支持自定义pei

##### 事务

###### 传播

####### PROPAGATION_REQUIRED

######## 当前有就用当前的，没有就新建

####### PROPAGATION_SUPPORTS

######## 当前有就用当前的，没有就算了

####### PROPAGATION_MANDATORY

######## 当前有就用当前的，没有就抛异常

####### PROPAGATION_REQUIRES_NEW

######## 无论有没有，都新建

####### PROPAGATION_NOT_SUPPORTED

######## 无论有没有，都不用

####### PROPAGATION_NEVER

######## 如果有，抛异常

####### PROPAGATION_NESTED

######## 如果有，则在当前事务里再起一个事务

###### 隔离

####### ISOLATION_READ_UNCOMMITTED

- 脏读
- 不可重复读
- 幻读

####### ISOLATION_READ_COMMITTED

- 不可重复读
- 幻读

####### ISOLATION_REPEATABLE_READ

- 幻读

####### ISOLATION_SERIALIZABLE

- 

##### 注解

###### 实体

####### @Entity

####### @MappedSuperclass

######## 标注于父类

####### @Table

######## 表名

###### 主键

####### @Id

######## @GeneratedValue

######## @SequenceGenerator

###### 映射

####### @Column

####### @JoinTable  @JoinColumn

##### reactive

###### @EnableR2dbcRepositories

###### ReactiveCrudRepository

####### 返回值都是Mono或Flux

####### 自定义的查询需要自己写@Query

#### spring-data-mongodb

##### 注解

###### @Document

###### @Id

##### mongoTemplate

###### save / remove

###### Criteria / Query / Update

###### find(Query) / findById()

###### updateFirst(query, new Update()...)

##### MongoRepository

###### @EnableMongoRepositories

#### spring-data-redis

##### Jedis

###### JedisPool

####### 基于Apache Common Pool

####### JedisPoolConfig

####### JedisPool.getResource

######## internalPool.borrowObject()

###### JedisSentinelPool

####### 监控、通知、自动故障转移、服务发现

####### MasterListener

###### JedisCluster

####### 配置

######## JedisSlotBasedConnectionHandler

######### getConnection: 随机

######### getConnectionFromSlot: 基于slot选择

######## JedisClusterInfoCache

####### 技巧

######## 单例：内置了所有节点的连接池

######## 无需手动借还连接池

######## 合理设置commons-pool

####### 不支持读slave

##### Lettuce

###### 支持读写分离

####### 只读主、只读从

####### 优先读主、优先读从

####### LettuceClientConfigurationBuilderCustomizer -> readFrom(ReadFrom.MASTER_PREFERRED)

##### RedisTemplate

##### Repository

###### @EnableRedisRepository

###### @RedisHash

####### Class级别：@RedisHash(value = "springbucks-coffee", timeToLive = 60)

###### @Id

###### @Index

####### 二级索引，自动创建另一套key-value

##### Converter

###### @WritingConverter

###### @ReadingConverter

###### byte[]与对象互转

#### reactor

```java
Flux.fromIterable(list)
  .publishOn(Schedulers.single())
  .doOnComplete(..)
  .flatMap(..)
  .concatWith()
  .onErrorResume(..)
  .subscribe()..;
```

##### Operators

###### publisher / subscriber

###### onNext() / onComplete() / onError()

###### Flux[0..N] / Mono[0..1]

##### Backpressure

###### Subscription

###### onRequest() / onCancel() / onDispose()

##### Schedulers

###### immediate() / single() / newSingle()

###### elastic() / parallel() / newParallel()

##### 错误处理

###### onError / onErrorReturn / onErrorResume

###### doOnError / doFinally

##### 配置

###### ReactiveRedisConnection / ReactiveRedisTemplate

###### ReactiveMongoDatabaseFactory / ReactiveMongoTemplate

```java
mongoTemplate.insertAll(list)
  .publishOn(Schedulers.elastic())
  .doOnNext(log..)
  .doOnComplete(..)
  .doOnFinally(..)
  .count()
  .subscribe();
```

### Spring Boot

#### 模式注解

##### 派生性

##### 层次性

#### 自动装配

##### 1.@EnableAutoConfiguration

##### 2. XXAutoConfiguration

###### 条件判断 @Conditional

###### 模式注解 @Configuration

###### @Enable模块：@EnableXX -> *ImportSelector -> *Configuration

##### 3.配置spring.factories (SpringFactoriesLoader)

#### 源码

##### SpringApplication

###### 准备阶段

####### 配置 Spring Boot Bean 源		

####### 推断Web应用类型

根据classpath

####### 推断引导类

根据 Main 线程执行堆栈判断实际的引导类

####### 加载ApplicationContextInitializer

spring.factorie

####### 加载ApplicationListener

spring.factories
例如`ConfigFileApplicationListener`

###### 运行阶段

####### 加载监听器 SpringApplicationRunListeners

spring.factories
getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args))

`EventPublishingRunListener` 
--> `SimpleApplicationEventMulticaster`

######## EventPublishingRunListener

######## SimpleApplicationEventMulticaster

####### 运行监听器 SpringApplicationRunListeners

listeners.starting();

####### 创建应用上下文 ConfigurableApplicationContext

createApplicationContext()
- NONE: `AnnotationConfigApplicationContext`
- SERVLET: `AnnotationConfigServletWebServerApplicationContext`
- REACTIVE: `AnnotationConfigReactiveWebServerApplicationContext` 

####### 创建Environment

getOrCreateEnvironment()
- SERVLET: `StandardServletEnvironment`
- NONE, REACTIVE: `StandardEnvironment` 

#### 运维

##### Actuator

###### 解禁Endpoints

####### management.endpoints.web.exposure.include=*

####### 生产环境谨慎使用

### Spring Cache

#### 标注

##### @EnableCaching

##### @Cacheable

##### @CacheEvict

##### @CachePut

##### @Caching

##### @CacheConfig(cacheNames = "")

### STOMP

## 算法

### 含义

#### 数据结构

##### 一组数据的存储结构

##### 数据结构为算法服务

#### 算法

##### 操作数据的一组方法

##### 算法要作用在特定的数据结构之上

### 复杂度分析

#### 时间复杂度

##### 渐进时间复杂度，表示代码执行时间随数据规模增长的变化趋势

##### 分析

###### 只关注循环执行次数最多的一段代码

###### 加法法则：总复杂度 = 量级最大的那段代码的复杂度

###### 乘法法则：嵌套代码的复杂度 = 嵌套内外代码复杂度的乘积

##### 常见复杂度量级

###### 常量阶 O(1)

####### 当不存在递归、循环

###### 对数阶 O(logn)

####### 循环变量按倍数增长

###### 线性阶 O(n)

###### 线性对数阶 O(nlogn)

###### 平方阶 O(n^2)

###### 指数阶 O(2^n)

###### 阶乘阶 O(n!)

##### 场景

###### 最好

###### 最坏

###### 平均

####### 加权平均时间复杂度、期望时间复杂度

###### 均摊

####### 摊还分析法

对一个数据结构进行一组连续操作中，大部分情况下时间复杂度都很低，只有个别情况下时间复杂度比较高，而且这些操作之间存在前后连贯的时序关系，这个时候，我们就可以将这一组操作放在一块儿分析，看是否能将较高时间复杂度那次操作的耗时，平摊到其他那些时间复杂度比较低的操作上。而且，在能够应用均摊时间复杂度分析的场合，一般均摊时间复杂度就等于最好情况时间复杂度

#### 空间复杂度

##### 渐进空间复杂度，表示存储空间与数据规模之间的增长关系

### 数据结构

#### 线性表

##### 数组

###### 插入低效

####### 优化：插入索引K, 则只将原K元素移到数组尾部

###### 删除低效

####### 优化：标记清除

###### vs. ArrayList

####### 封装细节

####### 动态扩容

###### 内存空间连续

####### 所以申请时如果连续空间不足，会OOM

##### 链表

###### 种类

####### 单链表

####### 双向链表

######## LinkedHashMap

######## 用空间换时间

####### 循环链表

####### 双向循环链表

####### 静态链表

###### 例子

####### LRU 缓存

使用有序单链表，尾部表示最早使用的节点。插入节点时，
- 遍历链表，查询是否已存在；
- 若存在，则从原位置删除、插入到头；
- 若不存在，且缓存未满，则插入到头；
- 若不存在，且缓存已满，则删除尾部节点，插入到头。


####### 判断字符串回文

####### 单链表反转

https://leetcode.com/problems/reverse-linked-list/ 

####### 链表中环的检测

https://leetcode.com/problems/linked-list-cycle/
https://leetcode.com/problems/linked-list-cycle-ii/

####### 有序链表合并

https://leetcode.com/problems/merge-two-sorted-lists/


####### 删除链表倒数第n个节点

https://leetcode.com/problems/remove-nth-node-from-end-of-list/


####### 求链表中间节点

https://leetcode.com/problems/middle-of-the-linked-list/

###### 技巧

####### 理解指针或引用的含义

####### 警惕指针丢失

####### 利用哨兵简化实现难度

######## 虚拟空头

####### 留意边界条件处理

######## 空、1节点、2节点

####### 举例、画图

##### 栈

###### 顺序栈

###### 链式栈

##### 队列

###### 普通队列

###### 双端队列

###### 阻塞队列

###### 并发队列

###### 阻塞并发队列

#### 散列表

##### 散列函数

##### 冲突解决

###### 链表法

###### 开放寻址

###### 其他

##### 动态扩容

##### 位图

#### 树

##### 二叉树

###### 平衡二叉树

###### 二叉查找树

###### 平衡二叉查找树

####### AVL 树

####### 红黑树

###### 完全二叉树

###### 满二叉树

##### 多路查找树

###### B 树

###### B+ 树

###### 2-3 树

###### 2-3-4 树

##### 堆

###### 小顶堆

###### 大顶堆

###### 优先级队列

###### 斐波那契堆

###### 二项堆

##### 其他

###### 树状数组

###### 线段树

#### 图

##### 图的存储

###### 邻接矩阵

###### 邻接表

##### 拓扑排序

##### 最短路径

##### 关键路径

##### 最小生成树

##### 二分图

##### 最大流

### 常见思路

#### 递归

##### 严格定义递归函数作用：参数、返回值、side-effect

##### 先一般，后特殊

##### 每次调用必须缩小问题规模

##### 每次问题规模缩小程度必须为1

#### 循环

##### 定义循环不变式，循环体每次结束后保持循环不变式

##### 先一般，后特殊

##### 每次必须向前推进循环不变式中涉及的变量值

##### 每次推进的规模必须为1

#### 二分

##### [a, b) 前闭后开

###### [a, b) + [b, c) = [a, c)

###### b - a = len([a, b))

###### [a, a) ==> empty range

### 基本算法思想

#### 贪心算法

#### 分治算法

#### 动态规划

#### 回溯算法

#### 枚举算法

### 常见算法场景

#### 排序

##### 二分查找

###### O(logN)

##### O(n^2)

###### 冒泡排序

在要排序的一组数中，对当前还未排好序的范围内的全部数，自上而下对相邻的两个数依次进行比较和调整，让较大的数往下沉，较小的往上冒。

即：每当两相邻的数比较后发现它们的排序与排序要求相反时，就将它们互换。

###### 插入排序

在要排序的一组数中，假设前面(n-1) [n>=2] 个数已经是排好顺序的。现在要把第n个数插到前面的有序数中，使得这n个数也是排好顺序的。

###### 选择排序

- 在要排序的一组数中，选出最小的一个数与第一个位置的数交换；
- 然后在剩下的数当中再找最小的与第二个位置的数交换，
- 如此循环到倒数第二个数和最后一个数比较为止。


###### 希尔排序

算法先将要排序的一组数按某个增量d（n/2,n为要排序数的个数）分成若干组，每组中记录的下标相差d.对每组中全部元素进行`直接插入排序`，
然后再用一个较小的增量（d/2）对它进行分组，在每组中再进行直接插入排序。
当增量减到1时，进行直接插入排序后，排序完成。

##### O(nlogn)

###### 归并排序

归并（Merge）排序法是将两个有序表合并成一个新的有序表。即：

- 把待排序序列分为若干个子序列，每个子序列是有序的。
- 然后再把有序子序列合并为整体有序序列。

###### 快速排序

- 选择一个基准元素，通常选择第一个元素或者最后一个元素。
- 通过一趟扫描，将待排序列分成两部分：一部分比基准元素小；一部分大于等于基准元素。
- 
此时基准元素在其排好序后的正确位置，然后再用同样的方法递归地排序划分的两部分。

###### 堆排序

堆排序需要两个过程，一是建立堆，二是堆顶与堆的最后一个元素交换位置。

所以堆排序有两个函数组成。
- 一是建堆的渗透函数，
- 二是反 复调用渗透函数实现排序的函数。

##### O(n)

###### 计数排序

###### 基数排序

- 将所有待比较数值（正整数）统一为同样的数位长度，数位较短的数前面补零。
- 然后，从最低位开始，依次进行一次排序。
- 这样从最低位排序一直到最高位排序完成以后，数列就变成一个有序序列。


###### 桶排序

#### 搜索

##### 深度优先搜索

##### 广度优先搜索

##### A*启发式搜索

#### 查找 

##### 线性表查找

##### 树结构查找

##### 散列表查找

#### 字符串匹配

##### 朴素

##### KMP

##### Robin-Karp

##### Boyer-Moore

##### AC 自动机

##### Trie

##### 后缀数组

#### 其他

##### 数论

##### 计算几何

##### 概率分析

##### 并查集

##### 拓扑网络

##### 矩阵运算

##### 线性规划
