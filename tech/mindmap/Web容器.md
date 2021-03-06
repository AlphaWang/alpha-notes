# Web容器

## 理论

### Servlet接口

#### 作用

##### 解耦 HTTP服务器 vs. 业务逻辑

否则要在HTTP服务器里判断各个请求对应到哪个业务代码

##### Servlet容器 加载和管理业务类、转发请求到具体Servlet

#### 方法

##### init()

##### destroy()

##### getServletConfig()

###### 封装 web.xml 中的初始化参数 

##### service()

##### getServletInfo()

#### 实现类 HttpServlet

##### doGet

##### doPost

### Servlet容器

#### 工作流程

##### HTTP服务器用ServletRequest封装用户请求，并调用Servlet容器的service方法

##### Servlet容器根据URL和Servlet的映射关系，找到对应Servlet

###### 若Servlet还没加载，则反射创建；调用init()初始化；

##### 调用Servlet.service()处理请求

##### 将ServletResponse返回给HTTP服务器

#### 如何注册Servlet

##### 目录结构

###### web.xml

```xml
    <servlet>
      <servlet-name>myServlet</servlet-name>
      <servlet-class>MyServlet</servlet-class>
    </servlet>

    <servlet-mapping>
      <servlet-name>myServlet</servlet-name>
      <url-pattern>/myservlet</url-pattern>
    </servlet-mapping>

```

####### 或者标注配置

@WebServlet("/myAnnotationServlet")

###### /lib

###### /classes

##### ServletContext

###### 每个web应用创建唯一的ServletContext

###### 相当于全局对象，各Servlet通过其共享数据

#### 扩展机制

##### Filter

###### 用于干预过程：对请求和响应做统一的定制化处理

###### 限流、根据地区修改响应内容

##### Listener

###### 用于监听状态：监听Web应用的启动、停止、用户请求达到

#### 与Spring容器的关系

##### ServletContext

###### Tomcat启动时，创建ServletContext，为Spring容器提供宿主环境

##### Spring IoC容器

###### Tomcat启动过程中，触发容器初始化事件

###### Spring ContextLoaderListener会监听该事件，初始化Spring IoC容器，将其存储到ServletContext中

##### Spring MVC容器 

###### Tomcat启动过程中，扫描到DispatcherServlet、初始化之

###### DispatcherServlet初始化时会创建SpringMVC容器，从ServletContext中取出Spring根容器，将其设为自己的父容器

## Service

### 连接器

#### 功能

##### 对外交流

- 监听网络端口。
- 接受网络连接请求。
- 读取请求网络字节流。
- 根据具体应用层协议解析字节流，生成Tomcat Request对象。
- 将Tomcat Request转成标准ServletRequest。

- 调用Servlet容器，得到ServletResponse。
- 转成Tomcat Response。
- 转成网络字节流。
- 将响应字节流写回给浏览器。

#### 模块

##### ProtocolHandler

###### EndPoint

####### 底层Socket通信

###### Processor

####### 应用层协议解析

##### Adaptor

###### Tomcat Req/Res 与 ServletRequest/ServletResponse转换

#### 模块图

### 容器

#### 功能

##### 内部处理

#### Container 层次结构

server.xml 的层次结构

##### Engine

###### 表示一个引擎，用来管理多个虚拟主机

###### 维护子容器Host表：HashMap<String, Container> children

###### 负责触发Host.pipeline，将请求转发给Host自容器来处理

```java
final class StandardEngineValve extends ValveBase {

  public final void invoke(Request request, Response response) {
  
    // 拿到请求中的 Host 容器
    Host host = request.getHost();
  
    // 调用 Host Pipeline 中的第一个 Valve
    host.getPipeline().getFirst().invoke(request, response);
  }
  
}

```

##### Host

###### 表示一个虚拟主机

##### Context

###### 表示一个web应用程序

##### Wrapper

###### 表示一个Servlet

#### Mapper组件

##### 作用

###### 确定请求是有哪个Wrapper的Servlet来处理

##### 流程

###### 1.根据协议和端口号选定Service和Engine

###### 2.根据域名选定Host

###### 3.根据URL路径找到Context组件

###### 4.根据URL路径，从web.xml找到Wrapper（Servlet）

#### Pipeline-Valve管道

##### 作用

###### 让请求依次被Engine、Host、Context、Wrapper处理

##### Valve

###### 处理点，同时负责调用链

###### 方法

####### getNext() / setNext()

####### invoke() 

##### Pipeline

###### 维护Valve链表

###### 方法

####### addValve()

####### getBasic() / setBasic()

####### getFirst 

##### 触发

###### 初始由连接器中的Adapter触发

```java
// Calling the container
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
```


###### 不同容器的调用链如何触发：BasicValve

#######  

###### 结束后：最后一个valve会调用Filter.doFilter()链，最终调到Servlet.service()

###### Wrapper -> Filter -> DispatcherServlet -> Controller

### 请求流转图

####  

## LifeCycle

### 目的：实现tomcat一键式启停

#### 父组件init()方法里 创建子组件、并调用子组件的init()

#### 组合模式 

### LifeCycle接口

#### init / start / stop / destroy

#### LifeCycleBase抽象基类

##### init()：模板方法

```java
@Override
public final synchronized void init()  {
  //1. 状态检查
  if (!state.equals(LifecycleState.NEW)) {
    invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
  }

  try {
    //2. 触发 INITIALIZING 事件的监听器
    setStateInternal(LifecycleState.INITIALIZING, null, false);
        
    //3. 调用具体子类的初始化方法
    initInternal();
        
    //4. 触发 INITIALIZED 事件的监听器
    setStateInternal(LifecycleState.INITIALIZED, null, false);
        
  } catch (Throwable t) {
     ...
  }
}

```

##### 触发监听器

### LifecycleListener

#### 监听状态变化

#### 方便扩展新功能

##### server.xml中添加监听器

#### 观察者模式

## 管理组件

### 启动过程

#### 1. startup.sh

##### 运行启动类Bootstrap

#### 2. Bootstrap

##### 初始化类加载器，实例化Catalina

#### 3. Catalina 

##### 解析server.xml，创建Server组件，调用其start()

#### 4. Server

##### 维护Service数组，调用Service.start()

##### 数组动态扩容

#### 5. Service

##### 维护Connector数组、Engine实例，调用Engine/Connector.start()

## Jetty

### 架构

#### 架构图

#####  

#### 与tomcat的区别

##### Jetty没有service概念

##### Jetty connector是被所有handler共享的

##### Jetty connector共享一个全局线程池，而tomcat每个连接器都有自己的线程池

### Connector: NIO

#### Acceptor

```java
public void accept(int acceptorID) {
  ServerSocketChannel serverChannel = _acceptChannel;
  
  if (serverChannel != null && serverChannel.isOpen())
  {
    // 这里是阻塞的
    SocketChannel channel = serverChannel.accept();
    
    // 执行到这里时说明有请求进来了
    accepted(channel);
  }
}

private void accepted(SocketChannel channel){
  channel.configureBlocking(false);
  Socket socket = channel.socket();
  configure(socket);
  
  // _manager 是 SelectorManager 实例，里面管理了所有的 Selector 实例
  _manager.accept(channel);
}

```


##### 在全局线程池中执行

##### 作用：通过阻塞方式接受连接、连接成功后将SocketChannel设为非阻塞，交给Selector处理

#### Selector

```java
public void accept(SelectableChannel channel, Object attachment) {
  // 选择一个 ManagedSelector 来处理 Channel
  ManagedSelector selector = chooseSelector();
  // 提交一个任务 Accept 给 ManagedSelector
  selector.submit(selector.new Accept(channel, attachment));
}

```

##### Selector.register()：把Channel注册到Selector上，拿到一个SelectionKey

```java
 _key = _channel.register(selector, SelectionKey.OP_ACCEPT, this);

```


##### 创建EndPoint和Connection，并跟SelectionKey绑在一起

```java
private void createEndPoint(SelectableChannel channel, SelectionKey selectionKey) {
  //1. 创建 Endpoint
  EndPoint endPoint = _selectorManager.newEndPoint(channel, this, selectionKey);
    
  //2. 创建 Connection
  Connection connection = _selectorManager.newConnection(channel, endPoint, selectionKey.attachment());
    
  //3. 把 Endpoint、Connection 和 SelectionKey 绑在一起
  endPoint.setConnection(connection);
  selectionKey.attach(endPoint);
    
}

```

##### EndPoint返回一个Runnable ，扔给线程池执行

#### Connection

##### 上一步的Runnable会调用Connection回调方法来处理请求

getEndPoint().fillInterested(_readCallback);


###### 使用回调函数来模拟异步IO

##### Connection调用Handler进行业务处理

#### 流程图

##### 

### Handler：负责处理请求

#### 协调 Handler

##### 负责将请求路由到一组Handler中去

##### 例如`HandlerCollection`

#### 过滤器 Handler

##### 自己处理请求，处理完后转发到下一个Handler

##### 例如`HandlerWrapper`

#### 内容 Handler

##### 真正调用Servlet来处理请求，生成响应

##### 例如`ServletHandler`

#### 启动示例

```java
// 新建一个 WebAppContext，WebAppContext 是一个 Handler
WebAppContext webapp = new WebAppContext();
webapp.setContextPath("/mywebapp");
webapp.setWar("mywebapp.war");

// 将 Handler 添加到 Server 中去
server.setHandler(webapp);

// 启动 Server
server.start();
server.join();

```
