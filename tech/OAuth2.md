[toc]

# | 基础

**开放系统的授权问题**

- 复制用户名密码

- 通用的developer key
  - 可用于同一公司两个业务部分
- 特殊令牌



**传统安全问题**

- Filter：鉴权登录

- cookie + session



# | 核心概念

## 角色



### 1. 客户应用 Client Application

- 第三方应用。



### 2. 资源服务器 Resource Server 

- 受保护的资源。



### 3. 授权服务器 Authorized Server



提供 API

- /oauth2/authorize 授权
- /oauth2/token 访问令牌
- /oauth2/introspect
- /oauth2/revoke 撤回



### 4. 资源拥有者 Resource Owner

- 登录用户。



## 信息

### 客户凭证 Client Credentials



### 令牌 Token

1. **访问令牌 Access Token**

访问受保护资源

2. **刷新令牌 Refresh Token**

用于去授权服务器获取一个新的访问令牌

3. **授权码 Authorization Code Token**

用于交换获取访问令牌和刷新令牌

4. **Bearer Token**

不管谁拿到Token都可以访问资源

5. **PoP Token (Proof of Possession)**

可以校验client是否对token有明确的拥有权





### 作用域 Scopes





# | 模式

## || 概念

### 前端渠道

授权服务器 - 资源拥有者



### 后端渠道

授权服务器 - 资源服务器





## || 典型OAuth Flow

### 1. 授权码模式 Authorization Code

![image-20210630103055094](../img/auth/access_code_flow.png)

**Steps**

1. 用户访问客户应用。
2. 客户端**重定向**到授权服务器。
3. 用户授权。
4. 授权后，授权服务器生成授权码 Authorization Code，
5. 并将用户**重定向**到客户应用事先指定的 Redirect URI，同时附上授权码 Authorization Code。
6. 客户应用服务器收到授权码，附上早先的"重定向URI"，向授权服务器**申请令牌**。这一步是在客户应用的后台的服务器上完成的，对用户不可见。
7. 认证服务器核对了授权码和重定向URI，确认无误后生成 access_token，
8. 并向客户应用服务器发送访问令牌（access token）和更新令牌（refresh token）。
9. 接下来用户就可以使用客户应用了。



**前端渠道 vs. 后端渠道**

- 通过前端渠道，客户获取授权码Authorization Code

- 通过后端渠道，客户使用授权码去交换Access Token、以及可选的Refresh Token

**场景**

- 假定资源拥有者和客户在不同的设备上

- 安全性好，因为令牌不会传递经过user-agent



**为什么必须要有授权码？**

Q: Step 4/5 跳过授权码，直接返回用 access token 不行吗？

- 如果把安全保密性要求极高的访问令牌暴露在浏览器上，有访问令牌失窃的安全风险。所以只能把访问令牌返回给后端服务。但是，浏览器此时还处于被重定向到授权服务 --> 用户和客户应用之间的“连接”就断了，缺了 Step5 重定向。
- 有了授权码的参与，访问令牌可以在后端服务之间传输，同时还可以重新建立用户和客户应用之间的“连接”。这样通过一个授权码，既“照顾”到了用户的体验，又“照顾”了通信的安全。



### 2. 简化模式 Implicit 

（A）客户端将用户导向认证服务器。

（B）用户决定是否给于客户端授权。

（C）假设用户给予授权，认证服务器将用户导向客户端指定的"重定向URI"，并在URI的Hash部分包含了访问令牌。

（D）浏览器向资源服务器发出请求，其中不包括上一步收到的Hash值。

（E）资源服务器返回一个网页，其中包含的代码可以获取Hash值中的令牌。

（F）浏览器执行上一步获得的脚本，提取出令牌。

（G）浏览器将令牌发给客户端。



**流程**

直接在浏览器中向认证服务器申请令牌，跳过了"授权码"这个步骤

只有前端渠道：Access Token直接从授权服务器返回

不支持Refresh Token

**场景**

假定资源拥有者和公开客户在同一个设备上

安全性低：令牌对访问者是可见的，且客户端不需要认证



### 3. 用户名密码模式 Resource Owner Credentials
（A）用户向客户端提供用户名和密码。

（B）客户端将用户名和密码发给认证服务器，向后者请求令牌。

（C）认证服务器确认无误后，向客户端提供访问令牌。

**流程**

用户向客户端提供自己的用户名和密码。

客户端使用这些信息，从授权服务器获取Access Token

不支持Refresh Token

**场景**

假定资源拥有者和公开客户在相同设备上

适用用用户名密码登录的应用，例如桌面app



### 4. 客户端凭证模式 Client Credentials
（A）客户端向认证服务器进行身份认证，并要求一个访问令牌。

（B）认证服务器确认无误后，向客户端提供访问令牌。

**流程**

只有后端渠道：使用客户凭证获取一个Access Token

客户端以自己的名义，而不是以用户的名义，向"服务提供商"进行认证。

**场景**

适用于服务器间通信场景

因为客户凭证可以使用对称/非对称加密，该方式支持共享密码或者证书

## || 如何选择？

若访问令牌拥有人是机器

--> 客户端凭证模式



若访问令牌拥有人是用户

客户类型是web服务器应用

-> 授权码模式



客户类型是第三方APP

-> 授权码模式

客户类型是第一方APP/SPA

-> 用户名密码模式

客户类型是第三方SPA

-> 简化模式





# | Spring Security OAuth2





# | 参考

OAuth2最简指导

https://medium.com/@darutk/the-simplest-guide-to-oauth-2-0-8c71bd9a15bb

阮一峰：理解OAuth2

http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

OIDC

https://connect2id.com/assets/oidc-explained.pdf
