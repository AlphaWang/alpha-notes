# Java架构知识图谱

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

##### 构造参数

##### 线程池大小设置

###### CPU密集型任务：线程池尽可能小

####### CPU核数 + 1 

###### IO密集型任务：线程池尽可能大

####### CPU核数 * (1 + IO耗时/CPU耗时 )

#### Fork / Join

#### Future

#### 模式

##### Guarded Suspension模式

##### Balking模式

##### Thread-Per-Message模式

##### 生产者消费者模式

##### Worker Thread模式

##### 两阶段终止模式

### 协作

#### Semaphore

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

#### CyclicBarrier

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

##### 线程封闭

###### 不和其他线程共享变量

##### 线程本地存储

###### ThreadLocal

##### CAS

##### Copy-on-Write

##### Atomic原子类

###### 类型

####### 基本类型：AtomicInteger, AtomicLong

####### 数组类型：AtomicIntegerArray, Atomic ReferenceArray

AtomicIntegerArray, AtomicLongArray, Atomic ReferenceArray

####### 引用类型：AtomicReference, AtomicStampedReference

####### 对象属性修改：AtomicIntegerFieldUpdater, AtomicStampedReference

###### 原理

####### CAS + volatile

- 利用 `CAS` (compare and swap) + `volatile` 和 native 方法来保证原子操作，从而避免 `synchronized` 的高开销，执行效率大为提升。

- CAS的原理是拿期望的值和原本的一个值作比较，如果相同则更新成新的值。`UnSafe.objectFieldOffset()` 方法是一个本地方法，这个方法是用来拿到“原来的值”的内存地址，返回值是 valueOffset。另外 value 是一个volatile变量，在内存中可见，因此 JVM 可以保证任何时刻任何线程总能拿到该变量的最新值。

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

###### condition

####### 必须在排它锁中使用

###### 最佳实践

####### 永远只在更新对象的成员变量时加锁

####### 永远只在访问可变的成员变量时加锁

####### 永远不在调用其他对象的方法时加锁

##### 读写锁

## 分布式

### ZooKeeper

#### 特性

##### 顺序一致性

对同一个客户端来说

##### 原子性

所有事务请求的处理结果在集群中所有机器上的应用情况是一致的。


##### 单一视图

客户端无论连接哪个zk服务器，看到的数据模型都是一致的


##### 可靠性

##### 实时性

#### CP (ZAB协议保证一致性)

Zookeeper Atomic Broadcast.
- 所有事务请求必须由全局唯一的Leader服务器来协调处理。
- Leader将客户端的事务请求转换成一个`事务Proposal`，将其分发给所有Follower，并等待Follower的反馈.
- 一旦超过半数的Follower进行了正确的反馈，则Leader再次向所有Follower分发`commit消息`，要求其将前一个Proposal进行提交


##### 单一主进程

- 使用单一的主进程来接收并处理所有事务请求（事务==写），
- 并采用ZAB原子广播协议，将服务器数据的状态变更以事务Proposal的形式广播到所有副本进程。

###### 单一的主进程来接收并处理所有事务请求

###### 对每个事务分配全局唯一的ZXID

####### zxid低32位：单调递增计数器

对每个事务请求，生成Proposal时，将计数器+1


####### zxid高32位：Leader周期的epoch编号

###### 数据的状态变更以事务Proposal的形式广播到所有副本进程

##### 顺序应用

必须能保证一个全局的变更序列被顺序应用，从而能处理依赖关系

##### 1. (发现)崩溃恢复模式：选举要保证新选出的Leader拥有最大的ZXID

崩溃恢复：Leader服务器出现宕机，或者因为网络原因导致Leader服务器失去了与过半 Follower的联系，那么就会进入崩溃恢复模式从而选举新的Leader。
当选举产生了新的Leader，同时集群中有过半的服务器与该Leader完成了状态同步（即数据同步）之后，Zab协议就会退出崩溃恢复模式，进入消息广播模式。

######  

则可保证新Leader一定具有所有已提交的Proposal；
同时保证丢弃已经被跳过的Proposal

##### 2. (同步) 检查是否完成数据同步

###### 对需要提交的，重新放入Proposal+Commit

###### 对于Follower上尚未提交的Proposal，回退

###### 同步阶段的引入，能有效保证Leader在新的周期中提出Proposal之前，
所有的进程都已经完成了对之前所有Proposal的提交。

##### 3. (广播) 消息广播模式：Proposal (ZXID), ACK, Commit

消息广播：所有的事务请求都会转发给Leader，Leader会为事务生成对应的Proposal，并为其分配一个全局单调递增的唯一ZXID。
当Leader接受到半数以上的Follower发送的ACK投票，它将发送Commit给所有Follower通知其对事务进行提交，Leader本身也会提交事务，并返回给处理成功给对应的客户端。

###### 1.Leader为事务生成对应的Proposal，分配ZXID

必须将每一个事务Proposal按照其ZXID的先后顺序来进行排序与处理。

###### 2.半数以上的Follower回复ACK投票

###### 3.发送Commit给所有Follower通知其对事务进行提交

###### 4.返回给处理成功给对应的客户端

###### 类似一个2PC提交，移除了中断逻辑

#### 原理

##### version保证原子性

###### version表示修改次数

###### 乐观锁

##### watcher

###### 特性

####### 一次性

####### 客户端串行执行

####### 轻量

######## 推拉结合

######## 注册watcher时只传输ServerCnxn

###### 流程

####### 客户端注册Watcher

####### 客户端将Watcher对象存到WatchManager: Map<String, Set<Watcher>>

####### 服务端存储ServerCnxn

######## watchTable: Map<String, Set<Watcher>>

######## watch2Paths: Map<Watch, Set<String>>

####### 服务端触发通知

######## 1.封装WatchedEvent

######## 2.查询并删除Watcher

######## 3.process: send response (header = -1)

####### 客户端执行回调

######## 1.SendThread接收通知， 放入EventThread

NIO

######## 2.查询并删除Watcher

######## 3.process: 执行回调

###### WatchedEvent

网络传输时序列化为 `WatcherEvent`

####### KeeperState

- SyncConnected
- Disconnected
- Expired
- AuthFailed


####### EventType

- None
- NodeCreated
- NodeDeleted
- NodeDataChanged
- NodeChildrenChanged

###### Curator 如何解决一次性watcher问题？

##### ACL

###### Scheme

####### IP:192.168.1.1:permission

####### Digest:username:sha:permission

####### World:anyone:permission

####### Super:username:sha:permission

###### Permission

####### C, Create

####### D, Delete

####### R, Read

####### W, Write

####### A, Admin

###### 权限扩展体系

####### 实现AuthenticationProvider

####### 注册

######## 系统属性 -Dzookeeper.authProvider.x=

######## zoo.cfg: authProvider.x=

##### 客户端

###### 通讯协议

####### 请求

######## RequestHeader

######### xid

记录客户端发起请求的先后顺序

######### type

- 1: OpCode.Create
- 2: delete
- 4: getData


######## Request

####### 响应

######## ReplyHeader

######### xid

原值返回


######### zxid

服务器上的最新事务ID


######### err

######## Response

###### ClientCnxn：网络IO

####### outgoingQueue

待发送的请求Packet队列

####### pendingQueue

已发送的、等待服务端响应的Packetdui'lie

####### SendThread: IO线程

####### EventThread: 事件线程

######## waitingEvents队列

##### Session

###### SessionID: 服务器myid + 时间戳

###### SessionTracker: 服务器的会话管理器

####### 内存数据结构

######## sessionById:     HashMap<Long, SessionImpl>

######## sessionWithTimeout: ConcurrentHashMap<Long, Integer>

######## sessionSets:     HashMap<Long, SessionSet>超时时间分桶

####### 分桶策略

- 将类似的会话放在同一区块进行管理。
- 按照“下次超时时间”
- 好处：清理时可批量处理

####### 会话激活

- 心跳检测
- 重新计算下一次超时时间
- 迁移到新区块


######## 客户端发送任何请求时

######## sessionTimeout / 3时，发送PING请求

####### 超时检测：独立线程，逐个清理

####### 会话清理

######## 1. isClosing设为true

######## 2.发起“会话关闭”请求

######## 3.收集需要清理的临时节点

######## 4.添加“节点删除”事务变更

######## 5.删除临时节点

######## 6.移除会话、关闭NIOServerCnxn

####### 重连

######## 连接断开

- 断开后，客户端收到None-Disconnected通知，并抛出异常`ConnectionLossException`；
- 应用需要捕获异常，并等待客户端自动完成重连；
- 客户端自动重连后，收到None-SyncConnected通知

######## 会话失效

- 自动重连时 超过了会话超时时间。
- 应用需要重新实例化ZooKeeper对象，重新恢复临时数据

######## 会话转移

- 服务端收到请求时，检查Owner 如果不是当前服务器则抛出`SessionMovedExceptio`

#### 角色

##### Leader: 读写

##### Follower: 读。参与Leader选举、过半写成功

##### Observer: 读

#### 应用

##### 配置中心：数据发布订阅

推拉结合的方式：
- 推：节点数据发生变化后，发送Watcher事件通知给客户端。
- 拉：客户端收到通知后，需要主动到服务端获取最新的数据


##### 负载均衡：域名注册、发现

##### 命名服务：全局ID生成器（顺序节点）

##### 分布式协调、通知：任务注册、任务状态记录

##### 集群管理：分布式日志收集系统、云主机管理

日志收集系统要解决：
1. 变化的日志源机器
2. 变化的收集器机器

- 注册日志收集器，非临时节点：`/logs/collectors/[host]`
- 节点值为日志源机器。
- 创建子节点，记录状态信息 `/logs/collectors/[host]/status`
- 系统监听collectors节点，当有新收集器加入，或有收集器停止汇报，则要将之前分配给该收集器的任务进行转移。

##### Master选举：

利用zk强一致性，保证客户端无法重复创建已存在的节点

##### 分布式锁

###### 排他锁

排他锁，X锁（Exclusive Locks），写锁，独占锁：
- 同时只允许一个事务，其他任何事务不能进行任何类型的操作。
- 创建临时节点


###### 共享锁

共享锁，S锁（Shared Locks），读锁：
- 加共享锁后当前事务只能进行读操作；其他事务也只能加共享锁。
- `W`操作必须在当前没有任务事务进行读写操作时才能进行。
- 创建临时顺序节点 `/s_locks/[hostname]-请求类型-序号`。
- 节点上表明是`R`还是`W`。
- 如果是`R`，且比自己序号小的节点都是`R`，则加锁成功；
- 如果是`W`，如果自己是最小节点，则加锁成功。

优化：
R节点只需要监听比他小的最后一个W节点；
W节点只需要监听比他小的最后一个节点。

##### 分布式队列

###### FIFO

- 注册临时顺序节点；
- 监听比自己小的最后一个节点。

###### Barrier

- 父节点`/queue_barrier`，值为需要等待的节点数目N。
- 监听其子节点数目；
- 统计子节点数目，如果数目小于N，则等待

##### 实例

###### Hadoop

####### ResourceManager HA

多个ResourceManager并存，但只有一个处于Active状态。
- 有父节点 `yarn-leader-election/pseudo-yarn-rm-cluster`, RM启动时会去竞争Lock**临时**子节点。
- 只有一个RM能竞争到，其他RM注册Wather


####### ResourceManager 状态存储

Active状态的RM会在初始化阶段读取 `/rmstore` 上的状态信息，并据此信息继续进行相应的chu'li

###### HBase

####### RegionServer系统容错

####### RootRegion管理

###### Kafka

####### Broker注册: /broker/ids/[brokerId]

`/broker/ids/[BrodkerId]` (临时节点)

####### Topic注册: /brokers/topics/[topic]

每个topic对应一个节点`/brokers/topics/[topic]`；
Broker启动后，会到对应Topic节点下注册自己的ID **临时节点**，并写入Topic的分区总数，`/brokers/topics/[topic]/[BrokerId] --> 2`


####### Producer负载均衡

Producer会监听`Broker的新增与减少`、`Topic的新增与减少`、`Broker与Topic关联关系的变化`

####### Consumer负载均衡:  /consumers/[groupId]/owners/[topic]/[brokerId-partitionId]

当消费者确定了对一个分区的消费权利，则将其ConsumerId写入到分区**临时节点**上：
`/consumers/[groupId]/owners/[topic]/[brokerId-partitionId] --> consumerId`

####### Consumer注册: /consumers/[groupId]/ids/[consumerId]

- 消费者启动后，注册**临时节点** `/consumers/[groupId]/ids/[consumerI]`。并将自己订阅的Topic信息写入该节点 
- 每个消费者都会监听ids子节点。
- 每个消费者都会监听Broker节点：`/brokers/ids/`

####### 消费进度offset记录: /consumers/[groupId]/offsets/[topic]/[brokerId-partitionId]

消费者重启或是其他消费者重新接管消息分区的消息消费后，能够从之前的进度开始继续进行消费：`/consumers/[groupId]/offsets/[topic]/[brokerId-partitionId] --> offset`

### Kafka

#### 集群

##### 集群成员：对等，没有中心主节点

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

###### count(字段) < count(id) < count(1) = count(*)

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

## NoSql

### Memcached

#### memcached是lazy clean up, 那么如何保证内存不被占满？

### Redis

#### 数据类型

##### string

set name codehole
set age 30
incr age
incrby age 5

###### SDS, Simple Dynamic String

####### capacity

####### len

####### content：以字节\0结尾

####### 扩容：< 1M 之前，加倍扩容； > 1M后，每次扩容1M 

###### 内部编码

####### raw

####### int

####### embstr

###### 常用命令

####### set k v

####### setnx k v: 不存在才设置(add)

####### set k v xx: key存在 才设置(update)

####### setex k 60 v: 生存时间

####### getset: 返回旧值

####### append: 

####### strlen:

####### incr / decr / incrby / decrby

####### mget / mset

####### del

##### hash

hset books java "think in java"


###### dict -> dictht[2] -> disctEntry

####### 扩容时机：元素的个数等于第一维数组的长度; 
bgsave时不扩容，除非达到dict_force_resize_ratio

####### 扩容：扩容 2 倍

####### 缩容：元素个数低于数组长度10%时

####### ziplist: 元素个数较小时，用ziplist节约空间

压缩列表是一块连续的内存空间，元素之间紧挨着存储，没有任何冗余空隙。

####### 渐进式rehash

它会同时保留旧数组和新数组，然后在定时任务中以及后续对 hash 的指令操作中渐渐地将旧数组中挂接的元素迁移到新数组上。这意味着要操作处于 rehash 中的字典，需要同时访问新旧两个数组结构。如果在旧数组下面找不到元素，还需要去新数组下面去寻找。

###### 内部编码

####### hashtable

####### ziplist

###### 常用命令

####### hget / hset name k v

####### hsetnx name k v

####### hdel

####### hincrby / hincrbyfloat

####### hgetall name

####### hvals name: 返回所有value

####### hkeys name: 返回所有key

##### list

rpush books python java golang

###### 内部编码

####### 后期：quicklist

`quicklist` 是 `ziplist` 和 `linkedlist` 的混合体，
- 它将 `linkedlist` 按段切分，
- 每一段使用 `ziplist` 来紧凑存储，多个 `ziplist` 之间使用双向指针串接起来。



####### 早期：元素少时用 ziplist，元素多时用 linkedlist

###### 常用命令

####### lpush / rpop

######## lpush list-name c b a

####### rpush / lpop

####### linsert key before|after v new-v

####### lset key index v

####### llen

####### lrem key count v 

######## count = 0: 删除所有v

######## count > 0: 从左到右删除count个

######## count < 0: 从右到左删除count个

####### ltrim key start end: 修剪

####### lrange key start end: 范围

####### blop / brpop key timeout: 阻塞pop

###### 应用

####### lpush + lpop --> stack

####### lpush + rpop --> queue

####### lpush + ltrim --> capped collection

####### lpush + brpop --> MQ

##### set

 sadd books python
 

###### 内部编码

####### hashtable

####### IntSet: 当元素都是整数并且个数较小时，使用 intset 来存储

intset 是紧凑的数组结构，同时支持 16 位、32 位和 64 位整数。

###### 常用命令

####### sadd key e

####### srem key e

####### scard: 集合大小

####### sismember: 判断存在

####### srandmember: 随机取元素

####### smembers: 所有元素，慎用

####### sdiff / sinter / sunion

sdiff / sinter / sunion + store destkey
将集合操作的结果存入destkey


###### 应用

####### sadd --> tagging

####### spop / srandmember --> random item

####### sadd + sinter --> social graph

##### zset

zadd books 9.0 "think in java"

它类似于 Java 的 SortedSet 和 HashMap 的结合体，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以给每个 value 赋予一个 score，代表这个 value 的排序权重。它的内部实现用的是一种叫做「跳跃列表」的数据结构。

###### hash: value -> score

hash 结构来存储 value 和 score 的对应关系

###### 内部编码

####### skiplist

######## 二分查找

skiplist提供指定 score 的范围来获取 value 列表的功能，二分查找

####### ziplist

######## 元素个数较小时，用ziplist节约空间

###### 常用命令

####### zadd key score e

####### zscore key e

####### zrem key e

######## zremrangebyrank key start end

######## zremrangebyscore key min-score max-score

####### zincrby key score e

####### zcard

####### zrange key start end [withscores]

######## zrevrange

######## zrangbyscore key min-score max-score

####### zcount key min-score max-score

####### zinterstore / zunionstore

#### 原理

##### AP: 最终一致性

- Redis 的主从数据是 **异步同步** 的，所以分布式的 Redis 系统并不满足「一致性」要求。
- Redis 保证「最终一致性」，从节点会努力追赶主节点，最终从节点的状态会和主节点的状态将保持一致。

##### 通讯协议：RESP, Redis Serialization Protocal

单行字符串 以 + 符号开头。
多行字符串 以 $ 符号开头，后跟字符串长度。
整数值 以 : 符号开头，后跟整数的字符串形式。
错误消息 以 - 符号开头。
数组 以 * 号开头，后跟数组的长度。

##### 多路复用

###### 指令队列

Redis 会将每个客户端套接字都关联一个指令队列。客户端的指令通过队列来排队进行顺序处理，先到先服务。

###### 响应队列

Redis 服务器通过响应队列来将指令的返回结果回复给客户端。 
- 如果队列为空，那么意味着连接暂时处于空闲状态，不需要去获取写事件，也就是可以将当前的客户端描述符从write_fds里面移出来。等到队列有数据了，再将描述符放进去。避免select系统调用立即返回写事件，结果发现没什么数据可以写。出这种情况的线程会飙高 CPU。

###### epoll事件轮询API

最简单的事件轮询 API 是select函数，它是操作系统提供给用户程序的 API。
输入是读写描述符列表read_fds & write_fds，输出是与之对应的可读可写事件。

同时还提供了一个timeout参数，如果没有任何事件到来，那么就最多等待timeout时间，线程处于阻塞状态。

一旦期间有任何事件到来，就可以立即返回。时间过了之后还是没有任何事件到来，也会立即返回。拿到事件后，线程就可以继续挨个处理相应的事件。处理完了继续过来轮询。于是线程就进入了一个死循环，我们把这个死循环称为事件循环，一个循环为一个周期。

##### pipeline

客户端通过改变了读写的顺序带来的性能的巨大提升.

``` 
Pipeline pl = jedis.pipelined();
loop pl.hset("key", "field", "v");
pl.syncAndReturnAll()
```

###### 节省网络开销

###### 注意每次pipeline携带的数据量

###### 注意m操作与pipeline的区别：原子 vs 非原子

###### pipeline每次只能作用在一个redis节点上

##### 事务

###### multi/exec/discard

- 所有的指令在 exec 之前不执行，而是缓存在服务器的一个事务队列中，服务器一旦收到 exec 指令，才开执行整个事务队列，执行完毕后一次性返回所有指令的运行结果。

- 因为 Redis 的单线程特性，它不用担心自己在执行队列的时候被其它指令打搅，可以保证他们能得到的「原子性」执行

###### 隔离性

###### 不能保证原子性

Redis 的事务根本不能算「原子性」，而仅仅是满足了事务的「隔离性」，隔离性中的串行化——当前执行的事务有着不被其它事务打断的权利。

###### 结合pipeline

较少网络IO

pipe = redis.pipeline(transaction=true)
pipe.multi()
pipe.incr("books")
pipe.incr("books")
values = pipe.execute()

###### watch

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

##### 过期策略

###### 惰性策略

###### 定时扫描

Redis 默认会每秒进行十次过期扫描，过期扫描不会遍历过期字典中所有的 key，而是采用了一种简单的贪心策略。

从过期字典中随机 20 个 key；
删除这 20 个 key 中已经过期的 key；
如果过期的 key 比率超过 1/4，那就重复步骤 1；

###### 实践：过期时间随机化

##### 内部存储结构

###### redisObject

####### type 数据类型

string
hash
list
set
sorted set


####### encoding 编码方式

raw
int
ziplist
linkedlist
hashmap
intset



####### ptr 数据指针

####### vm 虚拟内存

####### 其他

#### 持久化

##### RDB

###### 触发条件

####### SAVE

######## 同步

######## 阻塞客户端命令

######## 不消耗额外内存

####### BGSAVE

######## 异步

######## 不阻塞客户端命令

######## fork子进程，消耗内存

子进程名：redis-rdb-bgsave


####### 配置文件: save seconds changes

save 900 1

save after 900 seconds if there is at least 1 change to the dataset

######## BGSAVE

######## 不建议打开

####### SHUTDOWN命令时

####### 从节点SYNC时 (=BGSAVE)

###### 原理

####### fork子进程生成快照


- 调用 glibc 的函数fork产生一个子进程，快照持久化完全交给子进程来处理，父进程继续处理客户端请求。

######## 内存越大，耗时越长

######## info: latest_fork_usec

####### COW (Copy On Write)

使用操作系统的 COW 机制来进行数据段页面的分离。
当父进程对其中一个页面的数据进行修改时，会将被共享的页面复制一份分离出来，然后对这个复制的页面进行修改。这时子进程相应的页面是没有变化的，还是进程产生时那一瞬间的数据。

###### 缺点

####### 不可控、会丢失数据

####### 耗时 O(n)、耗性能、耗内存

##### AOF

- 先执行指令才将日志存盘.
- 可用 `bgrewriteaof` 指令对 AOF 日志进行瘦身。

###### 触发条件

####### always

####### every second

####### no

###### 原理

####### 写命令刷新到缓冲区

####### 每条命令 fsync 到硬盘AOF文件

###### AOF重写

####### bgrewriteaof 命令

######## 类似bgsave, fork子进程重新生成AOF

####### 配置文件

######## auto-aof-rewrite-min-size

######## auto-aof-rerwrite-percentage

######## 推荐配置

``` 
appendonly yes
appendfilename "appendonly-${port}.aof"
appendfsync everysec
dir /bigdiskpath

no-appendfsync-on-rewrite yes

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

######## 动态应用配置

######### config set appendonly yes

######### config rewrite

###### AOF追加阻塞

####### 对比上次fsync时间，>2s则阻塞

####### info: aof_delayed_fsync (累计值)

##### 建议

###### 混合

在 Redis 重启的时候，可以先加载 rdb 的内容，然后再重放增量 AOF 日志。比 AOF 全量文件重放要快很多。

###### 持久化操作主要在从节点进行

#### 集群

##### 主从

###### 配置

####### slaveof

- 命令
- 配置

####### slave-read-only yes

####### 查看主从状态：info replication

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

###### 主从复制流程

####### 全量复制

######## 1.【s -> m】psync runId offset

首次：`psync ? -1`

######## 2.【m -> s】+FULLRESYNC {runId} {offset}

######## 3.【s】save masterInfo

######## 4.【m】bgsave / write repl_back_buffer

######## 5.【m -> s】send RDB

######## 6.【m -> s】send buffer

######## 7.【s】flush old data

######## 8.【s】load RDB

####### 部分复制

######## 1.【s -> m】psync runId offset

######## 2.【m -> s】CONTINUE

######## 3.【m -> s】send partial data

###### 问题

####### 开销大

######## 【m】bgsave时间开销

######## 【m】RDB网络传输开销

######## 【s】清空数据时间开销

######## 【s】加载RDB时间开销

######## 【s】可能的AOF重写时间

####### 读写分离问题

######## 复制数据延迟

######## 读到过期数据

######## 从节点故障

####### 主从配置不一致

######## maxmemory配置不一致

######### 可能丢失数据

master: maxmemory=4g
slave: maxmemory=2g

当数据>2g，slave会使用maxmemory-policy删除数据，failover之后的表现就是丢失数据。

######## 数据结构优化参数不一致

######### 内存不一致

####### 规避全量复制

######## 第一次全量复制

######### 不可避免

######### 优化：小主节点（小分片），低峰

######## 节点runId不匹配导致复制

######### 主节点重启后runId变化

######### 优化：故障转移（哨兵、集群）

######## 复制积压缓冲区不足

######### 网络中断后无法进行部分复制

######### 优化：rel_backlog_size（默认1m）

####### 规避复制风暴

######## 主节点重启，多个从节点复制

######## 优化：更换复制拓扑

##### sentinel

###### 原理

####### 三个定时任务

######## 每1秒，sentinel对其他sentinel和redis执行ping

######### 心跳检测

######### 失败判定依据

######## 每2秒，sentinel通过master的channel交换信息

######### master频道：__sentinel__:hello

######### 交换对节点的看法、以及自身信息

######## 每10秒，sentinel对m/s执行info

######### 发现slave节点

sentinel初始配置只关心master节点

######### 确认主从关系

####### 故障转移流程

########  sentinel 集群可看成是一个 ZooKeeper 集群

######## 【1. 故障发现】多个sentinel发现并确认master有问题

######### 主观下线

相关配置
```
sentinel monitor myMaster <ip> <port> <quorum>
sentinel down-after-milliseconds myMaster <timeout>
```

######### 客观下线

######## 【2. 选主】选举出一个sentinel作为领导

######### 原因：只有一个sentinel节点完成故障转移

######### 实现：通过sentinel is-master-down-by-addr命令竞争领导者

######## 【3. 选master】选出一个slave作为master, 并通知其余slave

######### 选新 master 的原则

########## 选slave-priority最高的

########## 选复制偏移量最大的

########## 选runId最小的

######### 对这个slave执行slaveof no one

######## 【4. 通知】通知客户端主从变化

######## 【5. 老master】等待老的master复活成为新master的slave

######### sentinel会保持对其关注

####### 客户端流程

######## 【0. sentinel集合】 预先知道sentinel节点集合、masterName

######## 【1. 获取sentinel】遍历sentinel节点，获取一个可用节点

######## 【2. 获取master节点】get-master-addr-by-name masterName

######## 【3. role replication】获取master节点角色信息

######## 【4. 变动通知】当节点有变动，sentinel会通知给客户端 （发布订阅）

######### JedisSentinelPool -> MasterListener --> sub "+switch-master"

######### sentinel是配置中心，而非代理！

###### 消息丢失

####### min-slaves-to-write 1

主节点必须至少有一个从节点在进行正常复制，否则就停止对外写服务，丧失可用性

####### min-slaves-max-lag 10

如果 10s 没有收到从节点的反馈，就意味着从节点同步不正常

###### 运维

####### 上下线节点

######## 下线主节点

######### sentinel failover <masterName>

######## 下线从节点

######### 考虑是否做清理、考虑读写分离

######## 上线主节点

######### sentinel failover进行替换

######## 上线从节点

######### slaveof

######## 上线sentinel

######### 参考其他sentinel节点启动

####### 高可用读写分离

######## client关注slave节点资源池

######## 关注三个消息

######### +switch-master: 从节点晋升

######### +convert-to-slave: 切换为从节点

######### +sdown: 主观下线

##### codis

###### 用zookeeper存储槽位关系

###### 代价

- 不支持事务；
- 同样 rename 操作也很危险；
- 为了支持扩容，单个 key 对应的 value 不宜过大。
- 因为增加了 Proxy 作为中转层，所有在网络开销上要比单个 Redis 大。
- 集群配置中心使用 zk 来实现，意味着在部署上增加了 zk 运维的代价

####### 不支持事务

####### rename 操作也很危险

####### 为了支持扩容，单个 key 对应的 value 不宜过大

####### 网络开销更大

####### 需要运维zk

##### cluster

###### 创建

####### 原生

######## 配置文件：cluster-enabled yes

cluster-enabled yes
cluster-config-file "xx.conf"
cluster-require-full-coverage no
cluster-node-timeout 15000

######## 启动: redis-server *.conf

######## gossip通讯：cluster meet ip port

######## 分配槽(仅对master)：cluster addslots {0...5461}

######## 配置从节点：cluster replicate node-id

####### 脚本

######## 安装ruby

######## 安装rubygem redis

######## 安装redis-trib.rb

####### 验证

######## cluster nodes

######## cluster info

######## cluster slot

######## redis-trib.rb info ip:port

###### 特性

####### 复制

######## 主从复制 (异步)：SYNC snapshot + backlog队列

- slave启动时，向master发起 `SYNC` 命令。

- master收到 SYNC 后，开启 `BGSAVE` 操作，全量持久化。

- BGSAVE 完成后，master将 `snapshot` 发送给slave.

- 发送期间，master收到的来自clien新的写命令，正常响应的同时，会再存入一份到 `backlog 队列`。

- snapshot 发送完成后，master会继续发送backlog 队列信息。

- backlog 发送完成后，后续的写操作同时发给slave，保持实时地异步复制。

######### 快照同步

######### 增量同步

异步将 buffer 中的指令同步到从节点，从节点一边执行同步的指令流来达到和主节点一样的状态，一边向主节点反馈自己同步到哪里了 (偏移量)。

######### 无盘复制

无盘复制是指主服务器直接通过套接字将快照内容发送到从节点，生成快照是一个遍历的过程，主节点会一边遍历内存，一边将序列化的内容发送到从节点，从节点还是跟之前一样，先将接收到的内容存储到磁盘文件中，再进行一次性加载。


######### wait 指令

wait 指令可以让异步复制变身同步复制，确保系统的强一致性。
- `wait N t`: 等待 wait 指令之前的所有写操作同步到 N 个从库，最多等待时间 t。

####### 高可用

####### 分片

######## slots

######### 16384个

######### 槽位信息存储于每个节点中

########## Rax

`Rax slots_to_keys` 用来记录槽位和key的对应关系
- Radix Tree 基数树


######### 每个节点通过meet命令交换槽位信息

######### 定位：crc16(key) % 16384

######## 计算槽位

######### cluster keyslot k

######## 扩展性：迁移slot (同步)

- Redis 迁移的单位是槽，当一个槽正在迁移时，这个槽就处于中间过渡状态。这个槽在原节点的状态为`migrating`，在目标节点的状态为`importing`，  


- 迁移过程是同步的，在目标节点执行`restore指令`到原节点删除key之间，原节点的主线程会处于阻塞状态，直到key被成功删除。 >> 要尽可能避免大key

原理：
- 以原节点作为目标节点的「客户端」
- 原节点对当前的key执行dump指令得到序列化内容，然后发送指令restore携带序列化的内容作为参数
- 目标节点再进行反序列化就可以将内容恢复到目标节点的内存中
- 原节点收到后再把当前节点的key删除


######### dump

######### restore

######### remove

###### 原理

####### 伸缩

######## 扩容

######### 准备新节点

######### 加入集群

########## meet

########## redis-trib.rb add-node

redis-trib.rb add-node new_host:new_port existing_host:existing_port --slave --master-id

######### 迁移槽和数据

########## 手工

########### 1_对目标节点：cluster setslot {slot} importing {sourceNodeId}

########### 2_对源节点：cluster setslot {slot} migrating {targetNodeId}

########### 3_对源节点循环执行：cluster getkeysinslot {slot} {count}，每次获取count个键

########### 4_对源节点循环执行：migrate {targetIp} {targetPort} key 0 {timeout}

0: db0


########### 5_对所有主节点：cluster setslot {slot} node {targetNodeId}

########## pipeline migrate

########## redis-trib.rb reshard

redis-trib.rb reshard host:port
--from
--to
--slots

host:port是任一个节点的

######## 收缩

######### 迁移槽

######### 忘记节点

########## cluster forget {downNodeId}

########## redis-trib.rb del-node

redis-trib.rb del-node ip:port {downNodeId}

######### 关闭节点

######## 迁移slot过程中如何同时提供服务？--> ask

######### 0.先尝试访问源节点

######### 1.源节点返回ASK转向

######### 2.向新节点发送asking命令

在迁移没有完成之前，这个槽位还是不归新节点管理的，它会向客户端返回一个`-MOVED`重定向指令告诉它去源节点去执行。如此就会形成 `重定向循环`。
asking指令的目标就是打开目标节点的选项，告诉它下一条指令不能不理，而要当成自己的槽位来处理。

######### 3.向新节点发送命令

####### 故障转移

######## 故障发现

######### 通过ping/pong发现故障

######### 主观下线

- node1 发送ping消息
- node2 回复pong消息
- node1 收到pong，并更新与node2的`最后通信时间`
- cron定时任务：如果最后通信时间超过node-timeout，则标记为`pfail`

######### 客观下线

- 接受ping
- 消息解析：其他pfail节点 + 主节点发送消息
- 维护故障链表
- 尝试客观下线：计算有效下线报告数量
- if > 槽节点总数一半，则更新为客观下线；
-并向集群广播下线节点的fail消息。

########## 当半数以上主节点都标记其为pfail

######## 故障恢复

######### 资格检查

每个从节点：检查与主节点断线时间；
- if > `cluster-node-timeout` * `cluster-slave-validity-factor`，则取消资格

######### 准备选举时间

offset越大，则延迟选举时间越短


- slave通过向其他master发送FAILOVER_AUTH_REQUEST消息发起竞选，master回复FAILOVER_AUTH_ACK告知是否同意。

######### 选举投票

收集选票，if > N/2 + 1，则可替换zhu'jie'dian

######### 替换主节点

1. slaveof no one
2. clusterDelSlot撤销故障主节点负责的槽；
3. clusterAddSlot把这些槽分配给自己；
4. 向集群广播pong消息，表明已经替换了故障jie

####### 一致性: 保证朝着epoch值更大的信息收敛

保证朝着epoch值更大的信息收敛: 每一个Node都保存了集群的配置信息`clusterState`。

- `currentEpoch`表示集群中的最大版本号，集群信息每更新一次，版本号自增。
- nodes列表，表示集群所有节点信息。包括该信息的版本epoch、slots、slave列表

配置信息通过Redis Cluster Bus交互(PING / PONG, Gossip)。
- 当某个节点率先知道信息变更时，将currentEpoch自增使之成为集群中的最大值。
- 当收到比自己大的currentEpoch，则更新自己的currentEpoch使之保持最新。
- 当收到的Node epoch大于自身内部的值，说明自己的信息太旧、则更新为收到的消息。


###### 客户端

####### 客户端路由

######## moved

######### 1.向任意节点发送命令

######### 2.节点计算槽和对应节点

######### 3.如果指向自身，则执行命令并返回结果

######### 4.如果不指向自身，则回复-moved (moved slot ip:port)

######### 5.客户端重定向发送命令

######## tips

######### redis-cli -c 会自动跳转到新节点

######### moved vs. ask

########## 都是客户端重定向

########## moved: 表示slot确实不在当前节点（或已确定迁移）

########## ask: 表示slot在迁移中

####### 批量操作

######## 问题：mget/mset必须在同一个槽

######## 实现

######### 串行 mget

######### 串行IO

########## 客户端先做聚合，crc32 -> node，然后串行pipeline

######### 并行IO

########## 客户端先做聚合，然后并行pipeline

######### hash_tag

###### 运维

####### 集群完整性

######## cluster-require-full-coverage=yes

######### 要求16384个槽全部可用

######### 节点故障或故障转移时会不可用：(error) CLUSTERDOWN

######## 大多数业务无法容忍

####### 带宽消耗

######## 来源

######### 消息发送频率

节点发现与其他节点最后通信时间超过 cluster-node-timeout/2 时，会发送ping消息


######### 消息数据量

- slots槽数据：2k
- 整个集群1/10的状态数据

######### 节点部署的机器规模

集群分布的机器越多，且每台机器划分的节点数越均匀，则集群内整体可用带宽越高。

######## 优化

######### 避免“大”集群

######### cluster-node-timeout: 带宽和故障转移速度的均衡

######### 尽量均匀分配到多机器上

######## 集群状态下的pub/sub

######### publish在集群中每个节点广播，加重带宽

######### 解决：单独用一套sentinel

####### 倾斜

######## 数据倾斜

######### 节点和槽分配不均

########## redis-trib.rb info 查看节点、槽、键值分布

########## redis-trib.rb rebalance 重新均衡（慎用）

######### 不同槽对应键值数量差异较大

########## CRC16一般比较均匀

########## 可能存在hash_tag

########## cluster countkeyinslot {slot} 查看槽对应键值数

######### 包含bigkey

########## 从节点执行 redis-cli --bigkeys

########## 优化数据结构

######### 内存配置不一致

######## 请求倾斜

######### 原因：热点key、bigkey

######### 优化

########## 避免bigkey

########## 热键不要用hash_tag

########## 一致性不高时，可用本地缓存，MQ

####### 读写分离

######## 只读连接

######### 从节点不接受任何读写请求

######### 会重定向到负责槽的主节点（moved）

######### readonly命令

- 默认情况下，某个slot对应的节点一定是一个master节点。客户端通过`MOVED`消息得知的集群拓扑结构也只会将请求路由到各个master中。

- 即便客户端将读请求直接发送到slave上，slave也会回复MOVED到master的响应。

- 客户端向slave发送READONLY命令后，slave对于读操作将不再返回moved，而是直接处理。

######## 读写分离客户端会非常复杂

######### 共性问题：复制延迟、过期数据、从节点故障

######### cluster slaves {nodeId} 获取从节点列表

####### 数据迁移

######## redis-trib.rb import

######### 只能 单机 to 集群

######### 不支持在线迁移

######### 不支持断点续传

######### 单线程迁移，影响速度

######## 在线迁移

######### 唯品会 redis-migrate-tool

######### 豌豆荚 redis-port

###### 缺点

####### key批量操作支持有限

######## mget/mset 必须在同一个slot

####### key事务和lua支持有限

######## 操作的key必须在同一个slot

####### key是数据分区最小粒度

######## bigkey无法分区

######## 分支主题

####### 复制只支持一层

######## 无法树形复制

#### 应用

##### 分布式锁

###### 命令

####### setnx + expire

####### set xx ex 5 nx

###### 集群问题

主节点挂掉时，从节点会取而代之，客户端上却并没有明显感知。原先第一个客户端在主节点中申请成功了一把锁，但是这把锁还没有来得及同步到从节点，主节点突然挂掉了。然后从节点变成了主节点，这个新的节点内部没有这个锁，所以当另一个客户端过来请求加锁时，立即就批准了。这样就会导致系统中同样一把锁被两个客户端同时持有

####### Redlock算法

过半节点加锁成功


##### 延时队列

###### lpush / rpush 

###### rpop / lpop -> brpop / blpop

##### 位图

位图不是特殊的数据结构，它的内容其实就是普通的字符串，也就是 byte 数组。

我们可以使用普通的 get/set 直接获取和设置整个位图的内容，也可以使用位图操作 getbit/setbit 等将 byte 数组看成「位数组」来处理。

###### type: string, 最大512M

###### 命令

####### setbit k offset v

零存：`setbit s 4 1`
整存：`set s <string>`

####### getbit

整取：`get s`
零取：`getbit s 1`


####### bitcount k [start end] 统计

####### bitop op destKey key1 key2 位运算

op:
- and
- or
- not
- xor

####### bitpos k targetBit [start] [end] 查找

####### bitfield操作多个位

##### HyperLogLog

HyperLogLog 提供不精确的去重计数方案

###### 极小空间完成独立数量统计

###### type: string

###### 缺点

####### 有错误率 0.81%

####### 不能返回单条元素

###### 命令

####### 添加：pfadd key e1 e2...

####### 计数：pfcount key

####### 合并：pfmerge destKey sourceKey1 sourceKey2

##### 布隆过滤器

布隆过滤器能准确过滤掉那些已经看过的内容，那些没有看过的新内容，它也会过滤掉极小一部分 (误判)

###### bf.add / bf.madd

###### bf.exists / bf.mexists

##### 简单限流: zset实现滑动时间窗口

###### key: clientId-actionId

###### value: ms

###### score: ms

##### 漏斗限流: redis-cell模块

###### cl.throttle key capacity count period 1

##### GeoHash

GeoHash 算法将二维的经纬度数据映射到一维的整数，这样所有的元素都将在挂载到一条线上，距离靠近的二维坐标映射到一维后的点之间距离也会很接近。

###### 用于地理经纬度计算

###### type: zset

###### 命令

####### 添加：geoadd key longitude latitude member

####### 获取：geopos key member

####### 距离：geodist key member1 member2 [unit]

####### 范围：georadius/georadiusbymember 

####### 删除：zrem key member

####### geohash

##### 搜索key

###### keys

- 没有 offset、limit 参数，一次性吐出所有满足条件的 key。
- keys 算法是遍历算法，复杂度是 O(n)

###### scan

scan <cursor> match <regex> count <limit>
在 Redis 中所有的 key 都存储在一个很大的字典中，scan 指令返回的游标就是第一维数组的位置索引，limit 参数就表示需要遍历的槽位数
  

##### Stream

##### PubSub

###### publish channel-name msg

###### subscribe/unsubscribe channel-name

#### 运维

##### 内存回收

###### 无法保证立即回收已经删除的 key 的内存

###### flushdb

##### eviction

###### LRU: Least Recently Used

当字典的某个元素被访问时，它在链表中的位置会被移动到表头。

所以链表的元素排列顺序就是元素最近被访问的时间顺序。

位于链表尾部的元素就是不被重用的元素，所以会被踢掉。

- 缺点：需要大量的额外的内存


###### 近似LRU

- **随机**采样出 5(可以配置) 个 key，
- 然后淘汰掉最旧的 key，
- 如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于 maxmemory 为止。

Redis给每个 key 增加了一个额外的小字段，这个字段的长度是 24 个 bit，也就是最后一次被访问的时间戳。



###### LFU: Least Frequently Used

##### 保护

###### rename-command flushall ""

######  spiped: SSL代理

##### 懒惰删除

###### del -> unlink

###### flushdb -> flushdb async

##### 慢查询

###### 配置

####### slowlog-max-len

######## 先进先出队列、固定长度、内存

######## 默认10ms, 建议1ms

####### slowlog-log-slower-than

######## 建议1000

###### 命令

####### slowlog get [n]

####### slowlog len

####### slowlog reset

### MongoDb

#### 常用命令

##### show dbs / collections

##### use db1

##### db.collection1.find();

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

## 微服务

### Spring Cloud

### service mesh

#### consumer端 sidecar

##### 服务发现

##### 负载均衡

##### 熔断降级

#### provider端 sidecar

##### 服务注册

##### 限流降级

##### 监控上报

#### Control Plane

##### 服务注册中心

##### 负载均衡配置

##### 请求路由规则

##### 配额控制

### 高可用

#### 降级

##### hystrix fallback 原理

#### 熔断

#### 限流

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

##### Bean Factory: IoC容器

##### ApplicationContext: 应用上下文，Spring容器

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
- 以及方位信息


###### Target 目标对象

Advice增强逻辑的织入目标类

###### Introduction 引介

为类添加属性和方法，可继承 `DelegatingIntroductionInterceptor`

###### Weaving 织入

将Advice添加到目标类的具体连接点上的过程。

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

##### HandlerMapping

###### 通过HandlerMapping找到处理请求的Handler

##### HanlderAdapter

###### 通过HandlerAdapter封装调用Handler

###### HttpMessageConverter

####### MappingJackson2HttpMessageConverter

##### ViewResolver

###### 通过ViewResolver解析视图名到真实视图对象

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

###### @RequestParam

###### @RequestHeader

###### @CookieValue

###### @MatrixVariable

###### HttpServletRequest / HttpSession

###### WebRequest

###### ModelMap / Map

SpringMVC在调用方法前会创建一个隐含的模型对象。如果方法入参为Map/Model，则会将隐含模型的引用传递给这些入参。


##### 入参原理

###### HttpMessageConverter

- MappingJackson2HttpMessageConverter
- ByteArrayHttpMessageConverter

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

### 排序

#### 插入排序

在要排序的一组数中，假设前面(n-1) [n>=2] 个数已经是排好顺序的。现在要把第n个数插到前面的有序数中，使得这n个数也是排好顺序的。

##### O(N^2)

#### 选择排序

- 在要排序的一组数中，选出最小的一个数与第一个位置的数交换；
- 然后在剩下的数当中再找最小的与第二个位置的数交换，
- 如此循环到倒数第二个数和最后一个数比较为止。


##### O(N^2)

#### 归并排序

归并（Merge）排序法是将两个有序表合并成一个新的有序表。即：

- 把待排序序列分为若干个子序列，每个子序列是有序的。
- 然后再把有序子序列合并为整体有序序列。

##### O(NlogN)

#### 快速排序

- 选择一个基准元素，通常选择第一个元素或者最后一个元素。
- 通过一趟扫描，将待排序列分成两部分：一部分比基准元素小；一部分大于等于基准元素。
- 
此时基准元素在其排好序后的正确位置，然后再用同样的方法递归地排序划分的两部分。

##### O(NlogN)

#### 冒泡排序

在要排序的一组数中，对当前还未排好序的范围内的全部数，自上而下对相邻的两个数依次进行比较和调整，让较大的数往下沉，较小的往上冒。

即：每当两相邻的数比较后发现它们的排序与排序要求相反时，就将它们互换。

#### 希尔排序

算法先将要排序的一组数按某个增量d（n/2,n为要排序数的个数）分成若干组，每组中记录的下标相差d.对每组中全部元素进行`直接插入排序`，
然后再用一个较小的增量（d/2）对它进行分组，在每组中再进行直接插入排序。
当增量减到1时，进行直接插入排序后，排序完成。

#### 堆排序

堆排序需要两个过程，一是建立堆，二是堆顶与堆的最后一个元素交换位置。

所以堆排序有两个函数组成。
- 一是建堆的渗透函数，
- 二是反 复调用渗透函数实现排序的函数。

#### 基数排序

- 将所有待比较数值（正整数）统一为同样的数位长度，数位较短的数前面补零。
- 然后，从最低位开始，依次进行一次排序。
- 这样从最低位排序一直到最高位排序完成以后，数列就变成一个有序序列。


#### 二分查找

##### O(logN)
