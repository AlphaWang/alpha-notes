**数据结构与算法**



[toc]



# | 基础

数据结构

- 一组数据的存储结构

- 数据结构为算法服务

算法

- 操作数据的一组方法
- 算法要作用在特定的数据结构之上



## || 复杂度分析

### 时间复杂度

大O表示法

- 渐进时间复杂度；表示代码执行时间随数据规模增长的变化趋势


分析

- 只关注循环执行次数最多的一段代码
- 加法法则：`总复杂度 = 量级最大的那段代码的复杂度`
- 乘法法则：`嵌套代码的复杂度 = 嵌套内外代码复杂度的乘积`



常见复杂度量级

- 常量阶 O(1)
  - 当不存在递归、循环

- 对数阶 O(logn)
  - 循环变量按“倍数”增长

- 线性阶 O(n)

- 线性对数阶 O(nlogn)

- 平方阶 O(n^2)

- 指数阶 O(2^n)

- 阶乘阶 O(n!)

场景

- 最好

- 最坏

- 平均
  
- 加权平均时间复杂度、期望时间复杂度
  
- 均摊

  - 摊还分析法

    > 对一个数据结构进行一组连续操作中，大部分情况下时间复杂度都很低，只有个别情况下时间复杂度比较高，而且这些操作之间存在前后连贯的时序关系。
    > 这个时候，我们就可以将这一组操作放在一块儿分析，看是否能将较高时间复杂度那次操作的耗时，平摊到其他那些时间复杂度比较低的操作上。而且，在能够应用均摊时间复杂度分析的场合，一般均摊时间复杂度就等于最好情况时间复杂度
  



### 空间复杂度

渐进空间复杂度，表示存储空间与数据规模之间的增长关系



## || 常见思路

### 递归

三个条件

- 一个问题的解可以分解为几个**子问题**的解

- 该问题与子问题求解思路完全一样

- 存在**递归终止条件**



思路

- 写递归代码的关键就是找到如何将大问题**分解**为小问题的规律，并且基于此写出递推公式，然后再推敲终止条件，最后将递推公式和终止条件翻译成代码

- 写出递推公式

- 找到终止条件



如何写

- 严格定义递归函数作用：参数、返回值、side-effect

- 先一般，后特殊

- 每次调用必须缩小问题规模

- 每次问题规模缩小程度必须为1

注意

- 溢出

- 警惕重复计算



### 循环

如何写

- 定义循环不变式，循环体每次结束后保持循环不变式

- 先一般，后特殊

- 每次必须向前推进循环不变式中涉及的变量值

- 每次推进的规模必须为1



### 二分

[a, b) 前闭后开

- [a, b) + [b, c) = [a, c)

- b - a = len([a, b))

- [a, a) ==> empty range



# | 数据结构

## || 线性表

### 数组

- 插入低效
  - 优化：插入索引K, 则只将原K元素移到数组尾部

- 删除低效
  - 优化：标记清除
- 内存空间连续
  - 所以申请时如果连续空间不足，会OOM

- vs. ArrayList
  - 封装细节
  - 动态扩容：创建时应事先指定大小



### 链表

**种类**

- 单链表

- 双向链表

  - 用空间换时间

  - LinkedHashMap

    >  双向链表 + 散列表
    >
    > 原生支持LRU：访问一个元素，会将其移到链表末尾

- 循环链表

- 双向循环链表

- 静态链表



**技巧**

- 理解指针或引用的含义

- 警惕指针丢失

- 利用**哨兵**简化实现难度

  > 虚拟空头。
  >
  > 否则：插入删除操作，需要对插入第一个节点、删除最后一个节点特殊处理

- 留意边界条件处理

  > 空、1节点、2节点、处理头节点尾节点

- 举例、画图



**例题**

**LRU 缓存**

https://leetcode.com/problems/lru-cache/

思路：

> 使用有序单链表，尾部表示最早使用的节点。
>
> 插入节点时，
>
> - 遍历链表，查询是否已存在；
> - 若存在，则从原位置删除、插入到头；
> - 若不存在，且缓存未满，则插入到头；
> - 若不存在，且缓存已满，则删除尾部节点，插入到头。



- 可用双向链表 + 散列表 提升性能到O(1)

- 查找
  - 散列表 O(1)
  - 并移动到双向链表尾部

- 删除
  - 通过散列表定位到节点
  - 通过节点前驱指针、后驱指针，在双向链表中删除

- 添加

  - 已有节点：移动到双向链表尾部
  - 新节点：是否已满？
    - 满：删除头部节点、新增尾部节点
    - 未满：新增尾部节点

  

**234. 判断字符串回文**

https://leetcode.com/problems/palindrome-linked-list/

**206. 单链表反转**

No. 206 https://leetcode.com/problems/reverse-linked-list/ 

**141. 链表中环的检测**

No. 141 https://leetcode.com/problems/linked-list-cycle/
No. 142 https://leetcode.com/problems/linked-list-cycle-ii/

**21. 有序链表合并**

No. 21 https://leetcode.com/problems/merge-two-sorted-lists/

**19. 删除链表倒数第n个节点**

No. 19 https://leetcode.com/problems/remove-nth-node-from-end-of-list/

**876. 求链表中间节点**

No. 876 https://leetcode.com/problems/middle-of-the-linked-list/





### 跳表

**思路**

> 对链表建立多级索引：这样链表也可以“二分查找”了
> 空间换时间：空间复杂度 O(n)



**索引动态更新**

- 随机函数

- 决定插入到第 1 ~ K 级索引



**应用**

- Redis 有序集合

> 为什么redis不用红黑树？
>
> 区间查询：跳表 O(logn)，更快
>
> 更易实现



### 栈

**定义**

- 受限的线性表
  - 入栈 push()
  - 出栈pop()

**实现**

- 顺序栈
  - 用数组实现

- 链式栈
  - 用链表实现

**应用**

- 函数调用栈

- 表达式求值
  - 操作数栈
  - 运算符栈

- 括号匹配

- 浏览器前进后退
  - 两个栈
  - 或者用双向链表

**例题**

**20. Valid Parentheses**

No. 20 https://leetcode.com/problems/valid-parentheses/

**155. Min Stack**

https://leetcode.com/problems/min-stack/

**232. Implement Queue using Stacks**

https://leetcode.com/problems/implement-queue-using-stacks/

**844. Backspace String Compare**

https://leetcode.com/problems/backspace-string-compare/

**224. Basic Calculator**

https://leetcode.com/problems/basic-calculator/

**682. Baseball Game**

https://leetcode.com/problems/baseball-game/

**496. Next Greater Element I**

https://leetcode.com/problems/next-greater-element-i/





### 队列

**定义**

- 受限的线性表
  - 入队 enqueue()
  - 出队 dequeue()
- 接口

```
- Queue 的接口
  - offer() / add()  
  - poll() / remove()
  - peek() / element() null / exception
  
- BlockingQueue
  - put() / take()

- Deque 的接口
  - getFirst() / getLast()        : exception
  - pollFirst() / pollLast()      : null
  - removeFirst() / removeLast()  : exception
  - addFirst() / addLast()
  - offerFirst() / offerLast() : addFirst() + return true
```



**实现**

- 顺序队列

用数组实现

队空：head == tail

队满：tail == n

```java
public class ArrayQueue {
  // 数组：items，数组大小：n
  private String[] items;
  private int n = 0;
  
  private int head = 0;
  private int tail = 0;
  
  // 出队
  public String dequeue() {
    // 如果 head == tail 表示队列为空
    if (head == tail) return null;
    String ret = items[head];
    ++head;
    return ret;
  }
  
  // 入队
  public boolean enqueue(String item) {
    // 如果 tail == n 表示队列已经满了
    if (tail == n) return false;
    items[tail] = item;
    ++tail;
    return true;
  }
  
  
     // 入队 + 数据搬移
  public boolean enqueue(String item) {
    if (tail == n) {
      // tail ==n && head==0，表示整个队列都占满了
      if (head == 0) return false;
      // 数据搬移
      for (int i = head; i < tail; ++i) {
        items[i-head] = items[i];
      }
      // 搬移完之后重新更新 head 和 tail
      tail -= head;
      head = 0;
    }
    
    items[tail] = item;
    ++tail;
    return true;
  }

}

```

- 链式队列

用链表实现



**种类**

- 普通队列

- 循环队列

队空：head == tail

队满：(tail + 1) % n == head

```java
public class CircularQueue {
  // 数组：items，数组大小：n
  private String[] items;
  private int n = 0;
  
  private int head = 0;
  private int tail = 0;

  // 入队
  public boolean enqueue(String item) {
    // 队列满了
    if ((tail + 1) % n == head) return false;
    items[tail] = item;
    tail = (tail + 1) % n;
    return true;
  }

  // 出队
  public String dequeue() {
    // 如果 head == tail 表示队列为空
    if (head == tail) return null;
    String ret = items[head];
    head = (head + 1) % n;
    return ret;
  }
}

```

- 双端队列

- 阻塞队列
  - 生产者消费者模型

- 并发队列
  - 加锁
  - CAS：Disruptor

- 阻塞并发队列



**应用**

- Disruptor

- ArrayBlockingQueue实现公平锁



## || 散列表

**思路**

- 数组 按下标访问，时间复杂度 O(1)

- 散列函数：将key映射为数组下标



**冲突解决**

**开放寻址 open addressing**

- 线性探测 Linear Probing
  - 如果冲突，依次往后找
  - 场景：数据量小时、装载因子小时
- 二次探测 Quadratic Probing
  - 如果冲突，按指数步长往后找
- 双重散列 Double hashing
  - 如果冲突，换hash函数

**链表法 chaining**

场景：适合存储大对象、大数据量

因为指针要消耗空间的



**动态扩容**

- 避免哈希碰撞攻击
  - 让散列表退化为链表
  - 查询时消耗CPU/线程

- 扩容后，装载因子变低，散列表变得更空

- 优化：渐进式扩容

  - 目的：避免扩容时的插入操作 耗时过长

  - 要点

    - 扩容时，仅分配新空间，不迁移数据
    - 后续插入时，一步步迁移
    - 查询时，先查新数组，再查老数组

    

**位图**

- TBD



**应用**

- 将词典放入Map，用于拼写检查

- ThreadLocalMap 使用开放寻址法

  > 数据量小



## || 树

### 二叉树

**1. 平衡二叉树**

**2. 二叉查找树**

定义：左子树 < 当前节点 < 右子树

操作

- 查找
- 插入
- 删除
  - 如果待删除节点有2个子节点；需要从右子树中找到最小节点替换到待删除节点

特点

- 中序遍历 可得有序序列

- 支持重复数据

  - 链表法

  - 或插入右子树

    即：左子树 < 当前节点 <= 右子树

vs. 哈希表

- 哈希表数据无序
- 哈希表扩容耗时
- 哈希表要考虑hash函数设计、冲突解决、扩容



**3. 平衡二叉查找树**

https://en.wikipedia.org/wiki/Self-balancing_binary_search_tree

定义

- 任意节点左右子树高度相差不大于1
- 且是二叉查找树
- 为了解决二叉树频繁插入删除时，退化成链表



- **3.1 AVL 树**

  - 每个节点存储平衡因子：子树高度差

  - 如何平衡
    - 左旋
    - 右旋
    - 左右旋
    - 右左旋


  - 缺点
    - 需要额外存储
    - 调整次数频繁：但实际上没必要完美平衡


- **3.2 2-3 树**

https://www.cnblogs.com/tiancai/p/9072813.html

- 目的
  - 保持树的矮胖、平衡


- 插入时平衡调整

  - 插入到一个 2-节点，则扩充其为 3-节点


  - 插入到一个 3-节点

    - 若无父节点，则扩充其为 4-节点，然后分解变成二叉树
    - 若有 2-节点 父节点，则扩充其为 4-节点，然后分解，并将新树父节点融入到 2-节点父节点
    - 若有 3-节点 父节点，则扩充其为 4-节点，然后分解，向上融合；上面的3-节点继续扩充、融合

    


- **3.3 红黑树**

https://www.cnblogs.com/tiancai/p/9072813.html

- 思想

  - “近似”平衡二叉树

  - 左右子树高度差小于两倍


- **实现**

  - 根节点是黑色

  - 叶子节点都是黑色、且是空节点
    - 为了简化编码

  - 相邻节点不能同时为红色
    - 否则成为 4-节点
    - 插入删除时可能破坏该要求

  - 任意节点到可达叶子节点的路径，包含相同数目的黑色节点
    - 即：平衡
    - 插入删除时可能破坏该要求




- 平衡

  - 插入时平衡调整
    - 左旋、右旋
    - 改变颜色


  - 删除时平衡调整

    https://time.geekbang.org/column/article/68976




- **vs. 2-3 树**

  - 红色，标记 3节点
    - 红节点与父节点 可以合并为一个 3-节点

  - 黑色，普通节点


- **vs. AVL树**

  - AVL 严格平衡，读更快


  - 红黑树 插入、删除更快


  - 存储

    - AVL额外存储平衡因子 
    - 红黑树 仅需 1 bit，表示黑or红


  - 场景

    - 红黑树：高级语言库中
    - AVL 树：数据库

    


**4. 完全二叉树**

最后一层叶子节点靠左排列



**5. 满二叉树**

叶子节点全在最底层



**二叉树的表示法**

- 链式存储法

- 数组

  - 索引

    - 如果下标从 1 开始

      > i / 2 父节点
      >
      > 2 * i：左子节点
      >
      > 2 * 1 +1 ：右子节点

    - 如果下标从 0 开始

      > (i - 1) / 2 父节点 --> d叉树：(i - 1) / d
      >
      > 2 * i+ 1 左子节点
      >
      > 2 * i + 2 右子节点 --> d叉树：d * i + 1  / d * i+2

  - 适用
    - 完全二叉树、满二叉树
    - 其他类型 浪费数组空间
    
    

**二叉树的遍历**

- 前序

- 中序

- 后序

```
void preOrder(Node* root) {
  if (root == null) return;
  print root // 表示打印root节点
  preOrder(root->left);
  preOrder(root->right);
}

void inOrder(Node* root) {
  if (root == null) return;
  inOrder(root->left);
  print root // 表示打印root节点
  inOrder(root->right);
}

void postOrder(Node* root) {
  if (root == null) return;
  postOrder(root->left);
  postOrder(root->right);
  print root // 表示打印root节点
}
```



例题

**二叉树 按层遍历**

https://leetcode.com/problems/binary-tree-level-order-traversal/

**104. Max Depth**

https://leetcode.com/problems/maximum-depth-of-binary-tree/

**199. Right Side View**

https://leetcode.com/problems/binary-tree-right-side-view/

**450. Delete Node**

https://leetcode.com/problems/delete-node-in-a-bst/





### 多路查找树

- B 树
- B+ 树
- 2-3 树
- 2-3-4 树



### 堆

定义

- 堆是完全二叉树

- “每个节点的值”均 >= 或 <= “子树各节点值”

类型

- 小顶堆
- 大顶堆
- 优先级队列
- 斐波那契堆
- 二项堆

存储

- 数组
  - 因为是完全二叉树

操作

- 插入
  - 先放到最后；与父节点比较、交换
  - 从下往上堆化

- 删除堆顶，即：获取最大 最小值
  - 方法一
    - 从左右子节点中找出第二大元素，放到堆顶；再迭代地删除第二大节点
    - 会出现数组空洞，不满足完全二叉树
  - 方法二
    - 把最后一个节点放到堆顶；再从上往下堆化

应用

- 优先级队列

  - 合并有序小文件

    > 从N个小文件中，取出第一条记录，放入小顶堆
    >
    > 取出堆顶，放入大文件
  
  - 高性能定时器
  
    > 堆顶是需要最先执行的任务
    >
    > 避免 频繁扫描任务列表

- Top K
  - 维护 大小为K的  “小顶堆”
  - 遍历数组，如果比堆顶大，则插入堆；否则不做处理

- 求中位数

  - 思路1：先排序，再取中位数

    缺点：动态数据，则每次都要先排序

  - 思路2：维护两个堆

    - 大顶堆：存储前半部分数据，堆顶即是中位数
    - 小顶堆：存储后半部分数据
    - 动态插入数据
      - 如果 <= 大顶堆堆顶，则插入大顶堆
      - 然后移动到另一个堆，保持平衡

- 求百分位值（例如P99）
  - 求中位数问题的变形
  - 大顶堆：N*99% 个元素*
  -  小顶堆：N*1%个元素



**其他**

- 树状数组

- 线段树



## || 图

### 图的存储

- 邻接矩阵
  - 二维数组
  - 空间浪费

- 邻接表
  - 类似散列表
  - 如果链表过长，可替换为红黑树、跳表
  - 可分片存储

### 操作

- 拓扑排序
- 最短路径
- 关键路径
- 最小生成树
- 二分图
- 最大流

### 案例

- 微信好友关系：无向图

- 微博关注关系：有向图

  - 可用两个邻接表，分别存储关注关系、被关注关系

  

# | 算法场景

## || 排序

> https://visualgo.net/zh/sorting
>
> https://mp.weixin.qq.com/s/HQg3BzzQfJXcWyltsgOfCQ



**指标**

- 内存消耗

- 稳定性

- 时间复杂度
  - 逆序度



#### O(n^2)

**1. 冒泡排序**

- 思路：两两比较

- 性能：三次赋值

自上而下对相邻的两个数依次进行比较和调整，让较大的数往下沉，较小的往上冒。

即：每当两相邻的数比较后发现它们的排序与排序要求相反时，就将它们互换。

```java
for (int i=0; i<data.length; i++) {
  for (int j=i+1; j<data.length; j++) {
    if (data[i] > data[j]) {
       int tmp = data[j];
       data[j] = data[i];
       data[i] = tmp;
    }
  }
}
```





**2. 插入排序**

- 思路
  - 假定分为  排序区 + 未排序区
  - 把第n个值 插入前面n-1个元素的排序区
    - 找到待插入位置，同时移动元素
    - 插入

- 性能
  - 一次赋值



假设前面 (n-1) 个数已经是排好顺序的。
把第n个数插到前面的有序数中，使得这n个数也是排好顺序的。

- 找到待插入位置，查找过程中移动数据
- 把n插入找到的位置。

```java
for (int i = 1; i < data.length; i++) {
  int value = data[i];
  int targetIndex = -1;
            
  // 找到待插入位置
  for (int j = i - 1; j >= 0; j--) {
    if (data[j] > value) 
      // 数据移动
      data[j+1] = data[j]; 
      targetIndex = j;
    } else {
      break;
    }
  }
  
  if (targetIndex >= 0) {
     data[targetIndex] = value;
  }
}
```



**3. 选择排序**

- 思路
  - 假定分为  排序区 + 未排序区
  - 从未排序区`选择`最小元素，放到排序区末尾

- 性能
  - 非稳定排序

思路类似插入排序，分为已排序区、未排序区；
每次从未排序区找到最小元素，放到已排序区末尾。

```java
// 重复（n-1）次
for (int i = 0; i < n; i++) {
  // 找到最小值索引
  int min = i;
  for (int j = i+1; j < n; j++) { 
    if (data[j] < data[min]) {
        min = j;
    }
  }
            
  // 将最小值和第一个没有排序过的位置交换
  if (data[min] < data[i]) {
     int temp = data[min];
     data[min] = data[i];
     data[i] = temp;
  }
            
}
```



**4. 希尔排序**

算法先将要排序的一组数按某个增量d（n/2,n为要排序数的个数）分成若干组，每组中记录的下标相差d.对每组中全部元素进行`直接插入排序`，
然后再用一个较小的增量（d/2）对它进行分组，在每组中再进行直接插入排序。
当增量减到1时，进行直接插入排序后，排序完成。



#### O(nlogn)

**5. 归并排序**

- 思路
  - 分解、合并
  - 分治、递归

- 性能
  - 非原地排序
  - 空间复杂度 O(n)



分治思想，将两个有序表合并成一个新的有序表。即：

- 把待排序序列分为若干个子序列，que'bao每个子序列是有序的。

- 然后再把有序子序列合并为整体有序序列。



**6. 快速排序**

- 思路
  - 找个基准元素，比它小的放左边，比它大的放右边
  - 递归
    - 递归排序左右两部分

- 性能

  - 最坏情况
    - 原因：基准元素选取不合理
    -  优化：三数取中法、随机法
  -  非稳定排序

  

- 流程：
  - 选择一个基准元素，通常选择第一个元素或者最后一个元素。

  - 通过一趟扫描，将待排序列分成两部分：一部分比基准元素小；一部分大于等于基准元素。

  - 此时基准元素在其排好序后的正确位置，然后再用同样的方法递归地排序划分的两部分。



**7. 堆排序**

- 思路
  - 两个步骤
    - 建堆
    - 排序

- 特点
  - 排序过程需要将最后节点与堆顶节点互换
    - 会改变原始相对顺序
    - 交换次数多余快排
  - 对 CPU 缓存不友好，并非顺序访问



堆排序需要两个过程，一是建立堆，二是堆顶与堆的最后一个元素交换位置。

所以堆排序有两个函数组成。

- 一是建堆的渗透函数，
- 二是反 复调用渗透函数实现排序的函数。



#### O(n)

**8. 桶排序 Bucket Sort**

- 思路
  - 将数据分到几个有序的桶里
  - 每个桶里的数据单独排序
  - 然后按桶的顺序依次取出

- 要求
  - 数据能划分成桶
  - 各桶数据分布较均匀

- 性能
  
- 非稳定排序
  
- 应用

  - 外部排序： 10GB订单 按金额排序

  

**9. 计数排序 Counting Sort**

- 思路
  - 特殊的桶排序
  - N个桶
  - 每个桶内元素值相同，无需再桶内排序

- 要求
  - 数据范围不大
  - 正整数

- 性能

  - 非稳定排序

-  应用

  - 成绩排序

  

桶排序的特殊情况：n个数据，划分n个桶；

关键是如何计算排序后的下标。


``` java
public void countingSort(int[] a, int n) {
  if (n <= 1) return;

  // 查找数组中数据的范围
  int max = a[0];
  for (int i = 1; i < n; ++i) {
    if (max < a[i]) {
      max = a[i];
    }
  }
  
  // 申请一个计数数组 c，
  int[] c = new int[max + 1]; 
  for (int i = 0; i <= max; ++i) {
    c[i] = 0;
  }

  // 计算每个元素的个数，放入 c 中
  for (int i = 0; i < n; ++i) {
    c[a[i]]++;
  }

  // 依次累加：为计算下标做准备
  for (int i = 1; i <= max; ++i) {
    c[i] = c[i-1] + c[i];
  }

  // 临时数组 r，存储排序之后的结果
  int[] r = new int[n];
  
  // 计算排序的关键步骤，有点难理解
  for (int i = n - 1; i >= 0; --i) {
    int index = c[a[i]]-1;
    r[index] = a[i];
    c[a[i]]--;
  }

  // 将结果拷贝给 a 数组
  for (int i = 0; i < n; ++i) {
    a[i] = r[i];
  }
}


```



**10. 基数排序 Radix Sort**

- 思路
  
  - 最低位开始，依次排序
- 要求
  - 数据可以分割出独立的 “位”
  - 而且位之间有递进的关系

- 性能

  - 非稳定排序

- 应用

  - 排序手机号

    - 先按最后一位排序
    -  再按倒数第二位重新排序，以此类推

    

将所有待比较数值（正整数）统一为同样的数位长度，数位较短的数前面补零。

然后，从最低位开始，依次进行一次排序。

这样从最低位排序一直到最高位排序完成以后，数列就变成一个有序序列。	



**JDK 排序算法**

- 原始数据类型

  - 快排

  - 双轴快排 Dual-Pivot QuickSort 

    http://hg.openjdk.java.net/jdk/jdk/file/26ac622a4cab/src/java.base/share/classes/java/util/DualPivotQuicksort.java

- 对象类型
  - TimSort：结合 `归并` + `二分插入binarySort`

    >  1 元素个数 < 32, 采用二分查找插入排序(Binary Sort)
    >
    > 2 元素个数 >= 32, 采用归并排序，归并的核心是分区(Run)
    >
    > 3 找连续升或降的序列作为分区，分区最终被调整为升序后压入栈
    >
    > 4 如果分区长度太小，通过二分插入排序扩充分区长度到分区最小阙值
    >
    > 5 每次压入栈，都要检查栈内已存在的分区是否满足合并条件，满足则进行合并
    >
    > 6 最终栈内的分区被全部合并，得到一个排序好的数组

Timsort的合并算法非常巧妙：

1 找出左分区最后一个元素(最大)及在右分区的位置
2 找出右分区第一个元素(最小)及在左分区的位置
3 仅对这两个位置之间的元素进行合并，之外的元素本身就是有序的

- 并行排序算法 parallelSort





## || 数组：双指针

同向指针：

- 含义
  - `[0, i)`：已处理
  - `[i, j)`：已忽略
  - `[j, max)`：待处理
- 通用步骤
  - 初始化指针 i, j = 0
  - while j < array.length
    - if need array[j], keep it by array[i] = array[j]; i++
    - 



反向指针：

- 含义
  - `[0, i)`：已处理
  - `[i, j)`：待处理
  - `[j, max)`：已处理
- 反向指针处理后，不会保留原来的相对位置。
- 通用步骤
  - 初始化 i = 0, j = max
  - while i <= j
    - 根据 array[i], array[j] 的取值进行处理；
    - Move i or j forward



注意：

- 区间闭合保持一致，例如都是左闭右开。



例题

- 283 - Move zeros to the end. [1, 2, 0, 3, 0, 8] --> [1, 2, 3, 8, 0, 0]
  https://leetcode.com/problems/move-zeroes/ 

- 344 - Reverse String
  https://leetcode.com/problems/reverse-string/ 

- 26 - Remove Duplicates from Sorted Array

  https://leetcode.com/problems/remove-duplicates-from-sorted-array/ 

  > 要保持顺序，用同向指针

- 80 - Remove Duplicates from Sorted Array II
- 11 - Container with Most Water
- 42 - Trapping Rain Water
- 1047 Remove All Adjacent Duplicates In String



## || 搜索

### 深度优先搜索

TBD

### 广度优先搜索

TBD

### A*启发式搜索

TBD



## || 查找 

### 二分查找

- 复杂度
  
- O(logN): 极其高效
  
- 中点的计算

  >  (low + high / 2)  可能溢出
  >
  > low + ((high - low) / 2)
  >
  > ow + ((high - low) >> 1) 位运算更高效

- 缺点

  - 不适用链表
  - 要求有序
    - 不适合频繁插入删除的场景

  - 不适合数据量太小的情况
    - 小数据量顺序遍历即可

  - 不适合数据量太大的情况
    - 因为数组要求连续内存

- 变形
  - 31 查找 = value 的起始位置
  - 31 查找 = value 的结束位置
  - 查找 >= value 的起始位置
  - 查找 <= vlaue 的结束位置
    - https://leetcode.com/problems/search-in-rotated-sorted-array/
   - 33 循环有序数组的二分查找



### 跳表查找

- 概念：链表 + 多级索引

- 复杂度
  - O(logN): 和二分查找一样高效
  - 空间复杂度：O(N)

- 插入、删除
  
- 随机函数插入几级索引
  
- 跳表 vs. 红黑树

  - 跳表还支持按区间查找：找到起始位置，然后往后遍历

  

### 线性表查找

TBD

### 树结构查找

TBD

### 散列表查找

TBD



## || 哈希

- 安全加密
  - MD5
  - SHA
  - DES / AES
  - 注意：加 Salt

- 唯一标识
  
- 图片摘要：分区取N字节 + MD5
  
- 数据校验
  
- BT 种子文件
  
- 散列函数
- 负载均衡
  - Session Sticky
    - 映射关系表：会很大·且不好维护
    - 哈希算法能解决

- 数据分片
  - 统计大日志文件中的URL访问次数
    - 对URL分片
    - MapReduce

- 分布式存储

  - 一致性哈希

  

## || 字符串匹配

### 朴素 Brute Force

- 思路：遍历主串，依次与模式串对比
- 复杂度：O(n*m)

### Robin-Karp

- 思路

  - 对主串中 n-m+1 个子串分别求 HASH

  > 每个字母用 26 进制转成十进制数
  >
  > 相邻子串的哈希值计算有共享部分，节约效率

  - 与子串哈希值比较

- 复杂度：O(n)

- 问题：哈希值可能超出整型范围

  - 将字母转换为数字 0 ~26，求和得到哈希
  - 将字母转换为素数，求和得到哈希
  - 哈希冲突的处理：比较字符串

### Boyer-Moore

- 思路

  - BF/RK算法的问题：遇到不匹配的字符时，模式串往后滑动一位

  - 解决思路：尽量往后多滑动几位

- 原理

  - 坏字符规则
    - 匹配顺序：模式串从右往左匹配
    - 遇到不匹配字符（坏字符），则在模式串中查找是否有此字符
      - 有：滑动其到坏字符位置
      - 无：滑动整个模式串到坏字符之后

  - 好后缀规则
    -  概念：后缀匹配，但后缀{u}的前一个字符不匹配
    - 思路1
      - 如果模式串中有 {u}，则滑动与好后缀对齐
      - 如果没有，则整体滑动到好后缀之后
      - 可能会过头，遗漏
    - 思路2
      - 匹配{u}的后缀子串与模式串的前缀子串

### KMP

- 思路
  - 在模式串和主串匹配的过程中，当遇到坏字符后，对于已经比对过的好前缀，能否找到一种规律，将模式串一次性滑动很多位？

### Trie 树

- 目的：解决字符串快速匹配问题

- 实现：N 叉树
  - 用 Node[26] 来存储子节点
  - 以空间换时间
  - 每个节点存储一个字符

- 案例

  - 输入提示
    - 预先将字典构造成 Trie 树

  - 敏感词过滤
    - 预先将敏感词构造成 Trie 树
    - 将用户输入内容作为主串
      - 从第一个字符开始在Trie树种匹配
      - 如果不能匹配到，则将主串开始匹配位置后移一位
      - 可通过 AC自动机，使得匹配失败时尽可能往后躲滑动几位

### AC 自动机

- vs. Trie 树
  - 类似 朴素BF --> KMP

### 后缀数组

TBD





## || 时间轮

- 作用
  - 定时任务：例如超时标记、心跳维护
  - 否则
    - 为每个连接建立一个 Timer 线程，线程数会很多！
    - 用统一的 Timer 线程，浪费 CPU



- 实现

  https://www.singchia.com/2017/11/25/An-Introduction-Of-Hierarchical-Timing-Wheels/



## || 其他

### 数论

### 计算几何

### 概率分析

### 并查集

### 拓扑网络

### 矩阵运算

### 线性规划





# | 基本算法思想

## || 贪心算法

解决问题

- 选出几个数据，在满足限制值的情况下，期望值最大

思路

- 算出数据的单价：相同限制值的情况下，谁的期望值贡献最大

- 依次选择单价最大的数据

- 总是做出在当前看来是最好的选择

案例

- 分糖果
  - 限制糖果个数，期望满足最多的孩子；每个孩子对糖果大小的需求不一
  - 优先选择：需求最小的孩子

- 零钱凑整
  - 限制凑整数额，期望用到最少张数的零钱
  - 优先选择：币值大的

- 区间覆盖

  - 限制区间不可覆盖，找出两两不相交的区间最大值

  - 优先选择：左端点与前一个区间不重合、右端点尽量小

应用

- 霍夫曼编码



## || 分治算法

- 分而治之
  - 递归地解决这些子问题，然后合并结果
  - 将原问题划分成n个规模较小、结构与原问题相似的子问题 

- 应用

  - MapReduce

  - 10GB 订单文件按金额排序

- 例题

  - 二维平面上有 n 个点，如何快速计算出两个距离最近的点对？

  - 有两个 n*n 的矩阵 A，B，如何快速求解两个矩阵的乘积 C=A*B？

  

## || 回溯算法

- 解决问题
  - 解决广义的搜索问题：从一组可能的解中，选出一个满足要求的解
  - 贪心算法不一定得到最优解

- 思路
  - 问题求解过程分成多个阶段
  - 每个阶段随意选一条路，当发现走不通时，回退到上一路口，另选一条路
  - 本质是枚举解法

- 应用

  - 八皇后问题

  - 0-1 背包问题

  - 正则表达式

  

## || 动态规划

- 思路
  - 把问题分解为多个阶段，每个阶段对应一个决策。
  - 我们记录每一个阶段可达的状态集合（去掉重复的），
  - 然后通过当前阶段的状态集合，来推导下一个阶段的状态集合，动态地往前推进。

- 实现
  - 分解子问题、保存之前的运算结果
  - 根据之前的运算结果对当前进行选择，可回退

- 关键点

  - DP 与递归/分治 没有根本上的区别

  - 共性：找到重复子问题

  - 差异：最优子结构、中途可以淘汰次优解

  

## || 枚举算法

TBD

## 对比：贪心 / 回溯 / 动规

- 贪心：一条路走到黑，就一次机会，只能哪边看着顺眼走哪边
- 回溯：一条路走到黑，无数次重来的机会，还怕我走不出来 (Snapshot View)
- 动态规划：拥有上帝视角，手握无数平行宇宙的历史存档， 同时发展出无数个未来 (Versioned Archive View)



# | 实战

## || Redis 数据结构

- list
  - ziplist
  - 双向链表

- hash

  - ziplist

  - hashmap

- set

  - 有序数组：仅当元素较少，且为整数

  - 散列表

- sortedset
  - 跳表

## || 搜索

### 构建步骤

#### 1. 搜集

- 算法
  - 图的广度优先搜索
  - 布隆过滤器

- 存储
  - links.bin
    - 待爬取链接文件
    - 为什么不直接存到内存？：存不下、 断点续爬
  - bloom_filter.bin
    - 已爬取的布隆过滤器
  - doc_raw.bin
    - 合并存储已爬取的网页
    - doc1_id / doc1_size / doc1
  - doc_id.bin
    - 网页链接 - 编号 对应关系

#### 2. 分析

- 算法
  - AC 自动机
    - 过滤掉html标签
    - 抽取网页文本信息
  - Trie 树
    - 分词
    - 可用 最长匹配规则

- 存储
  - mp_index.bin
    - 单词与网页的对应关系
    - term1_id / doc_id
  - term_id.bin
    - 单词与网页编号的对应关系

#### 3. 索引

- 算法
  - 多路归并排序
  - 处理 tmp_inde.bin 关系文件，按单词编号排序

- 存储
  - index.bin
    - 倒排索引文件
    - 单词编号 与 网页编号对应关系
    - tid1_docIdList
  - term_offset.bin
    - 单词编号在索引文件中的位置偏移量
    - 以便快速找到索引

### 查询步骤

#### 4. 查询

- 步骤
  - 对输入文本进行分词
  - 查询 term_id.bin：查找单词编号
  - 查询 term_offset.bin：获得单词在倒排索引文件的偏移量
  - 查询 index.bin：获取网页编号列表
  - 网页评分排序
  - 查询 doc_id.bin

######## 获取网页链接

## || Disruptor 队列

https://www.baeldung.com/lmax-disruptor-concurrency

- 原理
  - 循环队列
  - 基于数组实现，而非链表

- 高性能

  - 无锁
  - 写入前，加锁批量申请空闲存储单元
    - 两阶段写入
  - 读取前，加锁批量申请可读取的存储单元

  

## || 微服务鉴权

例如按照URL进行规则匹配

- 精确匹配
  
- KMP / BM / BF
  
- 前缀匹配
  
- Trie 树
  
- 模糊匹配

  - 回溯算法

  

## || 微服务限流

- 固定时间窗口限流

- 滑动时间窗口限流
  
- 循环队列
  
- 更平滑的限流

  - 令牌桶

  - 漏桶

  

## || 短网址

### 哈希算法

- 通过 MurmurHash 获得 32 bits 数字

- 转成62 进制，得到短字符串

解决哈希冲突

- 查已生成的短网址表，如重复，则对比原始网址
- 或利用唯一索引

- 原始网址不一样，则冲突；拼接特殊字符  再次处理

优化

- 布隆过滤器，查询已有的短网址

### ID 生成器

- 数据库自增字段

- 前置发号器
  - 批量预先配置ID

- 多个ID生成器，规则保证 ID 不重复



# | 参考

- LeetCode解析

https://labuladong.gitbook.io/algo/

https://github.com/labuladong/fucking-algorithm 

- CS-NOTES 

https://github.com/CyC2018/CS-Notes

- 动图

https://visualgo.net/