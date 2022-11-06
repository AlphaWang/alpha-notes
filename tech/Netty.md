

# | 网络

## || 分层

> https://osi-model.com/physical-layer/ 

**Physical Layer**

- +delimiters：告知消息的开始和结束
- +time interval：



**Routing Layer**

> 包含 OSI 层：
>
> - Data Link Layer
> - Network Layer: ip
> - Transport Layer: Mac

- 否则只能广播
- Post office(Router) decode the address, & send to the right computer
- +IP, Mac



**Behavioral Layer**

> 包含 OSI 层：
>
> - Session 层

- +Frequency & Direction of the msg
- +Context --> Conversation ——session?



**Presentation Layer**

**Application Layer**



## || HTTP

演进

- HTTP/0.9：确立CS架构，域名+IP+端口，换行作为基本分隔符
- HTTP/1.0：返回码，header，多行请求

- HTTP/1.1

  - 长连接 keep-alive，分块传输 chunked

  - 方法，返回码更全面

  - 缓存控制

  - 内容协商

- HTTP/2.0
  - 头部压缩
  - 多路复用
  - 服务端推送

- HTTP/3.0
  - 0RTT建连（UDP）

HTTPS

- 解决的问题

  - 明文通信可能被窃听
    - 加密：共享密钥 / 对称密钥；非对称密钥：私有 + 公开

  - 中间人伪装、攻击
    - 证书

  - 无法证明报文完整性，可能已被篡改



## || websocket

https://www.tutorialspoint.com/websockets/

请求

- Connection: Upgrade

- Upgrade: websocket

```sh
GET wss://echo.websocket.org/?encoding=text HTTP/1.1
Host: echo.websocket.org
Origin: https://www.websocket.org
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: xxx
```



响应

```sh
HTTP/1.1 101 Web Socket Protocol Handshake
Connection: Upgrade
Sec-WebSocket-Accept: xxx
Upgrade: websocket
```





## || STOMP

目的

- 包装 websocket



## || REST

动词

- GET：获取

- POST：创建

- PUT：更新
- DELETE：删除



REST 成熟度

> https://martinfowler.com/articles/richardsonMaturityModel.html

- Level 0
- Level 1 -资源
  - 通过拆分，解决复杂性问题

- Level 2 -HTTP动词
  - GET 安全

- Level 3 超媒体控制

  - HATEOAS：Hypertext As The Engine Of Application State	
  - 引入可发现性、自我描述能力

  

# | IO模型



## || BIO: 同步阻塞

以读取网卡数据为例

- 用户线程发起read调用后就阻塞，让出CPU；

  > 其实有三种场景可能阻塞：
  >
  > - connect阻塞
  >
  > - accept阻塞
  >
  > - read/write阻塞

- 内核等待网卡数据到来，把数据从网卡拷贝到内核空间、再拷贝到用户空间、再把用户线程叫醒



## || NIO: 同步非阻塞

以读取网卡数据为例

- 用户线程不断发起read调用，数据没到内核空间时，每次都返回失败（不阻塞）
- 直到数据到了内核空间，read调用后 等待数据从内核空间拷贝到用户空间的时间里，线程阻塞
- 等数据到了用户空间再把线程叫醒



## || IO多路复用

以读取网卡数据为例

- select：线程先发起select调用，询问内核数据准备好了吗

- 数据准备好后，用户线程发起read调用；等待数据从内核空间拷贝到用户空间的时间里，线程阻塞

- 多路复用：一次select调用可以向内核查询多个channel的状态



系统调用

- select
  - 监听用户感兴趣的文件描述符上，可读可写、异常事件的发生
  - 调用后，select()会阻塞，直到有描述符就绪 或超时
  - 缺点
    - 调用前，需要把fd从内核空间拷贝到用户空间
    - 单个进程监视的fd有数量限制，1024

- poll
  - 机制与select类似
  - 没有最大fd数量限制

- epoll
  - 使用事件驱动的方式代替轮询扫描fd
  - 将fd放到内核的一个基于红黑树实现的事件表中
  - 使用mmap实现零拷贝
    - mmap可以代替read, write的IO读写操作
    - 实现用户空间和内核空间共享一个缓存数据



## || AIO: 异步非阻塞

以读取网卡数据为例

- 用户线程发起read调用，同时注册一个回调

- 内核将数据准备好后，调用指定的回调函数



## || 非阻塞 vs. 异步

- 阻塞
  - 发起 IO 操作时，等待

- 非阻塞
  - 发起 IO 操作时，立即返回

- 同步
  - 应用程序与内核通信时，数据从内核空间到应用空间的拷贝，是由应用程序来触发

- 异步
  - 应用程序与内核通信时，数据从内核空间到应用空间的拷贝，是由内核主动发起

## || 4 words

https://stackoverflow.com/questions/2625493/asynchronous-vs-non-blocking?noredirect=1&lq=1

synchronous / asynchronous is to describe the relation between two modules.
blocking / non-blocking is to describe the situation of one module.

An example:
Module X: "I".
Module Y: "bookstore".
X asks Y: do you have a book named "c++ primer"?

1) **blocking**: before Y answers X, X keeps waiting there for the answer. Now X (one module) is blocking. X and Y are two threads or two processes or one thread or one process? we DON'T know.

2) **non-blocking**: before Y answers X, X just leaves there and do other things. X may come back every two minutes to check if Y has finished its job? Or X won't come back until Y calls him? We don't know. <u>We only know that X can do other things before Y finishes its job</u>. Here X (one module) is non-blocking. X and Y are two threads or two processes or one process? we DON'T know. BUT we are sure that X and Y couldn't be one thread.

3) **synchronous**: before Y answers X, X keeps waiting there for the answer. It means that <u>X can't continue until Y finishes its job</u>. Now we say: X and Y (two modules) are synchronous. X and Y are two threads or two processes or one thread or one process? we DON'T know.

4) **asynchronous**: before Y answers X, X leaves there and X can do other jobs. <u>X won't come back until Y calls him</u>. Now we say: X and Y (two modules) are asynchronous. X and Y are two threads or two processes or one process? we DON'T know. BUT we are sure that X and Y couldn't be one thread.



**三种 IO 模式**

- BIO
  - 类比：排队打饭

- NIO
  - 类比：点单、等待被叫

- AIO
  - 类比：包厢

- 多路复用
  - 留下一个人排号等位，通知其他人



**阻塞 vs. 非阻塞**

- 概念

  - 等待资源阶段：数据就绪前，要不要死等
  - 描述两个模块之间的关系 ---- ????

- 对比

  - 阻塞：数据不可用时，一直死等

  - 非阻塞：数据不可用时，立即返回，直到被通知资源可用——或者定时反查？

**同步 vs. 异步**

- 概念
  - 使用资源阶段：数据就绪后，数据操作由谁完成？
  - 描述某个模块的状态
- 对比
  - 同步：IO请求在读写数据时会阻塞，直到完成
  - 异步：IO请求在读写数据时立即返回，操作系统完成IO请求将数据拷贝到缓冲区后，再通知应用IO请求执行完成

# | 网络编程

## || Netty

**原理**

- **EventLoop**
  - 类似Reactor
  - 负责监听网络事件，并调用事件处理器进行处理

- **EventLoopGroup**
  - bossGroup 处理连接请求
  - workerGroup 处理读写请求



**异步模型**

- 同步会阻塞线程等待资源
  - 例如接收数据时，需要一个线程阻塞在那等待数据到来 

- 异步可以解决IO等待的问题

**实现**

- Netty 基于Java NIO，实现了线程控制、缓存管理、连接管理

- 用户只需要关心Handler即可

## || Tomcat

**配置**

- **acceptCount**：accept队列的长度；默认值是100。

  - 类比取号。

  - 若超过，进来的连接请求一律被拒绝。


  - acceptCount的设置，与应用在连接过高情况下希望做出什么反应有关系。如果设置过大，后面进入的请求等待时间会很长；如果设置过小，后面进入的请求立马返回connection refused。


- **maxConnections**：Tomcat在任意时刻接收和处理的最大连接数。-1表示不受限制。

  > BIO: =maxThreads
  > NIO: 10000

  - 类比买票


  - 当接收的连接数达到maxConnections时，Acceptor线程不会读取accept队列中的连接；这时accept队列中的线程会一直阻塞着。


- **maxThreads**: 请求处理线程的最大数量。默认200。

  - 类比观影

  

## || nginx

**应用场景**

- 静态资源服务
- 反向代理服务
- API服务 -OpenResty

**tips**

- `cp -r contrib/vim/* ~/.vim` 将vim语法高亮



## || NIO

**核心概念**

- Buffer
  - 一块连续的内存卡，是NIO读写数据的中转地

- Channel
  - 表示缓冲数据的源头或目的地

- Selector
  - 用来检测Channel上的IO事件；例如连接就绪、读就绪
  - 基于事件驱动实现，Selector轮询注册在其上的Channel
  - 循环调用 SelectionKey select()

- 多路复用



**零拷贝**

- mmap：零拷贝

  - mmap可以代替read, write的IO读写操作

  - 实现用户空间和内核空间共享一个缓存数据

- Direct Buffer：内存零拷贝



# | Netty 背景

## || vs. Java NIO

- 功能更多
  - 支持常用应用层协议
  - 解决传输问题：粘包、半包现象
  - 支持流量整形
  - 完善的断连、Idle 异常处理

- API 更强大
  - ByteBuffer --> ByteBuf

- 屏蔽细节
  - NIO AIO实现



## || Reactor

**核心流程**

- 注册感兴趣的事件

- 扫描是否有感兴趣的事件发生

- 事件发生后做出相应的处理



**三种版本**

- **BIO: Thread-Per-Connection**

  ```java
  ServerSocket ss = ;
  while (!Thread.interrupted()) {
    new Thread(new Handler(ss.accept())).start();
  }
  ```

  

- **NIO: Reactor**

  - 单线程 Reactor
    ```java
    EventLoopGroup eventGroup = new NioEventLoopGroup(1); // 1
    ServerBootstrap serverBootstrap = new ServerBootstrap(); 
    serverBootstrap.group(eventGroup);
    ```

    > 一个人包揽所有
    >
    > accept
    > read
    > decode
    > compute
    > encode
    > send

  - 多线程 Reactor
    ```java
    EventLoopGroup eventGroup = new NioEventLoopGroup(); // empty
    ServerBootstrap serverBootstrap = new ServerBootstrap(); 
    serverBootstrap.group(eventGroup);
    ```

    > 多人同时做所有

  - 主从多线程 Reactor
    ```java
    EventLoopGroup bossGroup = new NioEventLoopGroup(); 
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    ServerBootstrap serverBootstrap = new ServerBootstrap();
    serverBootstrap.group(bossGroup, workerGroup); // 2 grups
    ```

    > 专人负责迎宾



- **AIO: Proactor**



# | Netty 功能

## || 编解码

**一次编解码：FrameDecoder**

- 作用
  - 封装成帧：处理 TCP粘包、半包

- 封帧方式

  - **固定长度**
    - 解码：FixedLengthFrameDecoder
    - 编码：无

  - **分隔符**
    - 解码：DelimiterBasedFrameDecoder
    - 编码：无

  - **固定长度字段存储内容长度信息 TLV**
    - 解码：LengthFieldBasedFrameDecoder
    - 编码：LengthFieldPrepender

- 源码
  - ByteToMessageDecoder
    - LengthFieldBasedFrameDecoder
    - FixedLengthFrameDecoder



**二次编解码：ByteBuf --> Object**

- 使用

  - 示例：WorldClockClientInitializer

  - pipeline().addLast()
    ```java
    //一次解码：可变长度length字段
    ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());
    //二次解码
    ch.pipeline().addLast(new ProtobufDecoder(PersonOuterClass.Person.getDefaultInstance()));
    //一次编码
    ch.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender()); 
    //二次编码
    ch.pipeline().addLast(new ProtobufEncoder());
    ```

    

- netty支持的二次解码
  - base64
  - json
  - marshalling
  - protobuf 等等

- 源码
  - MessageToMessageDecoder
    - ProtobufDecoder
    - Base64Decoder



## || keepalive & idle 监测

- TCP 默认keepalive的问题
  - 时间太长：> 2H
  - 默认关闭
  - 无法监测应用是否可以服务

- 作用
  - 认定对方存在问题（idle），于是开始发问（keepalive）

- 代码

  - bootstrap.childOption

    ```java
    bootstrap.childOption(ChannelOption.SO_KEEPALIVE,true) 
    bootstrap.childOption(NioChannelOption.of(StandardSocketOptions.SO_KEEPALIVE), true)
    ```

    

  - 自定义 Idle 检查：IdleStateHandler
    ```java
    ch.pipeline().addLast(
      "idleCheckHandler", 
      new IdleStateHandler(0, 20, 0, TimeUnit.SECONDS));
    
    ```

    



