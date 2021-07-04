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

概念

**前端渠道**

- 授权服务器 - 资源拥有者

**后端渠道**

- 授权服务器 - 资源服务器





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

  > 通过浏览器促成了授权码的交互流程；

- 通过后端渠道，客户使用授权码去交换Access Token、以及可选的Refresh Token

**场景**

- 假定资源拥有者和客户在不同的设备上

- 安全性好，因为令牌不会传递经过user-agent



**为什么必须要有授权码？**

Q: Step 4/5 跳过授权码，直接返回用 access token 不行吗？

- 如果把安全保密性要求极高的访问令牌暴露在浏览器上，有访问令牌失窃的安全风险。所以只能把访问令牌返回给后端服务。但是，浏览器此时还处于被重定向到授权服务 --> 用户和客户应用之间的“连接”就断了，缺了 Step5 重定向。
- 有了授权码的参与，访问令牌可以在后端服务之间传输，同时还可以重新建立用户和客户应用之间的“连接”。这样通过一个授权码，既“照顾”到了用户的体验，又“照顾”了通信的安全。



### 2. 资源拥有者凭据许可 Resource Owner Credentials

> 用户名密码模式
>
> 客户应用只需要使用一次用户名和密码数据来换回一个 token，进而通过 token 来访问资源，以后就不会再使用用户名和密码了。

![resource_owner](../img/auth/resource_owner_flow.png)



**Steps** 

1. 用户向客户应用提供用户名和密码。

2. 客户应用将用户名和密码发给认证服务器，请求 access_token。

   > `grant_type == password`

3. 认证服务器确认无误后，向客户应用提供 access_token。

4. 客户应用访问受保护资源时，附带 access_token.



**场景**

- 客户应用是官方出品。
- 假定资源拥有者和公开客户在相同设备上

- 适用用用户名密码登录的应用，例如桌面app



### 3. 客户端凭据许可 Client Credentials

![](../img/auth/client_credential_flow.png)



**Steps**

1. 客户应用向认证服务器进行身份认证，并要求一个访问令牌。

   > `grant_type == client_credentials`
   >
   > 传入 app_id, app_secret，返回 access_token

2. 认证服务器确认无误后，向客户端提供访问令牌。

**流程**

- 只有后端渠道：使用客户凭证获取一个Access Token

- 客户应用以自己的名义，而不是以用户的名义，向"服务提供商"进行认证。

**场景**

- 适用于服务器间通信场景，没有明确的资源拥有者。-- 例如获取服务logo信息。

因为客户凭证可以使用对称/非对称加密，该方式支持共享密码或者证书





### 4. 隐式许可 Implicit 

![](../img/auth/implicit_flow.png)

**Steps**

1. 用户通过浏览器访问客户应用，客户应用相当于嵌入在浏览器中执行的应用程序；

2. 浏览器请求 access_token

   > `response_type == token`，同授权码模式
   >
   > 请求中附带 redirect_uri / app_id

3. 授权服务生成 access_token，并返回；

   > 调用 redirect_uri，将 access_token 作为参数返回

4. 浏览器访问受保护资源时，携带 access_token.



**要点**

- 直接在浏览器中向认证服务器申请令牌，跳过了"授权码"这个步骤

- 只有前端渠道：Access Token直接从授权服务器返回

- 不支持Refresh Token



**场景**

- 客户应用没有后端服务的情况；可以理解为 客户应用直接嵌入在浏览器中。
- 没有 app_id， app_secret；
- 假定资源拥有者和公开客户在同一个设备上

**安全性低**：令牌对访问者是可见的，且客户端不需要认证。



## || 如何选择？

**按场景选择**

- 优先考虑**授权码模式**；
- 如果客户应用是官方出品，则可直接使用**资源拥有者凭据许可**；
- 如果客户应用要获取的信息不属于任何第三方用户，则可直接使用**客户端凭据许可**；
- 如果客户应用是只嵌入到浏览器的应用，且没有服务端，则只能选择**隐式许可**；



**对比**：4 种授权许可类型获取 access_token的方式不同

| 授权许可类型               | 获取 access_token 的方式                                |
| -------------------------- | ------------------------------------------------------- |
| Authorization Code         | 通过授权码获取 access_token                             |
| Resource Owner Credentials | 通过第三方软件 app_id / app_secret 获取 access_token    |
| Client Credentials         | 通过用户名密码获取 access_token                         |
| Implicit                   | 通过嵌入浏览器中的第三方软件 app_id 来获取 access_token |



# | 授权服务器流程

![auth_server_flow](../img/auth/auth_server_flow.png)



## || 客户应用注册

注册后存储的信息：

- app_id
- app_secret
- redirect_uri
- scope



## || 颁发授权码

Request

```
 curl -X GET -H "Accept: application/x-www-form-urlencoded" $authorization_endpoint
 ?client_id=<client application id in DN format>
 &redirect_uri=<registered client callback url>
 &response_type=code
 &scope=openid <custom scopes>
 &state=<random guid>
 &code_challenge= <code challenge>
 &code_challenge_method =S256
```

Response

```
HTTP/1.1 302 Found
Location: <registered redirect_uri>?code=<code>&state=<same guid from the request>
```



### 1. 验证基本信息

- 验证客户应用合法性

- 验证回调地址合法性。

验证通过后，生成页面，提示资源拥有者进行授权。



### 2. 验证权限范围1

- 客户应用会传入权限范围
- 授权服务器将传入的权限范围，与注册的 scope 做对比



### 3. 生成授权请求页面

- 用户可以在此页面选择 已注册的权限（多选），点击 approve；
- approve 之后，生成授权码的流程才正式开始。

Q: 这里和Step1/2 的页面不同？？



### 4. 验证权限范围2

- 收到用户选择的权限范围，与注册的 scope 做对比



### 5. 处理授权请求，生成授权码

- 检查传入的 `response_type == code`

  > response_type 可能取值：code, token

- 生成授权码，并关联到 app_id + user

  > ```java
  > String code = generateCode(appId,"USERTEST");//模拟登录用户为USERTEST
  > 
  > private String generateCode(String appId,String user) {
  >   ...
  >   String code = strb.toString();
  >   codeMap.put(code,appId+"|"+user+"|"+System.currentTimeMillis());
  >   return code;
  > }
  > ```

- 并将授权码与已经授权的scope关联

  > ```java
  > Map<String,String[]> codeScopeMap =  new HashMap<String, String[]>();
  > 
  > codeScopeMap.put(code, rscope);//授权范围与授权码做绑定
  > ```

- 注意授权码要有有效期



### 6. 重定向到客户应用

- 调用 redirect_uri，将授权码告知客户应用



## || 颁发访问令牌

Request

```
 curl -X POST -H "Content-type: application/json" -H "Accept: application/x-www-form-urlencoded" $token_endpoint
 ?grant_type= authorization_code
 &code= <authorization code>
 &client_id= <client application id in DN format>
 &client_assertion= <client's TrustFabric token>
 &client_assertion_type= urn:ietf:params:oauth:client-assertion-type:jwt-bearer
 &code_verifier=<code verifier generated for PKCE along with code challenge>
 &response_type= token
 &redirect_uri= <same redirect uri used in authorize call>
```



### 1. 验证客户应用是否存在

- 验证 `grant_type == authorization_code` 
- 验证 app_id
- 验证 app_secret



### 2. 验证授权码

- 颁发授权码时保存了 授权码和 app_id + user 的关系；验证当前授权码存在、并清空存储

  > ```java
  > codeMap.put(code,appId+"|"+user+"|"+System.currentTimeMillis());
  > ```

  > ```java
  > if(!isExistCode(code)){//验证code值
  >   //code不存在
  >   return;
  > }
  > codeMap.remove(code);//授权码一旦被使用，须立即作废
  > ```



### 3. 生成访问令牌

- 原则：唯一性、不连续性、不可猜性。

- 可用 JWT 

- 生成后存下来：access_token -- scope; access_token -- app_id + user，并设置过期时间。

  > ```java
  > Map<String,String[]> tokenScopeMap =  new HashMap<String, String[]>();
  > 
  > String accessToken = generateAccessToken(appId, "USERTEST");//生成访问令牌access_token的值
  > tokenScopeMap.put(accessToken, codeScopeMap.get(code));//授权范围与访问令牌绑定
  > 
  > //生成访问令牌的方法
  > private String generateAccessToken(String appId,String user){
  >   
  >   String accessToken = UUID.randomUUID().toString();
  >   String expires_in = "1";//1天时间过期
  >   tokenMap.put(accessToken,appId+"|"+user+"|"+System.currentTimeMillis()+"|"+expires_in);
  > 
  >   return accessToken;
  > }
  > ```



## || 刷新令牌

有了刷新令牌，用户在一定期限内无需重新点击授权按钮，就可以继续使用第三方软件。

### 1. 接收刷新令牌请求，验证基本信息

- 验证 `grant_type == refresh_token` 

  > grant_type 可能取值：authorization_code, refresh_token

- 验证客户应用是否存在

- 验证刷新令牌是否存在

  > ```java
  > if(!refreshTokenMap.containsKey(refresh_token))
  > ```

- 验证刷新令牌是否属于该客户应用

  > ```java
  > String appStr = refreshTokenMap.get("refresh_token");
  > if(!appStr.startsWith(appId+"|"+"USERTEST")){
  >     //该refresh_token值不是颁发给该第三方软件的
  > }
  > ```

一个刷新令牌被使用后，需要将其废弃，并重新颁发一个刷新令牌。



### 2. 重新生成访问令牌

流程同“颁发访问令牌”。



> 客户应用如何使用刷新令牌？
>
> 1. 将 expires_in 保存下来，并定时检测；发现将要过期则利用 refresh_token 重新请求授权服务、获取新的访问令牌。
> 2. 现场发现：当突然收到一个访问令牌失效的响应，则立即使用 refresh_token 请求访问令牌。



# | JWT 

> JSON Web Token

## JWT 结构

结构体

- HEADER
  - `typ`  类型
  - `alg` 签名算法
- PAYLOAD
  - `sub` Subject，令牌的主体
  - `aud` Audience，the recipients that the JWT is intended for.
  - `exp` Expiration Time，过期时间
  - `iat` Issue at，颁发时间 
  - `iss` Issuer
- SIGNATURE



更多字段：https://datatracker.ietf.org/doc/html/rfc7519 



## JWT 代码使用

JJWT 封装了 Base64URL 编码和对称 HMAC、非对称 RSA 的一系列签名算法。使用 JJWT，我们只关注上层的业务逻辑实现，而无需关注编解码和签名算法的具体实现。

```java
String sharedTokenSecret = "hellooauthhellooauthhellooauthhellooauth";//密钥
Key key = new SecretKeySpec(sharedTokenSecret.getBytes(),
                SignatureAlgorithm.HS256.getJcaName());

//生成JWT令牌
String jwts = Jwts.builder()
    .setHeaderParams(headerMap)
    .setClaims(payloadMap)
    .signWith(key,SignatureAlgorithm.HS256)
    .compact()

//解析JWT令牌
Jws<Claims> claimsJws  = Jwts.parserBuilder()
    .setSigningKey(key).build()
    .parseClaimsJws(jwts);
JwsHeader header = claimsJws.getHeader();
Claims body = claimsJws.getBody();  

```





## JWT 优缺点

优点

- 用计算代替存储。无需远程调用即可解析token内容。
- 加密。
- 无状态原则，增强可用性、伸缩性。



缺点

- 变更令牌状态之后，如何在已使用的令牌上生效？

  > 解决1：密钥粒度缩小到用户级别，当用户取消授权后，同时修改密钥。--> 需要额外的密钥管理服务。
  >
  > 解决2：把用户密码作为 jwt 密钥。



# | Spring Security OAuth2





# | 参考

OAuth2最简指导

https://medium.com/@darutk/the-simplest-guide-to-oauth-2-0-8c71bd9a15bb

阮一峰：理解OAuth2

http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

OIDC

https://connect2id.com/assets/oidc-explained.pdf
