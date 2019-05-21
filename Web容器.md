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
