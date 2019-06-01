# Java并发

## 基础

### 并发的问题

#### 微观

##### 原子性问题：线程切换导致 （e.g. i++）

##### 可见性问题：缓存导致

##### 有序性问题：编译优化导致

#### 宏观

##### 安全性问题

###### Data Race -数据竞争

####### 同时修改共享数据

###### Race Condition -竞态条件

####### 程序的执行结果依赖线程执行的顺序

###### 解决：互斥（锁）

##### 活跃性问题

###### 概念：指某个操作无法执行下去

###### 死锁

###### 活锁

####### 张猫

####### 解决：等待随机时间

###### 饥饿

####### 线程因为无法访问所需资源而无法执行下去

####### 解决：保证资源充足、公平分配资源（公平锁）、避免持有锁的线程长时间执行

##### 性能问题

###### 串行比不能过高

###### 解决

####### 无锁

######## Thread Local Storage

######## Copy on write

######## 乐观锁

######## 原子类

####### 减少锁持有的时间

######## ConcurrentHashMap

######## 读写锁

### happens-before

#### 含义：前一个操作的结果对后一个操作是可见的。

#### 1. 程序顺序性规则

与编译器重排序 冲突？

##### 前面的操作 happens-before后续的任意操作

#### 2. volatile变量规则

##### 对volatile的写操作 happens-before后续对这个变量的读操作

#### 3. 传递性

#### 4. 管程中锁的规则

- synchronized是Java对管程的实现。

##### 对一个锁的解锁 happens-before 后续对这个锁的加锁

#### 5. 线程start()规则

##### thread-A call thread-B.start()，start()执行前A的操作 happens-before B中的任意操作

#### 6. 线程join()规则

##### thread-A call thread-B.join()，B中的任意操作 happens-before join方法操作的返回

### 死锁

#### 发生条件

##### 1. 互斥

###### 共享资源X/Y只能被一个线程占用

###### 避免：不可避免

##### 2. 占有且等待

###### 线程取得X的锁后等待Y的过程中，不会释放X的锁

###### 避免：一次性申请所有资源

##### 3. 不可抢占

###### 占有的资源不可被其他线程抢占

###### 避免：如果等待不到，则主动释放已占有的锁 （Lock  timeout）

if (lock.tryLock() || lock.tryLock(timeout, unit))



##### 4. 循环等待

###### 互相等待

###### 避免：按照资源序号申请

#### 避免死锁

##### 其实就是破坏发生条件

##### 避免使用多个锁；设计要锁的获取顺序

### Thread

#### 线程状态

##### 初始

###### New

##### 可运行

###### Runnable

##### 运行

###### Runnable

##### 休眠

###### Blocked

####### 等待锁

###### Waiting / Timed_Waiting

####### wait()

####### join()

####### LockSupport.park() /  parkNanos/parkUntil

####### sleep(ms)

##### 终止

###### Terminated

####### 线程执行结束

####### 线程异常

######## interrupt()

######### 异常

########## 如果处于Waiting/Timed_Waiting，interrupt()则会触发InterruptedException

所以 wait(), join(), sleep()方法声明都有throws InterruptedException

########## 如果处于Runnable状态，并阻塞在InterruptibleChannel上；则会触发ClosedByInterruptException

########## 如果处于Runnable状态，并阻塞在Selector上；则Selector会立即返回

######### 主动监测

########## if isInterrupted()

######## 不可用stop()!! 其不会释放锁

### AQS

AbstractQueuedSynchronizer


#### 内部实现

##### volatile int state

###### 表示状态

##### FIFO 队列

###### 实现多线程建竞争和等待

##### 各种基于CAS的操作方法，以及抽象acquire() / release()

## 分工

### Executor / 线程池

#### execute() vs. submit()

##### execute不需要返回值，无法判断任务是否执行成功

##### submit 返回 Future

#### ThreadPoolExecutor

##### corePoolSize

###### 最小线程数

##### maximumPoolSize

###### 最大线程数

##### keepAliveTime & unit

###### 判断是否空闲的依据

##### workQueue

###### 工作队列

###### 慎用无界队列，避免OOM

##### threadFactory

##### handler: 拒绝策略

###### CallerRunsPolicy

####### 提交任务的线程去执行

###### AbortPolicy

####### 默认值，抛出 RejectedExecutionException

####### 慎用，因为并不强制要catch exception，很容易被忽略

####### 一般与降级策略配合

###### DiscardOldestPolicy

####### 丢弃最老的任务

#### Executors创建线程池

##### newCachedThreadPool()

##### newFixedThreadPool()

##### newSingleThreadExecutor()

##### newSingleThreadScheduledExecutor()

##### newWorkStealingPool()

#### 线程池大小设置

##### CPU密集型任务：线程池尽可能小

###### CPU核数 + 1 

##### IO密集型任务：线程池尽可能大

###### CPU核数 * (1 + IO耗时/CPU耗时 )

###### CPU核数 * (1 + 等待时间/工作时间 )

#### 与一般池化资源的区别

##### 本质是 生产者-消费者 模式

###### 生产者：线程使用方

###### 消费者：线程池本身

##### 并非 acquire / release

### Future

#### 作用：类比提货单

#### ThreadPoolExecutor.submit(xx)

##### Runnable

##### Callable

##### Runnable, T

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

#### 方法

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


##### cancel

##### isCancelled

##### isDone

##### get, get(timeout)

#### FutureTask 工具类

##### 实现接口

###### Runnable

####### 所以可以将FutureTask提交给Executor去执行

###### Future

####### 所以可以获取任务的执行结果

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

##### RunnableAdapter

###### 创建：Executors#callable(Runnable, T)

###### 原理：适配器模式

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

### CompletableFuture

#### 创建

##### = Future + 回调

##### 四个静态方法

###### runAsync(Runnable, [Executor])

###### supplyAsync(Supplier, [Executor])

##### 默认使用公共的ForkJoinPool线程池

###### 默认线程数 == CPU核数

###### 强烈建议不同业务用不同线程池

#### 实现接口

##### Future

###### 解决两个问题

####### 异步操作什么时候结束

####### 如何获取异步操作的执行结果

######## get()方法

######## join()方法

##### CompletionStage

###### 描述串行关系

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

####### thenApply(Function..), xxAsync

######## 返回值CompletionStage<R>

######## 转换，类似map

####### thenAccept(Consumer..), xxAsync

######## 返回值CompletionStage<Void>

####### thenRun(Runnable..), xxAsync

######## 返回值CompletionStage<Void>

####### thenCompose(Function..), xxAsync

######## 返回值CompletionStage<R>

######## 新创建一个子流程，最终结果和 thenApply 相同

###### 描述 AND 聚合关系

####### thenCombine(other, Function), xxAsync

######## R

####### thenAcceptBoth(other, Consumer), xxAsync

######## Void

####### runAfterBoth(other, Runnable), xxAsync

######## Void

###### 描述 OR 聚合关系

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

####### applyToEither(other, Function), xxAsync

######## R

####### acceptEither(other, Consumer), xxAsync

######## Void

####### runAfterEither(other, Runnable), xxAsync

######## Void

###### 异常处理

```java
CompletableFuture<Integer> f0 = CompletableFuture
    .supplyAsync(()->7/0))
    .thenApply(r->r*10)
    .exceptionally(e->0);
    
System.out.println(f0.join());

```

####### exceptionally(Function)

######## 类似catch

####### whenComplete(Consumer), xxAsync

######## 类似finally

######## 不支持返回结果

####### handle(Function), xxAsync

######## 类似finally

######## 支持返回结果

#### 扩展

##### RxJava Observable继承了CompletableFuture的概念，用来处理数据流

### CompletionService

#### 线程池 + Future任务 + 阻塞队列

CompletionService 内部维护了一个阻塞队列，当任务执行结束就把任务的执行结果加Future加入到阻塞队列中

#### 示例

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

#### 创建

##### ExecutorCompletionService(Executor)

###### 默认用无界 LinkedBlockingQueue

##### ExecutorCompletionService(Executor, BlockingQueue)

#### 方法

##### submit(Callable) / submit(Runnable, V)

##### take()

###### 如果队列为空，则阻塞

##### poll()

###### 如果队列为空，则返回null

### Fork / Join

#### 相当于单机版的MapReduce

#### 分治任务模型

##### 任务分解：Fork

##### 结果合并：Join

#### ForkJoin 计算框架

##### 线程池 ForkJoinPool

###### 方法

####### invoke(ForkJoinTask)

####### submit(Callable)

###### 原理

####### ForJoinPool内部有多个任务队列

######## （而ThreadPoolExecutor只有一个队列）

####### 任务窃取

######## 双端队列

##### 分治任务 ForkJoinTask

###### 方法

####### fork()

######## 异步执行一个子任务

####### join()

######## 阻塞当前线程等待子任务执行结束

###### 子类

####### RecursiveAction

######## compute() 无返回值

####### RecursiveTask

######## compute() 有返回值

##### 示例

###### 计算斐波那契数列

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

###### 统计单词数量

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

## 协作

### Monitor 管程

#### 概念

##### 管程：管理共享变量、以及对共享变量的操作过程，让他们支持并发

##### synchronized, wait(), notify(), notifyAll()都是管程的组成部分

#### 实现

##### MESA模型

##### 互斥

###### 讲共享变量及其操作封装起来

##### 同步

###### 入口等待队列

###### 条件变量等待队列

####### wait(): 加入条件变量等待队列

####### notify(): 从条件变量等待队列，进入入口等待队列

### CountDownLatch

#### 场景：一个线程等待多个线程；操作的是事件

#### 计数器不能循环再利用

#### 方法

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

##### countDown()

##### await()

### CyclicBarrier

#### 场景：一组线程之间互相等待

#### 计数器会循环利用：当减到0后，会自动重置到初始值

#### 可以设置回调函数

#### 方法

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

##### await()

### Semaphore

#### 场景：限流器，不允许多于N个线程同时进入临界区

#### 方法

##### acquire()

##### release()

### Phaser

#### 类似CountDownLatch，但允许线程动态注册上来

### Exchanger

### wait / notify

#### 机制

##### 线程首先获取互斥锁；

##### 当线程要求的条件不满足时，释放互斥锁，进入等待状态；

##### 当条件满足后，通知等待的线程，重新获取互斥锁

#### 实现

##### 前提是先获取锁，所以wait/notify/notifyAll必须在synchronized内部调用

否则会IllegalMonitorStateException

##### wait 要在while循环内部调用

因为被唤醒时表示`条件曾经满足`，但wait返回时条件可能出现变化。

##### 尽量用notifyAll

###### 何时可用notify?

####### 所有等待线程拥有相同的等待条件

####### 所有等待线程被唤醒后，执行相同的操作

####### 只需要唤醒一个线程

#### wait vs. sleep

##### wait会释放锁

##### wait必须在同步块中使用

##### wait不抛异常

### Condition

#### Condition实现了管程模型里的条件变量

#### await() / signal() / signalAll()

#### 示例

##### 使用Condition实现阻塞队列

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


##### 使用Condition实现异步转同步

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

## 互斥

### 无锁

#### 不变模式

##### 对象创建后状态不再发生变化

###### 实现

####### final 类

####### final 字段

####### 所有方法均是只读的

###### 保护性拷贝

###### 避免创建重复对象：享元模式 Flyweight Pattern

####### 本质是对象池

####### Long 缓存[-127, 127]

##### 无状态

###### 没有属性，只有方法

#### 线程封闭

##### 不和其他线程共享变量

#### 线程本地存储 ThreadLocal

##### 例子

###### 线程ID生成器

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

##### 原理

###### 方案一：ThreadLocal 引用Map<Thread, T>

###### 方案二：Thread引用Map<ThreadLocal, T>

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

###### 方案二的好处

####### 更合理：所有和线程相关的数据都存储在 Thread 里面

####### 不容易内存泄露

方案一，只要ThreadLocal对象存在，那么Map中的Thread对象就永远不会回收

##### 注意

###### 线程池中使用ThreadLocal可能导致内存泄露

- 线程池中线程存活时间长；
- 所以Thread.ThreadLocalMap不会被回收；


###### 手动释放：ThreadLocal.remove()

#### CAS

##### see Atomic原子类

#### Copy-on-Write

##### 场景

###### 读多写少

###### 弱一致性

###### 对读的性能要求很高

###### 函数式编程

##### 案例

###### RPC 客户端存储 路由哈希表

###### ConcurrentHashMap<String, CopyOnWriteArraySet<Router>>

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

###### CopyOnWriteArraySet, CopyOnWriteArrayList

##### 缺点

###### 消耗内存 (并非按需复制)

#### Atomic原子类

##### 类型

###### 基本类型 

####### AtomicLong、AtomicInteger、AtomicBoolean

###### 数组类型

AtomicIntegerArray, AtomicLongArray, Atomic ReferenceArray

####### AtomicIntegerArray、AtomicReferenceArray

###### 引用类型

####### AtomicReference

######## 例如 原子性设置upper/lower

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

####### AtomicStampedReference

boolean compareAndSet(
  V expectedReference,
  V newReference,
  int expectedStamp,
  int newStamp) 


######## 解决ABA问题

######## 增加版本号stamp

####### AtomicMarkableReference

boolean compareAndSet(
  V expectedReference,
  V newReference,
  boolean expectedMark,
  boolean newMark)


######## 解决ABA问题

######## 将版本号简化为boolean

###### 对象属性修改

####### AtomicIntegerFieldUpdater

####### AtomicReferenceFieldUpdater

####### 注意

######## 对象属性必须是volatile

######## 示例

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

###### 累加器

####### LongAdder

####### LongAccumulator

####### 注意

######## 不支持compareAndSet

######## 性能更好

##### 原理

###### CAS + volatile

- 利用 `CAS` (compare and swap) + `volatile` 和 native 方法来保证原子操作，从而避免 `synchronized` 的高开销，执行效率大为提升。

- CAS的原理是拿期望的值和原本的一个值作比较，如果相同则更新成新的值。


`UnSafe.objectFieldOffset()` 是一个本地方法，用来拿到“原来的值”的内存地址，返回值是 valueOffset。
另外 value 是一个volatile变量，在内存中可见，因此 JVM 可以保证任何时刻任何线程总能拿到该变量的最新值。

####### 只有 currentValue == expectedValue，才会将其更新为newValue

####### 简单示例代码：自旋


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


####### 示例：Unsafe.getAndAddLong()

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

####### 示例：AtomicLongFieldUpdater实现锁

```java
private static final AtomicLongFieldUpdater<AtomicBTreePartition> lockFieldUpdater =
        AtomicLongFieldUpdater.newUpdater(AtomicBTreePartition.class, "lock");

private void acquireLock(){
    long t = Thread.currentThread().getId();
    while (!lockFieldUpdater.compareAndSet(this, 0L, t)){
        // 等待一会儿，数据库操作可能比较慢
         …
    }
}

public class AtomicBTreePartition {
private volatile long lock;
public void acquireLock(){}
public void releaseeLock(){}
}


```

######## Java9 优化：VarHandle

```java
private static final VarHandle HANDLE = MethodHandles.lookup().findStaticVarHandle
        (AtomicBTreePartition.class, "lock");

private void acquireLock(){
    long t = Thread.currentThread().getId();
    while (!HANDLE.compareAndSet(this, 0L, t)){
        // 等待一会儿，数据库操作可能比较慢
        …
    }
}

```

###### ABA 问题

####### AtomicStampedReference

#### 并发容器

##### 同步容器

###### Collections.synchronizedList / Set / Map

###### Vector / Stack / Hashtable

##### List

###### CopyOnWriteArrayList

##### Map

###### ConcurrentHashMap

####### 无序

###### ConcurrentSkipListMap

####### 有序；跳表，性能高

##### Set

###### CopyOnWriteArraySet

###### ConcurrentSkipListSet

##### Queue

###### 单端阻塞队列

####### ArrayBlockingQueue

######## 利用锁来保证互斥

####### LinkedBlockingQueue

####### 区别：LinkedBQ 头尾操作使用不同的锁，吞吐量会更高

###### 双端阻塞队列

####### LinkedBlockingDeque

######## 队尾插入：addLast / offerLast

######## 队尾删除：removeLast / pollLast

######## 阻塞：take / put

###### 单端非阻塞队列

####### ConcurrentLinkedQueue

###### 双端非阻塞队列

####### ConcurrentLinkedDeque

###### SynchronousQueue

https://stackoverflow.com/questions/8591610/when-should-i-use-synchronousqueue

the SynchronousQueue is more of a handoff, whereas the LinkedBlockingQueue just allows a single element. The difference being that the put() call to a SynchronousQueue will not return until there is a corresponding take() call, but with a LinkedBlockingQueue of size 1, the put() call (to an empty queue) will return immediately.

I can't say that i have ever used the SynchronousQueue directly myself, but it is the default BlockingQueue used for the Executors.newCachedThreadPool() methods. It's essentially the BlockingQueue implementation for when you don't really want a queue (you don't want to maintain any pending data).


####### cachedThreaPool的默认队列

####### 避免任务排队，put后必须被另一个线程take：size == 1

####### 更像是一个在线程间进行移交的机制，而非真正的队列

###### Disruptor

####### 用法


```java
// 自定义 Event
class LongEvent {
  private long value;
}

// 指定 RingBuffer 大小,
// 必须是 2 的 N 次方
int bufferSize = 1024;

// 构建 Disruptor
Disruptor<LongEvent> disruptor 
= new Disruptor<>(
  LongEvent::new,//EventFactory
  bufferSize,
  DaemonThreadFactory.INSTANCE);

// 注册事件处理器
disruptor.handleEventsWith(
  (event, sequence, endOfBatch) ->System.out.println("E: "+event));

// 启动 Disruptor
disruptor.start();

// 获取 RingBuffer
RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

// 生产 Event
ByteBuffer bb = ByteBuffer.allocate(8);
for (long l = 0; true; l++){
  bb.putLong(0, l);
  // 生产者生产消息
  ringBuffer.publishEvent(
    (event, sequence, buffer) -> event.set(buffer.getLong(0)), bb);
  Thread.sleep(1000);
}

```

####### 优点

######## 内存分配更合理

- RingBuffer数据结构，数组元素初始化时一次性全部创建。
- 对象循环利用，避免频繁GC


######### RingBuffer：元素内存地址尽可能连续

初始化所有元素，内存地址大概率是连续的；
利用`程序的空间局部性原理`

```java
for (int i=0; i<bufferSize; i++){
  //entries[] 就是 RingBuffer 内部的数组
  //eventFactory 就是前面示例代码中传入的 LongEvent::new
  entries[BUFFER_PAD + i] 
    = eventFactory.newInstance();
}

```

######### publishEvent()发布Event时，并不创建新对象，而是修改原对象

######## 避免伪共享，提升缓存利用率

伪共享：由于共享缓存行，导致缓存无效的场景

```
// 前：填充 56 字节
class LhsPadding{
    long p1, p2, p3, p4, p5, p6, p7;
}
class Value extends LhsPadding{
    volatile long value;
}
// 后：填充 56 字节
class RhsPadding extends Value{
    long p9, p10, p11, p12, p13, p14, p15;
}
class Sequence extends RhsPadding{
  // 省略实现
}

```

######### 缓存行填充技术

######### Java8可用 @sun.misc.Contended避免伪共享

######## 无锁算法，性能好

入队算法：

```java
// 生产者获取 n 个写入位置
do {
  //cursor 类似于入队索引，指的是上次生产到这里
  current = cursor.get();
  // 目标是在生产 n 个
  next = current + n;
  
  // 减掉一个循环
  long wrapPoint = next - bufferSize;
  // 获取上一次的最小消费位置
  long cachedGatingSequence = gatingSequenceCache.get();
  
  // 没有足够的空余位置
  if (wrapPoint>cachedGatingSequence || cachedGatingSequence>current){
    // 重新计算所有消费者里面的最小值位置
    long gatingSequence = Util.getMinimumSequence(
        gatingSequences, current);
        
    // 仍然没有足够的空余位置，出让 CPU 使用权，重新执行下一循环
    if (wrapPoint > gatingSequence){
      LockSupport.parkNanos(1);
      continue;
    }
    
    // 从新设置上一次的最小消费位置
    gatingSequenceCache.set(gatingSequence);
  } else if (cursor.compareAndSet(current, next)){
    // 获取写入位置成功，跳出循环
    break;
  }
} while (true);

```

######### 入队：如果没有空余位置，出让CPU使用权，然后重新计算

######### 入队：如果有空余位置，通过CAS设置入队索引

######## 支持批量消费

### 互斥锁

#### synchronized

##### 三种使用方式：修饰实例方法、静态方法、代码块

##### 加锁对象

###### synchronized(obj)

###### 如果要保护多个相关的资源，则要选择一个粒度更大的锁，使其覆盖所有资源

##### 原理

###### 同步块

####### Java对象头中有monitor对象；
monitorenter指令：计数器+1
monitorexit指令：计数器-1

###### 同步方法

####### ACC_SYNCHRONIZED 标识

##### 对比ReentrantLock

###### 依赖于JVM vs. 依赖于API

###### Lock增加了高级功能

####### 能够响应中断

######## lockInterruptibly()

####### 支持超时

######## tryLock(timeout)

####### 非阻塞地获取锁

######## tryLock()

####### 可实现公平锁

######## ReentrantLock(boolean fair)

###### 等待通知机制：wait/notify vs. condition

###### 性能已不是选择标准

##### 对比volatile

###### volatile是线程同步的轻量级实现，只能修饰变量

###### volatile只保证可见性，不保证原子性

###### 对线程访问volatile字段不会发生阻塞，而sync会阻塞

##### Java6的优化

###### 偏向锁，轻量级锁，自旋锁；

偏向锁：
- 默认使用
- 利用CAS，在对象头 Mark Word部分设置线程ID

轻量级锁：
- 其他线程加锁时，需要撤销偏向锁，耗时。
- 撤销成功则切换到轻量级锁。
- 否则进一步升级为重量级锁。





###### 锁消除，锁粗化

###### 不再完全依靠操作系统内部的互斥锁，不再需要用户态内核态切换

#### Lock

##### ReentrantLock

###### ReentrantLock vs. synchronized

####### see above

##### ReadWriteLock

###### 方法：writeLock / readLock

###### 实例：缓存

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

###### 允许多个线程同时`读`共享变量

####### 适合读多写少场景

###### 升级vs降级

####### 不支持锁的升级：获取读锁后，不能再获取写锁

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



####### 但支持锁的降级

###### 读锁不能newCondition 

##### StampedLock

###### 加锁解锁

####### 加锁后返回stamp，解锁时需要传入这个stamp

####### writeLock() / unlockWrite(stamp)

####### readLock() / unlockRead(stamp)

####### validate(stamp)

######## 判断是否已被修改

###### 锁的降级升级

####### 降级：tryConvertToReadLock()

####### 升级：tryConvertToWriteLock()

###### 支持三种锁模式

####### 写锁

######## writeLock()，类似WriteLock

####### 悲观读锁

######## readLock()，类似ReadLock

当多个线程同时读的同时，写操作会被阻塞

####### 乐观读

######## tryOptimisticRead()


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

######## 允许一个线程同时获取写锁

######### 而在ReadWriteLock中，当多个线程同时读的同时，写操作会被阻塞

######## stamp 类似数据库乐观锁中的version

###### 注意事项

####### 不支持重入

####### 不支持newCondition

###### 示例

####### 读模板


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

####### 写模板


```java
long stamp = sl.writeLock();
try {
  // 写共享变量
  ......
} finally {
  sl.unlockWrite(stamp);
}

```

##### 最佳实践

###### 永远只在更新对象的成员变量时加锁

###### 永远只在访问可变的成员变量时加锁

###### 永远不在调用其他对象的方法时加锁

##### 锁的膨胀与降级

#### 条件变量 Condition

##### 必须在排它锁中使用

##### ReentrantLock

##### 通知机制

###### 将wait / notify / notifyAll 转化为对象

## 模式

### 避免共享

#### Immutability模式

##### 手段

###### final类、final属性、只读

##### 例子

###### String、Long

##### tip

###### 利用享元模式避免重复创建对象

####### 例如LongCache缓存

###### 解决不可变对象引用的原子性：原子类

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
    while(true) {
      WMRange or = rf.get();
      // 检查参数合法性
      if(v < or.lower){
        throw new IllegalArgumentException();
      }
      WMRange nr = new
          WMRange(v, or.lower);
      if(rf.compareAndSet(or, nr)){
        return;
      }
    }
  }
}


```

#### Copy-on-Write模式

##### 例子

###### CopyOnWriteArrayList：无锁

###### 读多写少的路由表：CopyOnWriteArraySet.computeIfAbsent

```java
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

##### 问题

###### 为什么没提供 CopyOnWriteLinkedList?

####### 完整复制LinkedList性能开销大

#### 线程本地存储模式

##### 例子

###### 每个线程分配唯一ID

```java
static class ThreadId {
  static final AtomicLong nextId=new AtomicLong(0);
  

  static final ThreadLocal<Long>  tl
  = ThreadLocal.withInitial(
    ()-> nextId.getAndIncrement());
    
  static long get(){
    return tl.get();
  }
}
```


##### 问题

###### 可否异步传递ThreadLocal？

### 多线程版本的 IF

#### Guarded Suspension模式
（等待唤醒机制）

##### 概念

###### 保护性性暂停，又称Guarded Wait, Spin Lock

###### 将异步转换为同步

###### 多线程版本的 if：等待一个条件满足

##### 标准写法

###### 1. GuardedObject

```java
class GuardedObject<T>{
  // 受保护的对象
  T obj;
  Lock lock = new ReentrantLock();
  Condition done = lock.newCondition();
  int timeout=1;
  
  // 获取受保护对象  
  T get(Predicate<T> p) {
    lock.lock();
    try {
      //MESA 管程推荐写法
      while(!p.test(obj)){
        done.await(timeout, TimeUnit.SECONDS);
      }
    } finally{
      lock.unlock();
    }
    return obj;
  }
  
  // 事件通知方法
  void onChanged(T obj) {
    lock.lock();
    try {
      this.obj = obj;
      done.signalAll();
    } finally {
      lock.unlock();
    }
  }
}

```

###### 2. GuardedObject + map

```java
class GuardedObject<T>{
  // 受保护的对象
  T obj;
  Lock lock = new ReentrantLock();
  Condition done = lock.newCondition();
  int timeout=2;
  
  // 保存所有 GuardedObject
  static Map<Object, GuardedObject> gos=new ConcurrentHashMap<>();
  
  // 静态方法创建 GuardedObject
  static <K> GuardedObject create(K key){
    GuardedObject go=new GuardedObject();
    gos.put(key, go);
    return go;
  }
  
  // 获取受保护对象  
  T get(Predicate<T> p) {
    lock.lock();
    try {
      //MESA 管程推荐写法
      while(!p.test(obj)){
        done.await(timeout, TimeUnit.SECONDS);
      }
    }finally{
      lock.unlock();
    }
    return obj;
  }
  
  // 事件通知方法
  static <K, T> void fireEvent(K key, T obj){
    GuardedObject go=gos.remove(key);
    if (go != null){
      go.onChanged(obj);
    }
  }
  
  void onChanged(T obj) {
    lock.lock();
    try {
      this.obj = obj;
      done.signalAll();
    } finally {
      lock.unlock();
    }
  }
}

```

##### 示例

###### Dubbo

https://github.com/apache/incubator-dubbo/blob/master/dubbo-remoting/dubbo-remoting-api/src/main/java/org/apache/dubbo/remoting/exchange/support/DefaultFuture.java#L168

###### MQ异步返回结果

```java
// 处理浏览器发来的请求
Respond handleWebReq(){
  int id= 序号生成器.get();
  // 创建一消息
  Message msg1 = new Message(id,"{...}");
  
  // 创建 GuardedObject 实例
  GuardedObject<Message> go = GuardedObject.create(id);  
  
  // 发送消息
  send(msg1);
  // 等待 MQ 消息
  Message r = go.get(t->t != null);  
}

// 唤醒等待的线程
void onMessage(Message msg){
  GuardedObject.fireEvent(
    msg.id, msg);
}

```

#### Balking模式

##### 概念

###### 当volatile状态变量满足某个条件时，执行某个逻辑

###### 与Guarded Suspenstion的区别

####### Guarded Suspension会等待if条件

####### Balking不会等待

##### 标准写法

###### 互斥锁方案

###### valatile方案

####### 适用于没有原子性要求时

##### 示例

###### synchronized实现autoSave

```java
boolean changed=false;
// 自动存盘操作
void autoSave(){

  synchronized(this){
    if (!changed) {
      return;
    }
    changed = false;
  }
  
  // 执行存盘操作
  this.execSave();
}

// 编辑操作
void edit(){
  ......
  change();
}

// 改变状态
void change(){
  synchronized(this){
    changed = true;
  }
}

```

###### volatile实现保存路由表

```java
public class RouterTable {
  //Key: 接口名, Value: 路由集合
  ConcurrentHashMap<String, CopyOnWriteArraySet<Router>> 
    rt;
    
  // 路由表是否发生变化
  volatile boolean changed;
  // 将路由表写入本地文件的线程池
  ScheduledExecutorService ses = Executors.newSingleThreadScheduledExecutor();
  
  // 启动定时任务
  // 将变更后的路由表写入本地文件
  public void startLocalSaver(){
    ses.scheduleWithFixedDelay(()->{
      autoSave();
    }, 1, 1, MINUTES);
  }
  
  // 保存路由表到本地文件
  void autoSave() {
    if (!changed) {
      return;
    }
    changed = false;
    // 将路由表写入本地"文件"
    this.save2Local();
  }
  
  // 删除路由
  public void remove(Router router) {
    Set<Router> set=rt.get(router.iface);
    if (set != null) {
      set.remove(router);
      // 路由表已发生变化
      changed = true;
    }
  }
  
  // 增加路由
  public void add(Router router) {
    Set<Router> set = rt.computeIfAbsent(
      route.iface, r -> 
        new CopyOnWriteArraySet<>());
    set.add(router);
    // 路由表已发生变化
    changed = true;
  }
}

```

###### 单次初始化

```java
class InitTest{
  boolean inited = false;
  synchronized void init(){
    if (inited) {
      return;
    }
    // 省略 doInit 的实现
    doInit();
    inited=true;
  }
}

```

###### 单例模式

```java
class Singleton{
  private static volatile Singleton singleton;
  
  private Singleton() {}
  
  // 获取实例（单例）
  public static Singleton getInstance() {
    // 第一次检查
    if(singleton==null){
     synchronize{Singleton.class){
        // 获取锁后二次检查
        if(singleton==null){
          singleton=new Singleton();
        }
      }
    }
    return singleton;
  }
}

```

### 分工模式

#### Thread-per-Message模式

##### 概念

###### 分工：为每个任务分配一个独立的线程

```java
final ServerSocketChannel ssc = 
  ServerSocketChannel.open().bind(new InetSocketAddress(8080));
   
try {
  while (true) {
    SocketChannel sc = ssc.accept();
    // 每个请求都创建一个线程
    new Thread(()->{
      try {
        // 读 Socket
        ByteBuffer rb = ByteBuffer.allocateDirect(1024);
        sc.read(rb);
        // 模拟处理请求
        Thread.sleep(2000);
        // 写 Socket
        ByteBuffer wb = (ByteBuffer)rb.flip();
        sc.write(wb);
        // 关闭 Socket
        sc.close();
      }catch(Exception e){
      }
    }).start();
  }
} finally {
  ssc.close();
}   

```

###### 优化：线程池、轻量级线程（协程、Fiber)

```java
final ServerSocketChannel ssc = 
  ServerSocketChannel.open().bind(new InetSocketAddress(8080));

try {
  while (true) {
    final SocketChannel sc = ssc.accept();
    
    Fiber.schedule(()->{
      try {
        // 读 Socket
        ByteBuffer rb = ByteBuffer.allocateDirect(1024);
        sc.read(rb);
        // 模拟处理请求       LockSupport.parkNanos(2000*1000000);
        // 写 Socket
        ByteBuffer wb =(ByteBuffer)rb.flip()
        sc.write(wb);
        // 关闭 Socket
        sc.close();
      } catch(Exception e){
      }
    });
  }
} finally {
  ssc.close();
}

```

###### 类比：委托他人办理（子线程）

##### 示例

###### HTTP Server委托子线程处理HTTP请求

##### 问题

###### 线程的频繁创建销毁、可能导致OOM

#### Worker Thread模式

##### 概念

###### Thread-per-Message模式的问题：会频繁创建销毁线程，影响性能

###### 用阻塞队列做任务池、创建固定数量的线程消费队列中的任务

```java
ExecutorService es = Executors.newFixedThreadPool(500);

ServerSocketChannel ssc = ServerSocketChannel.open().bind(new InetSocketAddress(8080));
    
try {
  while (true) {
    SocketChannel sc = ssc.accept();
    
    // 将请求处理任务提交给线程池
    es.execute(()->{
      try {
        // 读 Socket
        ByteBuffer rb = ByteBuffer.allocateDirect(1024);
        sc.read(rb);
        // 模拟处理请求
        Thread.sleep(2000);
        // 写 Socket
        ByteBuffer wb = (ByteBuffer)rb.flip();
        sc.write(wb);
        // 关闭 Socket
        sc.close();
      }catch(Exception e) 
      }
    });
  }
} finally {
  ssc.close();
  es.shutdown();
}   

```

####### 其实就是线程池！

###### 类比：车间工人

##### 注意

###### 死锁问题（同一个线程池的任务一定要相互独立）

##### 示例

###### 线程池

#### 生产者-消费者模式

##### 场景

###### 解耦

###### 异步：平衡速度差异

##### 示例

###### 批量存盘

```java

BlockingQueue<Task> bq=new
  LinkedBlockingQueue<>(2000);

// 启动 5 个消费者线程；执行批量任务 

void start() {
  ExecutorService es=xecutors.newFixedThreadPool(5);
  for (int i=0; i<5; i++) {
    es.execute(()->{
        while (true) {
          // 获取批量任务
          List<Task> ts=pollTasks();
          // 执行批量任务
          execTasks(ts);
        }
    });
  }
}

List<Task> pollTasks() {
  List<Task> ts=new LinkedList<>();
  
  // 阻塞式获取一条任务
  Task t = bq.take();
  while (t != null) {
    ts.add(t);
    // 非阻塞式获取一条任务
    t = bq.poll();
  }
  return ts;
}

// 批量执行任务
execTasks(List<Task> ts) {
  // 省略具体代码无数
}

```

###### 分阶段提交

- ERROR日志立即刷盘。
- 累积500条立即刷盘。
- 累积5秒立即刷盘。

```java
class Logger {
 BlockingQueue<LogMsg> bq = new BlockingQueue<>();
  static final int batchSize=500;
  ExecutorService es = Executors.newFixedThreadPool(1);
  
  // 启动写日志线程
  void start(){
    es.execute(()->{
      try {
        // 未刷盘日志数量
        int curIdx = 0;
        long preFT=System.currentTimeMillis();
        while (true) {
          LogMsg log = bq.poll(
            5, TimeUnit.SECONDS);
          // 写日志
          if (log != null) {
            writer.write(log.toString());
            ++curIdx;
          }
          // 如果不存在未刷盘数据，则无需刷盘
          if (curIdx <= 0) {
            continue;
          }
          
          // 根据规则刷盘
          if (log!=null && log.level==LEVEL.ERROR ||
              curIdx == batchSize ||
              System.currentTimeMillis()-preFT>5000){
            writer.flush();
            curIdx = 0;
            preFT=System.currentTimeMillis();
          }
        }
      }
    });  
  }
 
}

```

### 终止线程

#### 两阶段终止模式

##### 概念

###### 阶段1：线程T1向T2发送终止指令

###### 阶段2：线程T2响应终止指令

###### 终止指令：interrupt()方法、终止标志位

##### 原理

###### 只能从Runnable状态进入Terminated状态

###### 通过 interrupt()方法能转到Runnable

##### 要点

###### 1. 仅检查终止标志位是不够的，因为线程可能处于休眠状态

###### 2. 仅检查isInterrupt也是不够的，因为第三方类库可能没有正确处理中断异常

##### 示例

###### 在线终止监控

```java
class Proxy {
  volatile boolean terminated = false;
  boolean started = false;
  
  Thread rptThread;
  
  synchronized void start(){
    // 不允许同时启动多个采集线程
    if (started) {
      return;
    }
    started = true;
    terminated = false;
    rptThread = new Thread(()->{
      while (!terminated){
        report();
        try {
          Thread.sleep(2000);
        } catch (InterruptedException e){
          // 重新设置线程中断状态
          // 可否直接break???
                      Thread.currentThread().interrupt();
        }
      }
      // 执行到此处说明线程马上终止
      started = false;
    });
    rptThread.start();
  }
  
  
  // 终止采集功能
  synchronized void stop(){
    // 设置中断标志位
    terminated = true;
    // 中断线程 rptThread
    rptThread.interrupt();
  }
}


```

####### 捕获Thread.sleep中断异常后，需要重新设置线程中断状态

###### 终止线程池

####### shutdown() / shutdownNow()

shutdown:
- 会拒绝接收新的任务
- 等线程池中正在执行的、以及队列中的任务执行完才最终关闭

shutdownNow:
- 会拒绝接收新的任务
- 线程池中正在执行的、以及队列中的任务会作为返回值返回。

#### 毒丸对象：终止生产者消费者服务

## 案例

### Guava RateLimiter 限流器

#### 应用

```java
// 限流器流速：2 个请求 / 秒
RateLimiter limiter = 
  RateLimiter.create(2.0);
// 执行任务的线程池
ExecutorService es = Executors.newFixedThreadPool(1);

// 记录上一次执行时间
prev = System.nanoTime();
// 测试执行 20 次
for (int i=0; i<20; i++){
  // 限流器限流
  limiter.acquire();
  // 提交任务异步执行
  es.execute(()->{
    long cur=System.nanoTime();
    // 打印时间间隔：毫秒
    System.out.println(
      (cur-prev)/1000_000);
    prev = cur;
  });
}

输出结果：
...
500
499
499
500
499

```

##### RateLimiter.acquire()

#### 原理

##### 令牌通、漏桶

##### 实现1：用定时器+生产者消费者？高并发时定时器精度误差大

##### 实现2：记录并动态计算下一令牌发放的时间

```java
class SimpleLimiter {
  // 下一令牌产生时间
  long next = System.nanoTime();
  // 发放令牌间隔：纳秒
  long interval = 1000_000_000;

// 预占令牌，返回能够获取令牌的时间
  synchronized long reserve(long now){
    // 请求时间在下一令牌产生时间之后
    // 重新计算下一令牌产生时间
    if (now > next){
      // 将下一令牌产生时间重置为当前时间
      next = now;
    }
    
    // 能够获取令牌的时间
    long at=next;
    // 设置下一令牌产生时间
    next += interval;
    // 返回线程需要等待的时间
    return Math.max(at, 0L);
  }
  
  
  // 申请令牌
  void acquire() {
    // 申请令牌时的时间
    long now = System.nanoTime();
    // 预占令牌
    long at=reserve(now);
    long waitTime=max(at-now, 0);
    // 按照条件等待
    if(waitTime > 0) {
      try {
        TimeUnit.NANOSECONDS
          .sleep(waitTime);
      }catch(){
      }
    }
  }
}

```

### Netty 网络应用框架

### Disruptor 队列

### HikariCP 数据库连接池

#### FastList

##### 用处

###### 类似ArrayLIst，用于在Connection中保存Statement列表

##### 性能提升

###### 逆序删除时，不用顺序查找，而用逆序查找

#### ConcurrentBag

##### 用处

###### 维护数据库连接

##### 构成

```java
// 用于存储所有的数据库连接
CopyOnWriteArrayList<T> sharedList;

// 线程本地存储中的数据库连接
ThreadLocal<List<Object>> threadList;

// 等待数据库连接的线程数
AtomicInteger waiters;

// 分配数据库连接的工具
SynchronousQueue<T> handoffQueue;

```

###### CopyOnWriteArrayList  sharedList 保存所有连接

###### ThreadLocal  threadList 保存连接

###### AtomicInteger waiters 保存等待连接的线程数

###### SynchronousQueue 用来分配数据库连接

##### add() 创建连接

```java
// 将空闲连接添加到队列
void add(final T bagEntry){
  // 加入共享队列
  sharedList.add(bagEntry);
  
  // 如果有等待连接的线程，
  // 则通过 handoffQueue 直接分配给等待的线程
  while (waiters.get() > 0 
  && bagEntry.getState() == STATE_NOT_IN_USE 
  && !handoffQueue.offer(bagEntry)) {
      yield();
  }
}

```

###### 放入sharedList

###### 如有waiters，则SynchronousQueue.offer()

##### borrow() 获取连接

```java
T borrow(long timeout, final TimeUnit timeUnit){

  //1. 先查看线程本地存储是否有空闲连接
  final List<Object> list = threadList.get();
  
  for (int i = list.size() - 1; i >= 0; i--) {
    final Object entry = list.remove(i);
    final T bagEntry = weakThreadLocals ? ((WeakReference<T>) entry).get() : (T) entry;
    // 线程本地存储中的连接也可以被窃取，
    // 所以需要用 CAS 方法防止重复分配
    if (bagEntry != null 
      && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
      return bagEntry;
    }
  }

  //2. 线程本地存储中无空闲连接，则从共享队列中获取
  final int waiting = waiters.incrementAndGet();
  try {
    for (T bagEntry : sharedList) {
      // 如果共享队列中有空闲连接，则返回
      if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
        return bagEntry;
      }
    }
    
    
    //3. 共享队列中没有连接，则需要等待
    timeout = timeUnit.toNanos(timeout);
    do {
      final long start = currentTime();
      final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
      if (bagEntry == null 
        || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
          return bagEntry;
      }
      // 重新计算等待时间
      timeout -= elapsedNanos(start);
    } while (timeout > 10_000);
    // 超时没有获取到连接，返回 null
    return null;
  } finally {
    waiters.decrementAndGet();
  }
}

```

###### 优先取ThreadLocal连接

####### CAS status

###### 再取sharedList 

####### CAS status

###### 都取不到，则等待

####### SynchronousQueue.poll()

####### CAS status

##### requite() 释放连接

###### 更新status

###### 如有waiter，则SynchronousQueue.offer()

###### 否则保存到 ThreadLocal

##### 性能提升

###### 用ThreadLocal避免部分并发问题
