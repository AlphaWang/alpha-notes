# Java基础知识图谱

## Core Java

### Exception

#### 分类

- Throwable

- Error

  无需捕获，例如 `OutOfMemoryError`

- Exception

  - Checked Exception

    可检查异常，强制要求捕获；例如 `IOException`

  - Unchecked Exception

    运行时异常，不强制要求捕获；例如`NPE`, `ArrayIndexOutOfBoundsException`

#### 最佳实践

##### try-with-resources

##### throw early, catch late

###### 对入参判空

###### 重新抛出，在更高层面 有了清晰业务逻辑后，决定合适的处理方式

##### 构造异常时，包装老异常

###### 严禁 new RuntimeExcpetion() ，无参数！

#### 问题

##### try-catch 会产生额外的性能开销

##### 创建Exception对象会对栈进行快照，耗时

##### finally 中抛出异常

###### 会覆盖try中的主异常！

###### 建议

####### try-with-resouces

####### 或在 finally 中捕获异常，并用e.addSuppressed()将其附加到主异常上

#### 典型题目

##### NoClassDefFoundError vs. ClassNotFoundException

###### ClassNotFoundException

####### 动态加载类时（反射），classpath中未找到

####### 当一个类已经某个类加载器加载到内存中了，此时另一个类加载器又尝试着动态地从同一个包中加载这个类

###### NoClassDefFoundError

####### 编译的时候存在，但运行时却找不到

##### NoSuchMethod

###### NoSuchMethodException

####### 通过反射，找不到方法

###### NoSuchMethodError

####### jar类冲突时，可能引入非预期的版本使得方法签名不匹配

####### 或者字节码修改框架（ASM）动态创建或修改类时，修改了方法签名

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

```
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

```
interface Hello {
    void sayHello();
}

class HelloImpl implements Hello {
    @Override
    public void sayHello() {
        System.out.println("Hello World");
    }
}

```



####### Proxy.newProxyInstance

```
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
// Transfer method in java.util.HashMap -
// called to resize the hashmap
 
for (int j = 0; j < src.length; j++) {
  Entry e = src[j];
  if (e != null) {
    src[j] = null;
    do {
      Entry next = e.next; 
      int i = indexFor(e.hash, newCapacity);
     e.next = newTable[i];
     newTable[i] = e;
     e = next;
   } while (e != null);
  }
} 
```

##### 哈希冲突

###### 开放地址法

###### 再哈希函数法

###### 链地址法

##### put()

```java
public V put(K key, V value) {
  return putVal(hash(key), key, value, false, true);
}


```

###### hash()

```
static final int hash(Object key) {
   int h;
   return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
``


####### key.hashCode()) ^ (h >>> 16)

####### 右移16位

######## 取int类型的一半

####### 异或运算

######## 打乱hashCode真正参与运算的低16位

###### index

```
if ((tab = table) == null || (n = tab.length) == 0)
   n = (tab = resize()).length;

   // 通过 putVal 方法中的 (n - 1) & hash 决定该 Node 的存储位置
   if ((p = tab[i = (n - 1) & hash]) == null)
      tab[i] = newNode(hash, key, value, null);
```


####### (n-1) & hash

####### 保证index总是在索引范围内

####### n为2的次方，(n-1)的每一位都是1，保证数组每一位都能放入元素，保证均匀分布

##### java8优化

###### 扩容后保持链表顺序

#### List

##### Array to List

###### int[] 转 List

####### 不能直接 Arrays.asList

####### 可用 Arrays.stream(int[]).boxed().collect()

####### 或者用 Interger[]

###### Arrays.asList 返回的是不可变集合！

###### 对原始数组的修改会影响到 List

##### List.subList 可能会很大！注意 OOM

###### 可用Stream.skip / limit

##### ArrayList vs. LinkedList

###### ArrayList

####### 随机访问快

###### LinkedList

####### 随机插入快 （理论上）

######## 前提是先已获取到待插入的指针

#### ConcurrentModificationException

##### 作用

###### 防止“结果不可预期行为”

####### 删除当前游标之后的元素，遍历正常

####### 删除当前游标之前的元素，会导致遍历遗漏！

###### fail-fast

####### 否则很难 debug

##### 原理

###### 每次增加删除元素，会 modCount++

###### remove() / next() 中会检查 modCount == expectedModCount

### compareTo

#### 利用 Comparator.comparing().thenComparingInt().compare(x, y)

#### equals, hashCode, compareTo 三者逻辑要一致

### 数值

#### 精度

##### Double 四则运算会损失精度

##### 用BigDecimal.valueOf()

#### BigDecimal

##### equals要考虑精度

###### BigDecimal.value("1").equals(...1.0) 不等！

###### 应该用 compareTo 来比较

##### 所以作为 HashMap key要注意

###### 改用 TreeMap

####### TreeMap用的是compareTo

###### 存入 BigDecimal 时，用 stripTrailingZeros 去掉末尾的0

#### 格式化

##### DecimalFormat

###### setRoundingMode

##### String.format 默认四舍五入

### 文件

#### FileReader

##### read()

###### 注意无法指定字符集

###### 传入 byte[] 作为缓冲区

#### Files

##### readAllLines()

###### --> List<String>

###### 小心OOM

##### lines()

###### --> Stream

###### 注意用 try-with-resource

#### Guava Resources

##### getResource

###### --> URL

### 日期

## 函数式编程

### Lambda表达式

#### 闭包

##### lambda表达式引用的是值，而非变量：虽然可以省略 final

##### 给变量多次赋值，然后再lambda中引用它，会编译报错

#### 函数接口

##### 即lambda表达式的类型

### 流 Stream

#### 基础

##### 实现机制

###### 惰性求值方法

####### 返回Stream

###### 及早求值方法

####### 返回另一个值

##### 性能提升

###### 执行无状态中间操作时，只是创建一个Stage来标识用户的操作；最终由终结操作触发

####### Stage的构成

######## 数据来源

######## 操作

######## 回调函数

###### 执行有状态中间操作时，需要等待迭代处理完所有的数据

###### 并行处理

####### 终结操作的实现方式与串行不一样

####### 使用ForkJoin对stream处理进行分片

####### 适用场景

######## 多核、大数据量

否则 常规迭代 性能更好，串行迭代次之


#### 构造流

##### Collection.stream()

##### Stream.of(...)

###### 传入多个元素构成一个流

##### Stream.iterate().limit()

```
Stream.iterate(2, i -> i *2)
  .limit(10)
  .forEach();
```

###### 使用迭代方式构造无限流

##### Stream.generate().limit()

```
Stream.generate(Math::random)
  .limit()
  .forEach();
```

###### 根据外部Supplier构造无限流

##### IntStream / DoubleStream

```
IntStream.range(1, 3)
  .mapToObj(x -> y);
```

#### 操作

##### 中间操作 Intermediate Operation

懒操作


###### 无状态

####### filter()

######## 过滤

####### map()

######## 转换：将Stream中的值转换成一个新的流

####### flatMap()

######## 将元素转换为Stream，然后将多个Stream连接成一个Stream

####### peek()

###### 有状态

####### sorted() / reversed()

####### distinct()

####### limit()

####### skip()

######## 跳过前N个，实现类似分页效果

#####  终结操作 Terminal Operation

###### 非短路操作

####### collect(toList())

######## 根据Stream里的值生成一个列表

####### forEach()

####### reduce

######## 从一组值中生成一个值

####### min / max(Comparator)

######## Entry.comparingByValue().reversed()

####### count()

###### 短路操作

####### anyMatch() / allMatch() / noneMatch()

####### findFirst() / findAny()

#### Collectors

##### 原生方法

###### toCollection(TreeSet::new)

####### 转成其他收集器

###### maxBy, averagingInt，summingInt

####### 统计信息

###### groupingBy

```
//1.按照用户名分组，统计下单数量
orders.stream().collect(
  groupingBy(
    Order::getCustomerName,
    counting()))
    
  .entrySet().stream()
  .sorted(Entry.comparingByValue().reversed())
  .collect(toList()));

//2.按照用户名分组，统计订单总金额
orders.stream().collect(
  groupingBy(
    Order::getCustomerName,
    summingDouble(
Order::getTotalPrice)))
      
  .entrySet().stream()
  .sorted(Entry.comparingByValue().reversed())
  .collect(toList()));

//3.按照用户名分组，统计商品采购数量
orders.stream().collect(
  groupingBy(
    Order::getCustomerName,

    summingInt(order -> order.getOrderItemList()
     .stream()
     .collect(summingInt(
  OrderItem::getProductQuantity)))))
  .entrySet().stream()
  .sorted(Entry.comparingByValue().reversed())
  .collect(toList()));

//4.统计最受欢迎的商品，倒序后取第一个
orders.stream()
  .flatMap(order -> order.getOrderItemList().stream()).collect(

 groupingBy(
  OrderItem::getProductName, 
  summingInt(OrderItem::getProductQuantity)))

 .entrySet().stream()
 .sorted(Entry.comparingByValue().reversed())
 .map(Map.Entry::getKey)
 .findFirst()
 .ifPresent(System.out::println);

//4.统计最受欢迎的商品的另一种方式，直接利用maxBy
orders.stream()
  .flatMap(order -> order.getOrderItemList().stream()).collect(

 groupingBy(
  OrderItem::getProductName,
  summingInt(OrderItem::getProductQuantity)))
 .entrySet().stream()
 .collect(
  maxBy(Entry.comparingByValue()))
 .map(Map.Entry::getKey)
 .ifPresent(System.out::println);

//5.按照用户名分组，选用户下的总金额最大的订单
orders.stream().collect(
  groupingBy(
    Order::getCustomerName,
    collectingAndThen(
     maxBy(
      comparingDouble(
      Order::getTotalPrice)), Optional::get)))

 .forEach((k, v) -> System.out.println(k + "#" + v.getTotalPrice() + "@" + v.getPlacedAt()));

//6.根据下单年月分组，统计订单ID列表
orders.stream().collect(
  groupingBy(
    order -> order.getPlacedAt().format(DateTimeFormatter.ofPattern("yyyyMM")),
    mapping(order -> order.getId(), toList()))));

//7.根据下单年月+用户名两次分组，统计订单ID列表
orders.stream().collect(
  groupingBy(order -> order.getPlacedAt().format(DateTimeFormatter.ofPattern("yyyyMM")),
    groupingBy(o
      rder -> order.getCustomerName(),
      mapping(order -> order.getId(), toList())))));

```

####### 数据分组

###### partitionBy

```
//根据是否有下单记录进行分区
Customer.getData().stream()
 .collect(
    partitioningBy(
      customer -> orders.stream().mapToLong(Order::getCustomerId)
      .anyMatch(id -> id == customer.getId()))));
```

####### 数据分区：分 true/false两组

###### joining(分隔符，前缀，后缀)

####### 拼接字符串

###### reducing(identity, mapper, op)

####### 定制

##### 自定义收集器

###### supplier() -> Supplier，后续操作的初始值

###### accumulator() -> BiConsumer，结合之前操作的结果和当前值，生成并返回新值

###### combine() -> BinaryOperator，合并

###### finisher() -> Function，转换，返回最终结果

### 并行流

#### 创建

##### Collection.parallelStream()

##### Stream.parallel()

#### 实现

##### 使用ForkJoinPool

#### 场景

##### 数据量越大，每个元素处理时间越长，并行就越有意义

##### 数据结构要易于分解

###### 好：ArrayList, IntStream

###### 一般：HashSet，TreeSet

###### 差：LinkedList，Streams.iterate，BufferedReader.lines

##### 避开有状态操作

###### 无状态：map，filter，flatMap

###### 有状态：sorted，distinct，limit

#### Arrays提供的并行操作

##### Arrays.parallelSetAll

###### 并行设置value

##### Arrays.parallelPrefix

###### 将每一个元素替换为当前元素与前驱元素之和

##### Arrays.parallelSort

### 工具

#### 类库

##### mapToInt().summaryStatistics()

###### 数值统计：min, max, average, sum

##### @FunctionalInterface

###### 强制javac检查接口是否符合函数接口标准

#### 默认方法

##### 继承

###### 类中重写的方法胜出

##### 多重继承

###### 可用InterfaceName.super.method() 来指定某个父接口的默认方法

##### 用处

###### Collection增加了stream方法，如果没有默认方法，则每个Collection子类都要修改

#### 方法引用

##### 等价于lambda表达式，需要时才会调用

#### 集合类

##### computeIfAbsent()

##### forEach()

### 范例

#### 封装：log.debug(Suppiler)

```
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

#### 孤独的覆盖：ThreadLocal.withInitial(Supplier)

```
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

#### 重复：传入ToLongFunction

例如遍历订单中的item详情数目
```
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

### 调试

#### peek()

peek()让你能查看每个值，同时能继续操作流。
​```java
album.getMusicians()
  .filter(...)
  .map(...)
  .peek(item -> System.out.println(item))
  .collect(Collectors.toSet());
```

## 性能调优

### 性能测试

#### 指标

##### 响应时间

##### 吞吐量

##### 资源使用率：CPU，内存，IO

##### 负载承受能力：抛错的极限

#### 微基准测试 JMH

https://sq.163yun.com/blog/article/179671960481783808

​	

##### 介绍

http://openjdk.java.net/projects/code-tools/jmh/



##### 范例

http://hg.openjdk.java.net/code-tools/jmh/file/3769055ad883/jmh-samples/src/main/java/org/openjdk/jmh/samples

##### 避免JVM过度优化

###### 预热

###### 防止无效代码消除：Blackhole.consume()

Dead Code Elimination

```java

public void testMethod() {
   int left = 10;
   int right = 100;
   int mul = left * right;
}



public void testMethod(Blackhole blackhole) {
   // …
   blackhole.consume(mul);
}

```

###### 防止常量折叠：State机制

Constant Folding

```java
@State(Scope.Thread)
public static class MyState {
   public int left = 10;
   public int right = 100;
}

public void testMethod(MyState state, Blackhole blackhole) {
   int left = state.left;
   int right = state.right;
   int mul = left * right;
   blackhole.consume(mul);
}

```

##### 可视化


http://deepoove.com/jmh-visual-chart/ 

http://jmh.morethan.io/


### Java调优

#### String

##### 不可变性

##### 拼接：StringBuilder

##### String.intern()节省内存

注意：
- 常量池是类似一个HashTable，如果过大会增加整个字符串常量池的负担。

###### 但回收可能会不及时？

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

###### 避免回溯问题

##### 减少分支选择

(abcd|abef) --> ab(cd|ef)

##### 减少捕获嵌套

(x) --> (?:x)

#### ArrayList vs. LinkedList

##### ArrayList

```java
  // 默认初始化容量
    private static final int DEFAULT_CAPACITY = 10;
    // 对象数组
    transient Object[] elementData; 
    // 数组长度
    private int size;

```

为什么是transient数组?  
自己实现了writeObject/readObject，避免序列化数组中的空yuan'su

###### 数组，动态扩容

###### 新增：add(E), add(index, E)

####### 触发扩容，数组拷贝

###### 删除：remove(index)

####### 数组拷贝

##### LinkedList

```java
    transient int size = 0;
    transient Node<E> first;
    transient Node<E> last;
```



###### for循环：每一次都要遍历半个list

###### iterator循环：性能更好

#### 用Stream遍历集合

##### 惰性求值

##### 并行处理

##### 适合迭代次数较多时、多核时

#### IO

##### 传统IO的问题

###### 多次内存复制

####### 内核缓存 <--> 用户控件穿冲

###### 阻塞

###### 面向流

##### NIO

###### 面向Buffe，使用缓冲区优化读写流操作

###### 使用DirectBuffer减少内存复制

###### 避免阻塞

#### 序列化

##### Java序列化

###### 概念

####### 实现Serializable

serialVersionUID 用于区分不同版本

####### ObjectInputStream 序列化，ObjectOutputStream 反序列化

####### writeObject(), readObject() 可以自定义序列化机制

####### writeReplace(), readResolve() 可以在序列化前、反序列化后处理对象

###### 缺点

####### 无法跨语言

####### 易被攻击

######## 解决：仅支持基本类型和数组类型

####### 序列化后的流太大

####### 序列化性能差

##### Protobuf序列化

###### Protocol Buffers：TLV格式

####### Tag + Length + Value

###### 推荐！

##### Kyro序列化

##### Json序列化

###### SpringCloud

##### Hessian序列化

###### Dubbo

#### RPC

##### 通信协议

###### TCP / UDP

##### 长连接

##### 非阻塞、零拷贝

###### Netty

### 多线程调优

#### 锁

##### 锁的升级

##### 锁消除、锁粗化

##### 减少锁的粒度

例如 ConcurrentHashMap


##### 乐观锁：CAS --> LongAdder

#### 减少上下文切换

##### 上下文：线程切出切入过程中，OS需要保存和恢复的进度信息

##### 线程池并非越大越好

##### 减少锁的持有时间

###### 减少加锁代码块的范围

##### 降低锁的粒度

###### 读写锁

###### 分段锁

##### 乐观锁

###### volatile

###### CAS

##### 执行notify之后，应该尽快释放锁

### JVM调优

### 设计模式调优

### 数据库调优

## 坑

### oop

#### Interger对比

##### 要用equals，不能用 ==

##### IntegerCache 缓存了 -128 ~ 127

#### 浮点数对比

##### 不能用 ==，也不能用 equals !!!

##### 推荐

###### 比较差值

###### 转成BigDecimal，再用equals

###### 用 compareTo ?

#### new BigDecimal(double) 精度损失

##### 用 new BigDecimal(String)

##### 用 BigDecimal.valueOf()

### 集合

#### Collection.toArray()

##### 不建议用无参构造函数

##### 建议 c.toArray(c)

#### Arrays.asList()

##### 返回的是 Immutable

#### Map null 值

##### HashMap 支持null key / null value

##### ConcurrentHashMap 、 Hashtable 不支持 null !!

##### TreeMap 只支持 null value

#### Map 遍历

##### entrySet()

##### Map.forEach

### 泛型

#### <? extends T> 接收数据后，不可再 add

#### <? super T> 接收数据后，不可再 get

### 异常

#### NoSuchMethodException

##### 通过反射，找不到方法

#### NoSuchMethodError

##### jar类冲突时，可能引入非预期的版本使得方法签名不匹配

##### 或者字节码修改框架（ASM）动态创建或修改类时，修改了方法签名

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

###### entry index算法： hash & (n - 1)

####### n 取为2的次方

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

```
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

```
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
