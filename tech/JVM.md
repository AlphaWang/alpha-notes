[toc]

# | 内存区域

## || 运行时数据区

### 程序计数器

- **线程私有**
- 作用：记录各个线程执行的字节码地址
- 异常：无 OOM



### JVM 栈

- **线程私有**

- 栈桢 Stack Frame

  - 局部变量表

    > 存放方法参数、方法内定义的局部变量；
    >
    > 容量以slot为最小单位；
    >
    > 为了尽可能节省栈桢空间，局部变量表中的slot可以重用；
    >
    > 局部变量无“准备阶段”，无初始值

  - 操作数栈

    > 存放算术运算操作数、调用其他方法的参数

  - 动态连接

  - 返回地址

    > 正常完成出口 -Normal Method Invocation Completion
    >
    > 异常完成出口 -Abrupt Method Invocation Completion
    >
    > 方法退出时的操作
    >
    > - 回复上层方法的局部变量表、操作数栈
    >
    > - 把返回值压入调用者栈桢的操作数栈
    >
    > - 调整PC寄存器的值，指向方法调用指令后面的一条指令

- 异常

  - **StackOverflowError**，当栈深度过大

  - **OOM**：动态扩展时无法申请到足够内存。但 Hotspot 虚拟机不允许动态扩展，所以“单线程情况下”其只会抛出 StackOverflowError

    > **OOME: unable to create native thread**
    >
    > 复现：如果`-Xss`设置过大，当创建大量线程时可能OOM.（每个线程都会创建一个栈）



### 本地方法栈

- 管理本地方法调用



### 堆

- **线程共享**

  > 注意区别线程私有的分配缓冲区 TLAB，Thread Local Allocation Buffer

- 作用：存储对象实例、数组

- 异常

  - **OOM**：内部不够实例分配、且堆也无法再扩展时，调整 `-Xms` `-Xmx`

    > **OOME: Java heap space**
    >
    > 复现：不断往list中加入对象



### 方法区

- **线程共享**

- 作用：存储类信息、常量、静态变量、运行时常量

  > 扩展：运行时常量池的来源
  >
  > 1. class 文件常量池
  > 2. 运行时动态放入：intern()



- **方法区 vs. 永久代**
  - 方法区是JVM规范
  - HotSpot虚拟机使用永久代来实现

- 演变
  - Java7：将永久代的静态变量、字符串常量池 --> 合并到堆

  - Java8：替换为元空间 --> 存储在本地内存

    > 融合HotSpot 与 JRockit
    >
    > 避免永久代溢出 OOME: PermGen

- 异常
  - **OOM**：String.intern 运行期间将新的常量放入池中（Java7 以后不能复现，因为移到堆中）、CGLib动态类

    > 例如大量String.intern()、用CGLib生成大量动态类、OSGi应用、大量JSP的应用（JSP第一次运行需要便以为Javale）
    > `-XX:PermSize` `-XX:MaxPermSize`

- GC目标

  - 回收废弃的常量

  - 卸载类型

    > 条件 
    >
    > 类的所有实例都已被回收
    >
    > 对应的类加载器已被回收
    >
    > 对应的Class对象未被引用



## || 其他

### Direct Memory 堆外内存

https://www.baeldung.com/native-memory-tracking-in-jvm

创建

- NIO 通过 DirectByteBuffer 操作堆外内存：Buffer.isDirect()

- FileChannel.map() 创建 MappedByteBuffer：将文件按照指定大小直接映射为内存区域，

场景

- 创建和销毁开销大

- 适用于场景使用、数据较大的场景

回收

- -XX:MaxDirectMemorySize

- -XX:NativeMemoryTracking={summary|detail}

- 回收：FullGC时顺便清理，不能主动触发

异常

- **OOME: Direct buffer memory**

  > 复现：unsafe.allocateMemory() 
  >
  > 特点：dump文件看不到明显异常



### Threads 线程堆栈

异常

- 纵向异常：无法分配新的栈桢：**StackOverflowError**

- 横向异常：无法建立新的线程：**OutOfMemoryError: unable to create new native thread**



### Metaspace

- -XX:MetaspaceSize
- -XX:MaxMetaspaceSize



### Code Cache

- JIT编译器使用
- -XX:InitialCodeCacheSize 
  -XX:ReservedCodeCacheSize



### Garbage Collection

GC算法用off-heap数据结构来完成GC

### Symbols: String常量池





# | 对象

## || 类文件结构

**对象头 Header**

- Mark Word
  - HashCode
  - GC分代年龄
  - 锁状态标志
  - 线程持有的锁
  - 偏向线程ID
  - 偏向时间戳

- 类型指针
  - JVM通过这个指针来确定对象是哪个类的实例
  - 数组对象还会存储长度

  

**实例数据 Instance Data**

- 各种类型的字段内容（包含父类定义的）



**对齐填充 Padding**



**类文件结构**

- 魔数、版本号

  > 魔数类似于扩展名，但更安全

- 常量池 constant_pool

  - 字面量 Literal：Java 语言层面

  - 符号引用 Symbolic Reference：编译原理层面

    > - 类和接口的 全限定名
    >
    > - 字段名和描述符
    >
    > - 方法名和描述符
    >
    > JVM做类加载时，会从常量池获得对应的“符号引用”，再在类创建时或运行时解析、翻译到具体的内存地址之中。

- 访问标志 access_flags

  > 类/接口层次的访问信息，例如 类还是接口、是否public、是否abstract、是否final；
  >
  > 16 个比特位

- 继承关系

  > this_class \ super_class \ interfaces
  >
  > 此处只存索引，在常量池中查找

- 字段表集合 field_info

  > access_flag：是否public、static、final、volatile、transient
  >
  > name_index: 简单名称，指向常量池
  >
  > descriptor_index: 描述符。字段数据类型，方法参数列表、返回值
  >
  > attributes: 属性表：额外信息，例如final定义的常量值

- 方法表集合
  - 类似字段表集合；
  - 方法体存放在属性表`Code`属性里


方法的字节码，
访问权限，
方法名索引（与常量池中的方法引用对应），
描述符索引，
JVM执行指令，
属性集合，



## || 对象的访问定位

### 句柄访问

概念

- 堆分为“句柄池”、“实例池”

- 栈中的`reference`指向`句柄地址`，而句柄则指向 `对象实例数据、类型数据`

优点

- 对象被移动时（GC）只会改变句柄中的实例数据指针，而reference本身无需被修改



### 直接访问

概念

- 堆中只存储实例，实例数据会指向类型数据

- 栈中的reference指向对象地址

优点

- 速度更快

- 节省了一次指针定位的开销：reference --> 句柄 --> 实例/类型



## || 类加载

### 对象生命周期



**1. 在常量池找类的符号引用**



**2. 加载 Loading**

> 作用：查找字节流，并且据此创建类

- 根据全限定名来获取二进制字节流 `Class Loader`；
- 将字节流所代表的静态存储结构转化为方法区的运行时数据结构；
- 生成java.lang.Class对象

注意：数组类本身不通过类加载器创建，而是有JVM直接创建。



双亲委派模型

- 类加载器

  - 启动类加载器 Bootstrap ClassLoader

    > 加载 jre/lib
    >
    > 如何替换基础类库实现：-Xbootclasspath

  - 扩展类加载器 Extension ClassLoader

    > 加载jre/lib/ext
    >
    >  Java9改名为平台类加载器
    >
    > 如何替换扩展类库实现：-Djava.ext.dirs=

  - 应用类加载器 Application ClassLoader

    > 加载classpath
    >
    > 自定义：重写findClass, 而非loadClass



- 被破坏

  - jdk1.2之前
  - JNDI等SPI框架需要加载classpath代码
  - OSGi: 网状

  

**3. 连接 Linking**

> 作用：把原始的类定义信息 平滑地转入JVM运行的过程

步骤

- **验证 Verification**
  确保被加载类能够满足 Java 虚拟机的约束条件；否则抛出`VerifyError`
  - 文件格式验证：魔数开头、版本号、可能抛出`VerifyError`

  - 元数据验证：是否有父类、是否实现了接口中要求的方法

  - 字节码验证：通过数据流和控制流分析，确定程序语义是否合法

  - 符号引用验证：在讲符号引用转化为直接引用时进行（解析阶段），可能抛出`IllegalAccessError`, `NoSuchFieldError`, `NoSuchMethodError

- **准备 Preparation**
  - 在方法区中，为静态变量分配内存，并设置初始值

- **解析 Resolution**
  - 将常量池中的`符号引用` 替换为 `直接引用`



**4. 初始化 Initialization**

> 作用：执行类构造器<clinit>()方法

触发

- new / 反射

  > 5 种情况必须立即对类进行初始化：
  >
  > - new, getstatic  
  > - 反射调用  
  > - 先触发父类初始化  
  > - main  
  > - java7 REF_getStatic ?? 

- 反例
  - `SubClass.staticValue` 通过子类引用父类静态字段，子类不会被初始化
  - MyClass[]` 通过数组定义来引用类，不会触发类初始化
  - `MyCalss.CONSTANT_VALUE` 引用常量，不会触发类初始化



步骤

- 执行clinit方法（静态代码块）

- 常量值赋值



**初始化仅会被执行一次：可用于实现“单例”**

- clinit包括 类变量赋值动作、静态语句块。
- clinit保证多线程环境中被加锁、同步 - clint方法是带锁线程安全



**5. 分配内存**

- **指针碰撞**
  - 当内存规整时适用
  - 适用于Compact GC，例如Serial, ParNew

- **空闲列表**
  - 当内存不连续时适用
  - 适用于Mark-Sweep GC，例如CMS

- **并发问题**
  - CAS
  - TLAB: Thread Local Allocation Buffer 本地线程分配缓冲：每个线程在堆中(Ede)预先分配一小块内存。

**6. 使用 Using**

**7. 卸载 Unloading**





### ClassLoader

**loadClass()**

- public方法

- 执行父加载器逻辑

```java
Class<?> c = findLoadedClass(name);

if (parent != null) {
    c = parent.loadClass(name);
}
// 如果父加载器没加载成功，调用自己的 findClass 去加载
if (c == null) {
    c = findClass(name);
}
return c；
```




**findClass(String name)**

- 负责找到.class文件，解析成byte[]

```java
protected Class<?> findClass(String name){
  //1. 根据传入的类名 name，到在特定目录下去寻找类文件，把.class 文件读入内存
  ...         

  //2. 调用 defineClass 将字节数组转成 Class 对象
  return defineClass(buf, off, len)；
}
```



**defineClass()**

- 负责将byte[] 解析成Class对象

```java
// 将字节码数组解析成一个 Class 对象，用 native 方法实现
protected final Class<?> defineClass(
  byte[] b, 
  int off, 
  int len) {
       ...
    }
}
```

###### 

**异常**

- **ClassNotFoundException**
  - 当 “动态加载” Class 的时候找不到类会抛出该异常
  - 一般在执行Class.forName()、ClassLoader.loadClass()或ClassLoader.findSystemClass()的时候抛出

- **NoClassDefFoundError**
  - 编译成功以后，“执行”过程中Class找不到
  - 由JVM的运行时系统抛出



## 方法调用

**变量类型**

- 静态类型 Father f =
- 实际类型 f = new Son()

**静态分派**

- 依赖静态类型来定位方法
- 典型应用
  - 方法重载

**动态分派**

- 依赖实际类型来定位方法

- 典型应用

  - 方法重写

  

## 编译

将字节码翻译成机器码

**编译器**

- 解释执行

- JIT (Just in Time Compilation)
  - 即时编译，在运行时将热点代码编译成机器码
  - 如何探测热点代码？
    - 方法调用计数器 Invocation Counter
    - 回边计数器 Back Edge Counter：循环体代码的执行次数

- AOT (Ahead of Time Compilation) 
  - 避免JIT预热开销



**编译优化**

- 方法内联

  > -XX:MaxFreqInlineSize
  >
  > -XX:MaxInlineSize

- 逃逸分析 Escape Analysis
  - 判断对象是否被外部方法引用，或外部线程访问；如果只在一个方法中使用，则在栈上创建对象。
  - 锁消除
  - 标量替换：对象不被外部访问，且可被拆分时，则直接创建他的成员变量，而不创建对象



**参数**

- -Xint: 仅解释执行
- -Xcomp: 关闭解释器，最大优化级别；会导致启动变慢



## 字节码指令

- Opcode 操作码  + Operands 操作数

- load 加载指令：将一个局部变量加载到操作栈

- store 存储指令：将一个数值从操作数栈 存储到局部变量表

- 运算指令：iadd, isub, ladd, lsub

- 类型转换指令：i2b, d2f

- 对象创建与访问指令：new, newarray, getfield

- 操作数栈管理指令：pop, swap

- 控制转移指令

- 方法调用和返回指令
  - invokestatic：调用静态方法
  - invokespecial：调用 实例构造器、私有方法、父类方法
  - invokevirtual：调用所有的虚方法
  - invokeinterface：调用接口方法
  - invokedynamic：在运行时动态解析出调用点限定符所引用的方法
  - ireturn

- 异常处理指令：athrow

- 同步指令：monitorenter, monitorexit



# | GC

## || 引用类型



**强引用 Strong Reference**



**软引用 SoftReference**

- 发生内存溢出之前，会尝试回收
- 应用场景：可用于实现内存敏感的缓存



**弱引用 WeakReference**

- 下一次垃圾回收之前，一定会被回收

  > 只要GC，就会被回收？
  >
  > 额外条件：没有被其他强引用引用

- 可用来构建一种没有特定约束的关系，例如维护非强制性的映射关系

- 应用场景：缓存

  - vs. SoftReference ???
  - ThreadLocalMap.Entry

  

**虚引用 PhantomReference** 

- 不能通过它访问对象

  - 必须配合ReferenceQueue使用

  - 被 GC 的对象会被丢进这个 ReferenceQueue 里面。

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

    

- 虚引用仅提供一种确保对象被finalize后做某些事的机制；
- finalize 之后会变成虚引用
- 应用场景：跟踪对象被垃圾回收器回收的活动



**实践**

- -XX:+PrintReferenceGC 打印各种引用数量

- Reference.reachabilityFence(this)

  > 声明对象强可达 
  >
  > ```java
  > class Resource {
  > 
  >  public void action() {
  >  try {
  >      // 需要被保护的代码
  >      int i = myIndex;
  >      Resource.update(externalResourceArray[i]);
  >  } finally {
  >      // 调用 reachbilityFence，明确保障对象 strongly reachable
  >      Reference.reachabilityFence(this);
  >  }
  >  
  >  
  >  // 调用
  >  new Resource().action();
  > 
  > ```



## || GC回收判断

**引用计数法**

无法解决循环引用问题



**可达性分析**

GC Roots

- 虚拟机栈中，本地变量表引用的对象
- 本地方法栈中，JNI引用的对象
- 方法区中，类静态属性引用的对象
- 方法区中，常量引用的对象
- 被同步锁持有的对象 -?



## || GC算法

### 标记-清除 Mark-Sweep

**思想**

- 标记出所有需要回收的对象，标记完成后统一回收被标记的对象
- 关注延迟

**问题**

- 效率问题：如果大量对象需要回收，则要大量标记和清除
- 空间问题：内存碎片化



### 标记-复制 Copying

**思想**

- 将可用内存划分为两块；
- 当一块用完时，将存活对象复制到另一块内存

**问题**

- 内存空间浪费
- 优化：Eden : Survivor = 8:1，只浪费10%

**适用**

- 存活率低的场景



### 标记-整理 Mark-Compact

**思想**

- 标记出所有需要回收的对象，让所有存活对象都向一端移动
- 关注吞吐量

**问题**

- 要移动内存，回收时更复杂

  > 对比标记清除：不移动内存，分配时更复杂

- 要移动内存，停顿时间会更长

  > 对比标记清除：停顿时间短，但吞吐量低

**例子：老年代**



## || 实现细节

### 枚举根节点

- 会发生停顿 STW

- OopMap: 虚拟机知道哪些地方存放着对象引用，无需真正遍历方法区等

### 安全点

- 程序执行时，只有达到安全点时才能暂停：

- 种类
  - 方法调用
  - 循环跳转
  - 异常跳转



如何让所有线程都跑到最近的安全点停顿下来？

- 抢先式中断
  - 先中断全部线程，如果发现有线程中断的地方不在安全点上，则恢复线程
  - 少用

- 主动式中断
  - GC简单地设置一个标志
  - 线程执行时主动轮询这个标志，当为真时则在最近的安全点中断挂起



**安全区域**

目的

- 解决安全点的问题：程序不执行时（sleep/blocked），无法检查安全点

手段

- 在一段代码片段中，引用关系不会发生变化，在这个区域中任意地方开始GC都是安全的。
- 线程进入安全区域时：标记已进入
  - 此时VM发起垃圾收集时不用管这些已声明自己在安全区域的线程

- 线程离开安全区域时：检查VM是否完成根节点枚举(或其他需要STW的操作)
  - 若未完成，则等待

### 跨代引用的问题

问题

- 老年代会引用新生代，要把整个老年代加入到 GC Root 吗？

解决

- **Remembered Set**
  - 在新生代上的一个全局数据结构，标识老年代的“哪一块内存”存在跨代引用
  - MinorGC 时只需要将这些内存放入GC Root，而无需扫描整个老年代：缩减GC Root扫描范围
  - 代价
    - 改变引用关系时，需要维护 Remembered Set
    - 通过“写屏障 Write Barrier”，类似AOP，在每个赋值操作之后维护卡表

- 实现方式

  - 字长精度：精确记录一个机器字长，该字包含跨代指针

  - 对象精度：精确记录到一个对象，该对象中有字段含有跨代指针

  - 卡精度：精确到一块内存区域（Card Page），该区域内有对象含有跨代指针

    > Card Table
    >
    > 粒度更粗犷

### STW

STW 的场景

- **扫描 GC Root**

  - 优化耗时：OopMap，避免检查所有执行上下文和全局引用位置 

- **从 GC Root 往下遍历对象图** 

  - 问题：对象消失

    - 两个条件同时满足时，会回收掉不该回收的对象

      > 赋值器“插入”了从<黑色对象>到<白色对象>的新引用；
      >
      > 赋值器“删除”了全部从<灰色对象>到该<白色对象>的引用；

  - 解决

    - 增量更新 Incremental Update

      > <黑色对象>一旦新插入指向<白色对象>的引用之后，它就变回 <灰色对象>；
      >
      > 记录插入的新引用；当并发扫描结束后 再将记录的黑色对象为根，重新扫描；
      >
      > CMS

    - 原始快照 SATB, Snapshot At The Beginning

      > 记录删除的<灰色对象>到<白色对象>的引用；当并发扫描结束后 再将记录的灰色对象为根，重新扫描 -- 灰色对象出来的引用已被删除，重新扫描何用？？？
      >
      > G1、Shenandoah

- 

##### 内存分配策略

###### 对象优先在Eden分配

####### 如果Eden空间不足，则触发 MinorGC

###### 大对象直接进入老年代

####### PretenureSizeThreshold

####### 目的：避免在Eden和Survivor之间来回复制大对象

###### 长期存活的对象进入老年代

####### MaxTenuringThreshold

###### 动态对象年龄判断

####### 若 Survivor中相同年龄对象的总大小 > Survivor的一半；则该年龄及更老的对象 直接进入老年代

###### 空间分配担保

####### MinorGC之前，检查老年代最大可用的连续空间，是否大于新生代所有对象总空间。

####### 如果大于，则MinorGC是安全的。

####### 如果小于，且老年代最大可用连续空间 > 历次晋升到老年代对象的平均大小；则尝试 MinorGC

####### 否则要进行一次FullGCC.

### 垃圾收集器

#### Young

##### Serial

-XX:+UseSerialGC



###### 实现

####### 收集器是单线程的

####### 收集过程：Stop The World

###### 算法

####### 新生代：标记-复制算法

####### 老年代

######## Serial Old

###### 场景

####### 单线程，简单高效，适用于Client模式

##### ParNew

###### 实现

####### 新生代：多个GC线程

######## 是Serial的并行版本

######## 收集线程数 默认 == CPU核心数

可通过 -XX:ParallelGCThreads 进行xian'zhi

####### 其余和Serial一致，STW

###### 算法

####### 新生代：标记-复制算法

####### 老年代

######## CMS!!!

###### 场景

####### 常配合老年代的CMS

##### Parallel Scavenge

###### 算法

####### 新生代：标记-复制算法

####### 老年代：标记整理

######## Parallel Old

###### 场景

####### 吞吐量优先

######## 而其他算法关注缩短停顿时间

####### 适合后台运算任务，不需要太多的交互

###### 特点

####### 可设置 MaxGCPauseMillis 调整暂停时间, GCTimeRatio 调整吞吐量

######## 如调小GC停顿时间，会牺牲吞吐量和新生代空间

####### 可设置自适应策略 -XX:+UseAdaptiveSizePolicy

####### 其余类似ParNew

#### Tenured

##### Serial Old

###### 实现

####### 单线程

####### Serial 的老年代版本

###### 场景

####### CMS 发生失败时的后备预案

##### Parallel Old

###### 实现

####### 标记-整理

####### Parallel Scavenge的老年代版本

###### 场景

####### 吞吐量优先

##### CMS, 
Concurrent Mark Sweep

###### 步骤

####### 1.初始标记 (单线程，STW)
Initial Mark

######## 仅标记 GC Roots 直接关联的对象

######### 速度很快

######## STW

####### 2.并发标记 (单线程，并发)
Concurrent Mark

######## GC Roots Tracing 可达性分析，扫描整个堆里的对象图

########  耗时长，但与用户线程一起工作

####### 3.重新标记 (多线程，STW)
Remark

######## 修正上一步并发标记期间发生变动的对象

######### 基于“增量更新”

######## 多线程！

######## STW

####### 4.并发清除 (单线程，并发)
Concurrent Sweep

########  耗时长，但与用户线程一起工作

###### 并发

####### 并发体现在，清理工作与工作线程一起并发运行

####### 减少回收停顿时间

###### 算法

####### 标记清除算法（其他收集器用标记整理）

####### 会产生内存碎片

###### 缺点

####### CPU资源敏感，吞吐量会降低

######## 并发阶段会占用线程

######## 导致应用程序变慢，吞吐量降低

####### 无法处理浮动垃圾，可能 Concurrent Mode Failure

浮动垃圾：
并发标记、并发清理阶段，用户线程继续运行，会有新的垃圾对象不断产生。

`CMSInitiatingOccupancyFraction` == 触发CMS的启动阈值；不宜过高

######## 不能等到老年代几乎满时再收集，要预留空间给浮动垃圾

######## 若预留的内存无法满足程序需要，出现Concurrent Mode Failure

######## 此时会 临时启用Serial Old收集，停顿时间长

####### 内存碎片

######## 会给大对象分配带来麻烦

######## 缓解：提前触发FullGC时 开启内存碎片的合并整理

#### Mixed

##### G1

###### Mix GC 步骤

####### 1.初始标记 (单线程，STW)
Initial Marking

######## 仅标记 GC Roots 直接关联的对象；并修改TAMS指针值

TAMS: Top at Mark Start，并发回收时新分配的对象会在TAMS指针位置之上；
G1收集器默认TAMS之上的对象是被标记过的，不会对其进行回收

####### 2.并发标记 (单线程，并发)
Concurrent Marking

- 耗时长，但与用户线程一起工作
- GC Roots Tracing 可达性分析

######## GC Roots Tracing 可达性分析，扫描整个堆里的对象图

####### 3.最终标记 (多线程，STW)
Final Marking

######## 处理并发标记结束后遗留下来的SATB记录

SATB: 原始快照算法，Snapshot At the Beginning

####### 4.筛选回收 (多线程，STW)
Live Data Counting and Evacuation

######## 对Region的回收价值进行排序，把需要回收的Region内的存活对象复制到空Region

###### 算法

####### 新生代：并行的复制算法，STW

######## 其实不区分新生代老年代？？？

####### 老年代：并发标记、整理则是增量的，并且是新生代GC是顺带进行

####### 从整体上看：标记 - 整理

####### 从局部看：两个Region之间 标记 -复制

###### 场景

####### 兼顾吞吐量和停顿时间

###### 特点

####### 并行与并发

######## 充分利用多CPU, 多核

####### 空间整合 

######## -标记整理

####### 可预测的停顿

######## 停顿时间模型

######### 允许使用者明确指定M毫秒内GC时间不得超过N毫秒

######### 原理：记录每个Region的回收耗时、RS里的脏卡数量等

######## 将 Region 作为单次回收的最小单元

######### 避免全区域GC

###### 概念

####### Region 分代收集

######## G1将堆划分为大小相等的`Region`，新生代和老年代不再是物理隔离的

######## Eden、Survivor、Old、Humongous

######### 超过Region 50%的对象

######## 每个Region大小一致，为1M~32M之间的某个2次幂

######## 缺点：有空间浪费

####### Remembered Set

######## 用来记录region之间对象的引用关系

######## GC时保证老年代到新生代的跨区引用仍然有效

###### vs. CMS

####### G1包括Young GC 和 Mix GC；CMS 集中在老年代回收

####### G1 减少垃圾碎片产生

######## 用region方式划分堆内存

######## 可以做到局部区域的回收，不必每次都回收整个年轻代/年老代

######## 基于标记整理算法

####### 初始化标记阶段，搜索可达对象用到Card Table，实现方式不一样

####### G1 可指定期望停顿时间

####### 弱点

######## 内存占用较高

######### 每个Region都要有一份卡表

######## 执行负载较高

######### 卡表维护更繁琐

######### 原始快照搜索算法，比起增量更新算法的额外负担

#### 低延迟收集器

##### Shenandoah

https://shipilev.net/talks/devoxx-Nov2017-shenandoah.pdf

###### 九个步骤

####### 1.初始标记 (STW)
Initial Marking

######## 仅标记 GC Roots 直接关联的对象

TAMS: Top at Mark Start，并发回收时新分配的对象会在TAMS指针位置之上；
G1收集器默认TAMS之上的对象是被标记过的，不会对其进行回收

######## 同G1

####### 2.并发标记 (并发)
Concurrent Marking

- 耗时长，但与用户线程一起工作
- GC Roots Tracing 可达性分析

######## GC Roots Tracing 可达性分析，扫描整个堆里的对象图

######## 同G1

####### 3.最终标记 (STW)
Final Marking

######## 处理并发标记结束后遗留下来的SATB记录

SATB: 原始快照算法，Snapshot At the Beginning

######### 同G1

######## 同时统计回收价值最高的Region，组成“回收集”

####### 4.并发清理
Concurrent Cleanup

######## 清理一个存活对象都没有的Region

####### 5.并发回收
Cocurrent Evacuation

######## 将回收集里的存活对象复制到未被使用的Region中

######## 通过“转发指针”解决并发问题

####### 6.初始引用更新 (STW)
Initial Update Reference

######## 引用更新：把堆中所有指向旧对象的引用修正到复制后的新地址

######## 初始化阶段：确保所有收集器线程都已完成对象移动任务

######### 短时 STW

####### 7.并发引用更新
Concurrent Update Reference

######## 真正开始引用更新

####### 8.最终引用更新 (STW)
Final Update Reference

######## 修正GC Roots中的引用

######### 短时 STW

####### 9.并发清理
Concurrent Cleanup

######## 回收 “回收集”里的Region

###### vs. G1

####### 支持并发的整理算法

####### 默认不使用分代收集

######## 没有专门的新生代、老年代Region

####### 抛弃记忆集，改用连接矩阵（Connection Matrix）

######## 二维表格

##### ZGC

###### 特点

####### 内存布局

######## Region具有动态性

######### 小型 Region

########## 容量 2M，用于存放 <256K 的对象

######### 中型 Region

########## 容量 32M，用于存放 256K~4M 的对象

######### 大型 Region

########## 容量为 N*2M，用于存放 > 4M的单个对象

####### 并发整理算法

######## 染色指针技术

######### 直接把标记信息记在引用对象的指针上

######## 全程可并发

###### 步骤

####### 1. 并发标记
Concurrent Mark

######## 遍历对象图做可达性分析；标记指针，而不是对象

####### 2. 并发预备重分配
Concurrent Prepare for Relocate

######## 统计要清理的Region，记录到重分配集（Relocation Set）

####### 3. 并发重分配
Concurrent Relocate

######## 将重分配集里的存活对象复制到新Region；

######## 并为重分配集中的每个Region维护一个转发表，记录从旧对象到新对象的转向关系

####### 4. 并发重映射
Concurrent Remap

######## 修正整个堆中指向重分配集旧对象的所有引用

######## 这一步会合并到下一次并发标记阶段

## 问题排查

### GC

#### Young GC耗时

##### 新生代太大

##### -XX:G1NewSizePercent
-XX:G1MaxNewSizePercent

#### MinorGC 频繁

##### 解决

###### 增大新生代空间

##### 增大新生代不会增加单次GC的耗时

###### 复制 远比 扫描 耗时

###### 只要GC后存活对象数量不多即可

###### 适用于长期存活对象少的场景

#### FullGC 频繁

##### 监控

###### jstack

####### CPU飚高，jstack看到主要是垃圾回收线程

###### jstat

####### jstat 监控GC，看到FullGC非常多，并在增加

##### 验证步骤

###### top

####### 查看CPU占用情况，找到 进程PID

###### top -Hp PID

####### 查看进程内各线程运行情况

####### 找到线程ID，转换为16进制

######## printf "%x" 10

###### jstack

####### 按线程ID查找，确认是VM Thread，GC线程

###### jstat -gcutil PID <interval> <count>

####### 确认GC次数在增加

###### dump分析

####### 找到哪个对象消耗内存

####### 或者是否显式System.gc()调用？

######## 直接搜索dump文件

##### 解决思路

###### 保持老年代的相对稳定

###### 不可有大量、长生存时间的大对象

##### 解决手段

###### 池化

###### 减少创建大对象

####### 因为 大对象直接创建在老年代

###### 增大堆内存

####### 每次要回收的空间岂不更多？？？应该用小内存机器 + 集群部署？

###### -Xms == -Xmx
-XX:PermSize == -XX:MaxPermSize

####### 避免运行时自动扩容

####### 何时用？

######## 看GC日志，老年代总容量不断增加、且每次gc回收掉的内存过少时

###### -XX:+DisableExplicitGC

####### 忽略掉 System.gc()

###### 选择低延迟收集器

### 运行缓慢

#### 可能原因

##### 代码有阻塞性操作？

##### 线程停顿

###### 等待外部资源：数据库网络、网络

###### 死循环

###### GC

####### GC 本身耗时

####### 或者等待线程达到安全点耗时

######## 例如 索引 int 循环 ：可数循环，不会插入安全点

###### 锁等待

####### BLOCKED

###### 线程 Wait?

####### WAITING

##### 类加载时间长

###### 排查

####### jstat -class xx

###### 解决

####### -Xverify:none 禁止字节码验证环节

##### JIT 编译时间长

###### 问题

####### JIT 动态编译热点代码，能让程序越运行越快；但需要消耗资源，影响运行时间

###### 解决

####### -Xinit 禁止编译器运作

######## 慎用

#### 解决思路

##### 压测

##### jstack生成一系列线程转储日志

##### 分析线程转储日志，找WAITING、找deadlock

### 进程崩溃

#### 可能原因

##### 等待的线程 和 Socket连接过多，超过虚拟机承受能力

### CPU 飙高

#### 可能原因

##### Runtime.getRuntime.exec() 创建进程开销大

#### 解决

##### 利用 Java API 替代 runtime.exec()

### 堆外内存

#### 表现

##### -XX:+HeapDumpOnOutOfMemoryError 无效，OOM时不会产生dump文件

##### 基于监控发现问题：Memory - Heap Size

##### 堆外内存不足时 也很难触发GC

###### 只能等到老年代满后 FullGC时，“顺便”清理堆外内存

#### 排查

##### pmap

###### 查看物理内存分配

##### GDB

##### NMT

###### Native Memory Tracking

###### jcmd xx VM.native_memory summary.diff

#### 解决

##### -XX:MaxDirectMemorySize 调大

### 内存溢出、泄漏

#### 分析步骤

同样适用于 CPU 过高的场景

##### top

按 P: 按CPU排序；
按 M: 按内存排序；

###### 找到耗内存的pid

##### top -Hp pid

###### 查看线程占用资源

###### 输入 P：按CPU占用排序

###### 输入 M: 按内存排序

##### jstack pid

###### 查看线程堆栈，排除IO阻塞，死锁

##### jmap -heap pid

###### 查看堆内存占用

##### dump分析

###### Histogram

###### with incomming reference

#### 提示信息

##### OOM: Java heap space

###### 内存泄露

###### 堆过小

###### finalize() 滥用


定义finalize后，GC不会立即回收这些实例，而是将实例添加到一个 ReferenceQueue队列中。等finalize执行之后才回收。

队列中对象过多会导致OOM

##### OOM: GC overhead limit exceeded

###### 原因：当GC花费 98%的CPU，但只回收了 2% 堆空间，并连续5次。

###### 内存泄露

###### 堆过小

##### OOM: Requested array size exceeds VM limit

###### 创建了过大的数组

###### 堆过小

##### OOM: MetaSpace

###### MaxMetaSpaceSize过小

##### OOM: Unable to create native threads

###### 创建线程池失败

###### -Xss 每个线程栈空间过大

###### 操作系统限制

- ulimit 限制 (max user processes)
- sys.kernel.threads-max 限制
- sys.kernel.pid_max 限制

#### 编程问题

https://www.baeldung.com/java-memory-leaks

##### 滥用 大 static 字段

###### static 字段生命周期长

###### 除非应用停止，或对应的ClassLoder被GC

##### 使用资源未关闭

##### equals / hashCode 实现有误，导致大量重复key 加到集合里

##### 内部类引用外部类

###### 阻止外部类被GC

##### 滥用 finalize() 方法

###### 被GC之前会被放到队列里，不能及时GC

##### Interned String

###### intern() 后放在 永久代，生命周期长

##### ThreadLocal

https://www.programmersought.com/article/2886751515/

https://www.jianshu.com/p/1342a879f523

Thread引用 --> Thread --> ThreadLocalMap --> Entry --> value 泄漏

###### 线程池中的线程生命周期长，导致对应的ThreadLocal生命周期也长

###### ThreadLocal 本身是 弱引用，如果没有其他引用则可以被回收

###### 但 value 本身不会回收，只要 线程 仍然存活

###### 手段

####### remove()

####### 不可用set(null)

Do not use ThreadLocal.set(null) to clear the value — it doesn't actually clear the value but will instead look up the Map associated with the current thread and set the key-value pair as the current thread and null respectively.
https://www.baeldung.com/java-memory-leaks


####### 保护功能：get / set / remove 时都会清除 key==null的值

##### 线程池

###### 使用无界队列

###### 没有限制最大线程数量

##### tomcat 配置问题

###### tomcat maxHttpHeaderSize 过大，导致每个tomcat工作线程过大

##### WeakHashMap 也可能不被回收！

###### 例如 Value 引用了Key

## 工具

### 系统工具

#### top

##### CPU,内存使用率

##### top -Hp pid ：查看具体线程

##### 交互

###### 1

####### 查看分CPU统计

###### H

####### 查看每个线程

#### vmstat

procs
r：等待运行的进程数
b：处于非中断睡眠状态的进程数

memory
swpd：虚拟内存使用情况
free：空闲的内存
buff：用来作为缓冲的内存数
cache：缓存大小

swap
si：从磁盘交换到内存的交换页数量
so：从内存交换到磁盘的交换页数量

io
bi：发送到快设备的块数
bo：从块设备接收到的块数

system
in：每秒中断数
cs：每秒上下文切换次数

cpu
us：用户 CPU 使用事件
sy：内核 CPU 系统使用时间
id：空闲时间
wa：等待 I/O 时间
st：运行虚拟机窃取的时间

##### 监控进程上下文切换

#### pidstat

pidstat -p pid -t 

-p：指定进程号；
-u：默认参数，显示各个进程的 cpu 使用情况；
-r：显示各个进程的内存使用情况；
-d：显示各个进程的 I/O 使用情况；
-w：显示每个进程的上下文切换情况；
-t：显示进程中线程的统计信息

cswch/s：每秒主动任务上下文切换数量
nvcswch/s：每秒被动任务上下文切换数量


##### 监控线程上下文切换

#### lsof -i:8080

##### 端口查询

### JDK工具

#### jps -v

-m: 输出main() 参数；
-l: 输出主类全名；
-v: 输出JVM参数；


##### 查看 java 进程列表

#### jinfo

修改参数：
- jinfo -flag name=value
- jinfo -flag[+|-] name

##### 查看修改VM参数

#### jstat

jstat -gc vmid [interval] [count]

jstat -options
- class
- gc
- gcutil: 关注百分比
- gccause

gc字段含义：

`S0C`：年轻代中 To Survivor 的容量（单位 KB）；
`S1C`：年轻代中 From Survivor 的容量（单位 KB）；
`S0U`：年轻代中 To Survivor 目前已使用空间（单位 KB）；
`S1U`：年轻代中 From Survivor 目前已使用空间（单位 KB）；
`EC`：年轻代中 Eden 的容量（单位 KB）；
`EU`：年轻代中 Eden 目前已使用空间（单位 KB）；
`OC`：Old 代的容量（单位 KB）；
`OU`：Old 代目前已使用空间（单位 KB）；
`MC`：Metaspace 的容量（单位 KB）；
`MU`：Metaspace 目前已使用空间（单位 KB）；
`YGC`：从应用程序启动到采样时年轻代中 gc 次数；
`YGCT`：从应用程序启动到采样时年轻代中 gc 所用时间 (s)；
`FGC`：从应用程序启动到采样时 old 代（全 gc）gc 次数；
`FGCT`：从应用程序启动到采样时 old 代（全 gc）gc 所用时间 (s)；
`GCT`：从应用程序启动到采样时 gc 用的总时间 (s)。

##### 用于查看分区内存

###### 命令行版 JConsole

##### -class

##### -gccause

###### 查看最近一次gc的原因

##### -gc / -gcutil

###### jstat -gcutil PID <interval> <count>

###### 定时输出各空间大小

#### jmap

-dump: 生成堆转储快照；
-F: 强制生成dump快照；
-heap: 显示堆详细信息；
-histo: 显示堆对象统计信息；


##### jmap -dump:live

jmap -dump:live,format=b,file=/pang/logs/tomcat/heapdump.bin 1

###### 生成heap dump

###### :live 会触发FullGC，只转储活着的对象

##### jmap -heap <PID> 

###### 查看堆内存使用情况

#### jhat

jhat xx.hprof

```
jhat heapdump.hprof
Reading from heapdump.hprof...
Dump file created Wed Aug 05 20:22:47 CST 2020
Snapshot read, resolving...
Resolving 239374 objects...
Chasing references, expect 47 dots...............................................
Eliminating duplicate references...............................................
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.

http://localhost:7000/
```

##### 简单分析 heap dump 

#### jstack -l

-l: 包含锁信息；
-m: 包含本地方法堆栈；

##### 生成 thread dump

##### 分析死锁：找到BLOCKED的线程

##### 可通过编程方式：Thread.getAllStackTraces()

#### jcmd

`jcmd 1 GC.heap_dump ${dump_file_name}`

`jcmd 1 help`

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



https://www.javacodegeeks.com/2016/03/jcmd-one-jdk-command-line-tool-rule.html

##### heap dump

jcmd <process id/main class> GC.heap_dump filename=Myheapdump

##### thread dump

jcmd 17264 Thread.print


##### jcmd 11 VM.native_memory baseline
jcmd 11 VM.native_memory summary.diff

#### javap

`javap -verbose TestClass`

##### 分析class文件字节码

### GUI工具

#### JConsole

##### 内存：jstat

##### 线程：jstack

##### 查看 MBean

##### 查看分区内存

#### VisualVM

##### 插件

##### 监视：dump

##### Profiler

###### CPU

###### 内存

##### BTrace

###### 动态插入调试代码

###### 原理：Java Instrument，Arthas也是利用这个原理

#### JMC：Java Mission Control

https://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html

##### server 需要添加启动参数

-Dcom.sun.management.jmxremote=true
-Djava.rmi.server.hostname=127.0.0.1
-Dcom.sun.management.jmxremote.port=6666
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.managementote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false
-XX:+UnlockCommercialFeatures
-XX:+FlightRecorder

##### 原理：JFR

###### 运行时启动JFR记录

Jcmd <pid> JFR.start duration=120s filename=myrecording.jfr


#### GC

##### GC Viewer

https://sourceforge.net/projects/gcviewer/

##### https://www.gceasy.io

### instrument

#### ClassFileTransformer

#### Instrumentation

## 参数

### 运维

#### -XX:+HeapDumpOnOutOfMemoryError

#### -Xverify:none -忽略字节码校验

#### -Xint -禁止JIT编译器

### 基本

#### -Xms 
-Xmx

##### 最小堆，最大堆

##### 建议设置成一样大

###### 避免GC后调整堆大小带来压力

#### -Xmn

##### 新生代

#### -Xss

##### 栈内存大小

### 内存

#### -XX:PermSize  
-XX:MaxPermSize

#### -XX:MetaspaceSize
-XX:MaxMetaspaceSize

##### 元空间初始大小，若超过 则触发GC进行类型卸载

##### 同时收集器会根据GC后的空间释放情况，动态调整 MetaspaceSize

#### -XX:MaxDirectMemorySize 

##### 直接内存

#### -XX: NewRatio

##### 老年代新生代比例

##### 默认2

#### -XX: SurvivorRatio

##### 8:1:1

#### -XX:MaxTenuringThreshold

##### 转移到老年代的年龄

#### -XX:PetenureSizeThreshold

##### 超过阈值则直接分配到老年代

#### -XX:+AlwaysTenure -去掉Suvivor空间

#### -XX:NativeMemoryTracking={summary|detail}

##### 堆外

### GC

#### -XX:+DisableExplicitGC

##### -忽略System.gc()

#### -XX:+PrintReferenceGC // 打印各种引用数量

#### -Xloggc:xx.log

#### 日志

##### 查看GC基本信息

###### -XX:+PrintGC

###### -Xlog:gc

####### Java9

##### 查看GC详细信息

###### -XX:+PrintGCDetails
-XX:+PrintGCDateStamps

###### -Xlog:gc*

####### Java9

##### 查看GC前后容量变化

###### -XX:+PrintHeapAtGC

###### -Xlog:gc+heap=debug

####### Java9

##### 查看GC过程中用户线程并发时间、停顿时间

###### -XX:+PrintGCApplicationConcurrentTime / -XX:+PrintGCApplicationStoppedTime

###### -Xlog:safepoint

####### Java9

##### 查看Ergonomics机制：各分代大小、收集目标

###### -XX:+PrintAdaptiveSizePolicy

####### 打印 G1 Ergonomics 相关信息

###### -Xlog:gc+ergo*=trace

####### Java9

##### 查看收集后剩余对象的年龄分布

###### -XX:+PrintTenuringDistribution

###### -Xlog:gc+age=trace

####### Java9

#### G1

##### -XX:G1NewSizePercent
-XX:G1MaxNewSizePercent

##### -XX: +G1HeapRegionSize

###### G1区域大小

### 并发

#### -XX:-UseBiasedLocking 关闭偏向锁

### 参数确定

#### 老年代大小

##### FullGC之后，存活对象大小的 1.5~2 倍