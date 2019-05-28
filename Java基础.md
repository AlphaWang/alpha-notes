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

###### 创建

####### NIO可能会操作堆外内存：Buffer.isDirect()

####### FileChannel.map()创建MappedByteBuffer

将文件按照指定大小直接映射为内存区域，

###### 场景

####### 创建和销毁开销大

####### 适用于场景使用、数据较大的场景

###### 回收

####### -XX:MaxDirectMemorySize

####### -XX:NativeMemoryTracking={summary|detail}

####### 回收：FullGC时顺便清理，不能主动触发

####### 异常OutOfMemoryError: Direct buffer memory

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

###### 分析死锁：找到BLOCKED的线程

##### jcmd

`jcmd 1 GC.heap_dump ${dump_file_name}`

JFR.stop
JFR.start
JFR.dump
JFR.check

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
VM.check_commercial_features
VM.unlock_commercial_features
VM.native_memory detail 
VM.native_memory baseline
VM.native_memory detail.diff

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

##### -XX:NativeMemoryTracking={summary|detail}

#### 并发

##### -XX:-UseBiasedLocking 关闭偏向锁

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

### 集合

#### HashMap

##### resize与多线程

https://mailinator.blogspot.com/2009/06/beautiful-race-condition.html

```java
1:  // Transfer method in java.util.HashMap -
2:  // called to resize the hashmap
3:  
4:  for (int j = 0; j < src.length; j++) {
5:    Entry e = src[j];
6:    if (e != null) {
7:      src[j] = null;
8:      do {
9:      Entry next = e.next; 
10:     int i = indexFor(e.hash, newCapacity);
11:     e.next = newTable[i];
12:     newTable[i] = e;
13:     e = next;
14:   } while (e != null);
15:   }
16: } 
```

### 函数式编程

#### Lambda表达式

##### 闭包

###### lambda表达式引用的是值，而非变量：虽然可以省略 final

###### 给变量多次赋值，然后再lambda中引用它，会编译报错

##### 函数接口

###### 即lambda表达式的类型

#### 流

##### 实现机制

###### 惰性求值方法

####### 返回Stream

###### 及早求值方法

####### 返回另一个值

##### stream() 常用流操作

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

##### 收集器

###### 原生方法

####### 转成其他收集器：toCollection(TreeSet::new)

####### 统计信息：maxBy, averagingInt

####### 数据分块：partitionBy

####### 数据分组：groupingBy

####### 拼接字符串：joining(分隔符，前缀，后缀)

####### 定制：reducing(identity, mapper, op)

###### 自定义收集器

####### supplier() -> Supplier，后续操作的初始值

####### accumulator() -> BiConsumer，结合之前操作的结果和当前值，生成并返回新值

####### combine() -> BinaryOperator，合并

####### finisher() -> Function，转换，返回最终结果

##### 并行流

###### 创建

####### Collection.parallelStream()

####### Stream.parallel()

###### 实现

####### 使用ForkJoinPool

###### 场景

####### 数据量越大，每个元素处理时间越长，并行就越有意义

####### 数据结构要易于分解

######## 好：ArrayList, IntStream

######## 一般：HashSet，TreeSet

######## 差：LinkedList，Streams.iterate，BufferedReader.lines

####### 避开有状态操作

######## 无状态：map，filter，flatMap

######## 有状态：sorted，distinct，limit

###### Arrays提供的并行操作

####### Arrays.parallelSetAll

######## 并行设置value

####### Arrays.parallelPrefix

######## 将每一个元素替换为当前元素与前驱元素之和

####### Arrays.parallelSort

#### 工具

##### 类库

###### mapToInt().summaryStatistics()

####### 数值统计：min, max, average, sum

###### @FunctionalInterface

####### 强制javac检查接口是否符合函数接口标准

##### 默认方法

###### 继承

####### 类中重写的方法胜出

###### 多重继承

####### 可用InterfaceName.super.method() 来指定某个父接口的默认方法

###### 用处

####### Collection增加了stream方法，如果没有默认方法，则每个Collection子类都要修改

##### 方法引用

###### 等价于lambda表达式，需要时才会调用

##### 集合类

###### computeIfAbsent()

###### forEach()

#### 范例

##### 封装：log.debug(Suppiler)

```java
// AS-IS
if (logger.isDebugEnabled()) {
  logger.debug("...");
}

// TO-BE 调用者无需关心日志级别
logger.debug(() -> "...");

public void debug(Supplier msg) {
  if (isDebugEnabled()) {
    debug(msg.get());
  }
}
```

##### 孤独的覆盖：ThreadLocal.withInitial(Supplier)

```java
// AS-IS
new ThreadLocal<Object>() {
  @Override
  protected Object initialValue() {
    return db.get();
  }
}

//TO-BE
ThreadLocal.withInitial(() -> db.get());
```

##### 重复：传入ToLongFunction

例如遍历订单中的item详情数目
```java
public long count(ToLongFunction<Order> fun) {
  return orders.stream()
    .mapToLong(fun)
    .sum();
}

// 否则调用端需要重复遍历orders
public long countItem() {
  return count(order -> order.getItems().count())
}

public long countAmount() {
  return count(order -> order.getAmoun().count())
}
```

#### 调试

##### peek()

peek()让你能查看每个值，同时能继续操作流。
```java
album.getMusicians()
  .filter(...)
  .map(...)
  .peek(item -> System.out.println(item))
  .collect(Collectors.toSet());
```

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

## 性能调优

### 性能测试

#### 指标

##### 响应时间

##### 吞吐量

##### 资源使用率：CPU，内存，IO

##### 负载承受能力：抛错的极限

### Java调优

#### String

##### 不可变性

##### 拼接：StringBuilder

##### String.intern()节省内存

注意：
- 常量池是类似一个HashTable，如果过大会增加整个字符串常量池的负担。

#### 正则表达式

##### 概念

###### 普通字符

####### [a-zA-Z], [0-9], ...

###### 标准字符

####### \d, \w, \s

###### 限定字符

####### *, +, ?, {n}

###### 定位字符

####### $, ^

##### 少用贪婪模式，多用独占模式

贪婪模式：ab{1,3}c
懒惰模式：ab{1,3}?c
独占模式：ab{1,3}+bc

##### 减少分支选择

(abcd|abef) --> ab(cd|ef)

##### 减少捕获嵌套

(x) --> (?:x)

### 多线程调优

### JVM调优

### 设计模式调优

### 数据库调优

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

#### Vector, ArrayList, LinkedList

##### fail-fast: ConcurrentModificationException

##### 集合排序算法

##### 类图

#### Hashtable, HashMap, TreeMap

##### equals vs. hashCode vs. compareTo

##### HashMap

###### 哈希计算：(key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)

为什么这里需要将高位数据移位到低位进行异或运算呢？
这是因为有些数据计算出的哈希值差异主要在高位，而 HashMap 里的哈希寻址是忽略容量以上的高位的，那么这种处理就可以有效避免类似情况下的哈希碰撞。

###### entry index算法：(n - 1) & hash

###### resize

- 门限值等于（负载因子）x（容量）;
- 门限通常是以倍数进行调整 （newThr = oldThr << 1）。
- 扩容后，需要将老的数组中的元素重新放置到新的数组，这是扩容的一个主要开销来源。

###### 容量、负载因子

#######  负载因子 * 容量 > 元素数量

####### 容量 > 预估元素数据 / 负载因子

###### 非线程安全

####### 并发访问无限循环占用CPU

####### 并发访问size不准

#### 线程安全集合

##### ConcurrentHashMap

###### 原理

####### 早期

######## 分段锁

######## concurrentLevel决定segment数量

######## volatile value保证可见性

######## get()

```java
public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key.hashCode());
       // 利用位操作替换普通数学运算
       long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        // 以 Segment 为单位，进行定位
        // 利用 Unsafe 直接进行 volatile access
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
           // 省略
          }
        return null;
    }

```

######### 二次哈希，缓解哈希冲突

######### 保证可见性

######## put()

```java
public V put(K key, V value) {
  Segment<K,V> s;
  // 二次哈希，以保证数据的分散性，避免哈希冲突
  int hash = hash(key.hashCode());
  int j = (hash >>> segmentShift) & segmentMask;
  if ((s = (Segment<K,V>)UNSAFE.getObject   // nonvolatile; recheck
  (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
    s = ensureSegment(j);
    return s.put(key, hash, value, false);
}


final V put(K key, int hash, V value, boolean onlyIfAbsent) {
  // scanAndLockForPut 会去查找是否有 key 相同 Node
  // 无论如何，确保获取锁
  HashEntry<K,V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);
  V oldValue;
  HashEntry<K,V>[] tab = table;
  int index = (tab.length - 1) & hash;
  HashEntry<K,V> first = entryAt(tab, index);
  for (HashEntry<K,V> e = first;;) {
    if (e != null) {
       K k;
       // 更新已有 value...
    } else {
      // 放置 HashEntry 到特定位置，如果超过阈值，进行 rehash
      // ...
    }
}



```

######### 二次哈希

######### 获取锁：Segment extends ReentrantLock

######### 扩容：针对Segment

######## size()

######### 要同步吗？

######### 重试机制 RETRIES_BEFORE_LOCK

####### java8

######## 不再用Segment，仅保留概念以便兼容

######## 初始化 lazy-load

######## CAS

#### NIO

##### 演进

###### java.io / java.net

####### 同步阻塞IO

####### File, Socket, HttpURLConnection

###### 1.4 NIO

####### 同步非阻塞IO

####### Channel, Selector

###### 1.7 NIO2

####### 异步非阻塞IO（AIO）

####### 基于事件和回调机制

##### 概念

###### Buffer

####### 数据容器

######## capacity: 数组长度

######## position: 要处理的起始位置

######## limit: 要处理的最大位置

###### Channel

####### 类似Linux文件描述符

####### 比File/Socket更底层

###### Selector

####### Channel可以注册在Selector上

####### NIO会判断Selector上的Channel是否处于就绪状态

#### 文件拷贝方式

##### 方式

###### NIO: transferTo

public void copyFileByChannel(File source, File dest) {
  try (FileChannel sourceChannel = new FileInputStream(source).getChannel();
    FileChannel targetChannel = new FileOutputStream(dest).getChannel();) {
     for (long count = sourceChannel.size();count>0;) {
       long transferred = sourceChannel.transferTo(
                    sourceChannel.position(), count, targetChannel); 
                    sourceChannel.position(sourceChannel.position() + transferred);
       count -= transferred;
     }
    }
 }



###### FileInputStream --> FileOutputStream

```java
public void copyFileByStream(File source, File dest) {
    try (InputStream is = new FileInputStream(source);
    OutputStream os = new FileOutputStream(dest);){
      byte[] buffer = new byte[1024];
      int length;
      while ((length = is.read(buffer)) > 0) {
          os.write(buffer, 0, length);
      }
  }
}


```

###### Files.copy()

####### 并非零拷贝！

##### 实现机制

###### 内核态数据 -> 用户态数据 -> 内核态数据

###### 零拷贝

#### 接口 vs. 抽象类

##### 不支持多重继承 --> 所以引入工具类（例如Collections）--> default method

##### Marker Interface --> 还可用Annotation

#### 设计模式

##### SOLID

- Single Responsibility
- Open-Close
- Liskov Substitution
- Interface Segregation
- Dependent Inversion

### 并发

#### synchronized vs. ReentrantLock

#### 锁的升级降级

#### thread.start

#### 死锁

#### 并发工具类

#### ConcurrentLinkedQueue vs. LinkedBlockingQueue

#### 线程池种类

#### AtomicInteger原理

### JVM

#### 类加载过程

#### 如何动态生成一个类

#### 内存区域的划分

#### 如何监控诊断堆内、堆外内存

#### 常见垃圾收集器

#### GC调优思路

#### happen-before是什么

#### Java运行在Docker中的问题

### 安全和性能

#### 注入攻击

#### 安全的Java代码

#### 性能优化思路

#### Lambda对性能的影响

#### JVM优化代码

### 扩展

#### 事务隔离级别

#### 悲观锁，乐观锁

#### Spring Bean生命周期、作用域

#### Netty是如何实现高性能的

#### 分布式ID设计方案
