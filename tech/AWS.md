# Organizational Complexity 

## 多账户策略

### 身份账户

> Identity Account Architecture

- 在单一的中央区域对所有用户进行**集中管理**；并允许他们访问多个 AWS 账户下的不同 AWS 资源。
- 可以通过**跨账户 IAM 角色** 和 **身份联合**（Federation）来完成。



<img src="/Users/zhongxwang/Library/Application Support/typora-user-images/image-20210316221217896.png" alt="image-20210316221217896" style="zoom:67%;" />

#### 实践

**背景**

- AS-IS：假设有 4 个 aws 账户，就需要为新员工创建 4 个 IAM 用户、管理不便；
- TO-BE：只需将新员工加入身份账户；

**思路**

- 设置独立专用的 `身份账户 Identity Account`，集中建立用户、密码、访问秘钥、管理用户。

  > 建议身份账户只用于用户管理，不做资源管理

- 在身份账户与企业其他账户直接**建立信任关系**，在其他账户中建立**角色**，将身份账户设置为受信任实体，并设置**策略**。

- 用户登录身份账户后，通过**切换角色**，即可访问不同账户的资源。

**步骤**

- 在身份账户中创建一个 **IAM 用户** zhangsan；
  - IAM --> 用户 --> 添加用户 
- 在生成环境账户中创建一个 **跨账户角色** CA-TEST，并分配S3完全访问权限；
  - IAM --> 角色 --> 创建角色 
  - 选择受信任实体类型 = 其他AWS账户 --> 输入 zhangsan id
  - 选择权限策略 --> S3 完全访问
  - 摘要：拷贝出切换链接
- 在身份账户中配置允许 zhangsan 切换到生成环境账户的 CA-TEST角色；
  - IAM --> 用户 --> 添加内联策略
- 切换角色：输入地址
  - 登录zhangsan --> 执行切换链接 --> 即可切换到生产环境账户 --> 测试 S3 完全访问
  - 后续切换：右上角 --> 角色历史记录



### 日志账户

> Logging Account Architecture

- 将所有账户的所需日志**集中存储**在一个中心区域；

<img src="/Users/zhongxwang/Library/Application Support/typora-user-images/image-20210316221535050.png" alt="image-20210316221535050" style="zoom: 50%;" />

### 发布账户

> Publishing Account Structure

- **集中管理**整个企业预先批准的 AMI、aws CloudFormation 模板。

<img src="/Users/zhongxwang/Library/Application Support/typora-user-images/image-20210316221649988.png" alt="image-20210316221649988" style="zoom: 50%;" />



### 账单结构

> Billing Structure

- 建立主账户，**整合支付**所有成员子账户。
- 统一支付、账单追踪、合并使用量

<img src="/Users/zhongxwang/Library/Application Support/typora-user-images/image-20210316221717053.png" alt="image-20210316221717053" style="zoom:50%;" />



### 





## AWS Organizations





# Design for New Solutions

##



# Migration Planning



# Cost Control



# Continuous Improvement for Existing Solutions









