[toc]

https://www.datanucleus.org/products/accessplatform_5_1/jdo/mapping.html



## Classes

- PersistenceCapable
- PersistenceAware
- ReadOnly
  - 不支持修改
  - 是否支持创建？
- Detachable
- SoftDelete

## Inheritance

### new-table

-  each class has its own table in the datastore. 
- 优点：being the most normalised data definition
- 缺点：性能，因为每个表只保存了当前对象的属性，查询 sub-class的时候需要关联所有父类表

### subclass-table

- have its fields persisted in the table of its subclass. 
- 常用在 Abstract Class上，将抽象类的字段存储到子类表。
- 但 孙子类 还是仅包含当前对象的属性

### superclass-table

- have its fields persisted in the table of its superclass.
- 优点：可以一次查出所有子类记录
- 缺点：列数过多
- 配合使用 @Discriminator(value="BOOK") ，会额外增加一列来区分类型

### complete-table

- all classes in an inheritance tree to have their own table containing all fields.
- 用法：定义在 root class上，不用管sub class
  --> 如何处理 抽象类？即不想为 root class创建表 
  --> if any class in the inheritance tree is **abstract** then it won’t have a table since there cannot be any instances of that type 

- 如何查询继承对象

```java
tx.begin();
Extent e = pm.getExtent(com.mydomain.samples.store.Product.class, true); // 注意第二个参数
Query q = pm.newQuery(e);
Collection c=(Collection)q.execute();
tx.commit();
```



## Identity

- 与 PK的关系？

- 作用何在？

- 处理继承关系：
  When you have an inheritance hierarchy, you should specify the identity type in the **base** instantiable class for the inheritance tree. 

### Datastore Identity

- When implementing datastore identity all JDO implementations have to provide a public class that represents this identity.
  --> HOW? https://www.datanucleus.org/products/accessplatform_5_1/extensions/extensions.html#identity 
- Object id = pm.getObjectId(obj);
- Object obj = pm.getObjectById(id);

```java
@PersistenceCapable
@DatastoreIdentity(strategy="sequence", sequence="MY_SEQUENCE")
public class MyClass
```



### Application Identity

- Application identity requires a **primary key class** (unless you have a single primary-key field in which case the PK class is provided for you)
- objectIdClass ： 如果只有一个pk字段，则可以为空 
  --> 起什么作用：定义复合主键
- 查询：getObjectById ：： javax.jdo.identity.LongIdentity id = new javax.jdo.identity.LongIdentity(myClass, 101);
- 查询：如果只有一个主键：Object obj = pm.getObjectById(MyClass.class, mykey);

```java
@PersistenceCapable(objectIdClass=MyIdClass.class)
public class MyClass{
  @Persistent(primaryKey="true")
  private long myPrimaryKeyField;
}
```



### Nondurable Identity

- 无 identity，适合log files 等不需要按 key查询的场景



## Versioning

- 可用于支持 乐观锁
- Strategy 
  - none: 只存储 version number，但不检查乐观锁
  - version-number：从1开始编号
  - data-time：
  - state-image：hashcode

用法：

- 额外 surrogate column

```java
@PersistenceCapable
@Version(strategy=VersionStrategy.VERSION_NUMBER, column="VERSION")
public class MyClass
```

- 用字段

```java
@PersistenceCapable
@Version(strategy=VersionStrategy.VERSION_NUMBER, column="VERSION",
         extensions={@Extension(vendorName="datanucleus", key="field-name", value="myVersion")})
public class MyClass {
    protected long myVersion;
    ...
}
```



## Auditing

- @CreateTimestamp @UpdateTimestamp @CreateUser @UpdateUser
- 如何定义当前用户：datanucleus.CurrentUser
- Q: 无法和 Embbed Object 配合使用？

```java
@PersistenceCapable
public class Hotel
{
    @CreateTimestamp
    Timestamp createTimestamp;

    @CreateUser
    String createUser;

    @UpdateTimestamp
    Timestamp updateTimestamp;

    @UpdateUser
    String updateUser;

    ...
}
```



## Fields / Properties

- Read only

  ```java
      @Extension(vendorName="datanucleus", key="insertable", value="false")
      @Extension(vendorName="datanucleus", key="updateable", value="false")
      String myField;
  
      @ReadOnly
      String myField;
  ```

- non-persistent

  ```java
      @NotPersistent
      String unimportantField;
  ```

  

- Enum 存储

  ```java
  @Extension(vendorName="datanucleus", key="enum-value-getter", value="getValue") // getValue
  MyColour colour;
  ```

- AttributeConverter

  ```java
  public interface AttributeConverter<X,Y>{
      public Y convertToDatastore(X attributeValue);
      public X convertToAttribute (Y datastoreValue);
  }
  
      @Convert(URLStringConverter.class)
      URL url;
  ```

  - 定义默认 converter: javax.jdo.option.typeconverter.{javatype} and the value is the class name of the AttributeConverter.

- TypeConverter
  --> 与 AttributeConverter 什么区别？

  ```java
  @Extension(vendorName="datanucleus", key="type-converter-name", value="kryo-serialise")
  String longString;
  ```

- Column Adapter
  --> 用于执行函数

  ```java
  @Extension(vendorName="datanucleus", key="insert-function", value="TRIM(?)")
  @Extension(vendorName="datanucleus", key="update-function", value="TRIM(?)")
  String myStringField;
  
  @Extension(vendorName="datanucleus", key="select-function", value="UPPER(?)")
  String myStringField;
  ```

  

## Relations

### 1-1

- 数据库层面：For RDBMS a 1-1 relation is stored as a foreign-key column(s). For non-RDBMS it is stored as a String "column" storing the 'id' (possibly with the class-name included in the string) of the related object.
- 代码层面：You cannot have a 1-1 relation to a long or int field! JDO is for use with object-oriented systems, not flat data. 

用法

**1. 单向 Foreign Key**

- Account.USER_ID  外键
- 如果查询时不想带出User: fetch-fk-only=true

```java
public class Account {

    @Column(name="USER_ID")
    User user;
}
```



**2. 单向 JoinTable**

- 会多出一张表 account_user

```java
public class Account{
   
    @Persistent(table="ACCOUNT_USER")
    @Join
    User user;
}
```



**3. 双向 Foreign Key**

- Account.USER_ID  外键
- 表结构和单向 FK 一样？

```java
public class Account {
    @Column(name="USER_ID")
    User user;
}

public class User {
    @Persistent(mappedBy="user")
    Account account;
}
```



### 1-N

#### Collection<PC>

**单向 JoinTable**

- 对象集合
  - 额外的表 ACCOUNT_ADDRESSES

  - ACCOUNT_ADDRESSES.account_id_oid -- join

  - ACCOUNT_ADDRESSES.address_id_eid  -- element

    ```java
    public class Account {
        // 对象集合
        @Persistent(table="ACCOUNT_ADDRESSES")
        @Join(column="ACCOUNT_ID_OID")
        @Element(column="ADDRESS_ID_EID")
        Collection<Address> addresses;
      
        // 排序
        @Order(extensions=@Extension(vendorName="datanucleus", key="list-ordering", value="city ASC"))
    }
    ```

    



**单向 FK**

- Address.account_id

```java
public class Account {
    @Element(column="ACCOUNT_ID")
    Collection<Address> addresses;
}
```



**双向 JoinTable** ?

- 额外的表 ACCOUNT_ADDRESSES
- ACCOUNT_ADDRESSES.account_id_oid -- join
- ACCOUNT_ADDRESSES.address_id_eid  -- element

```java
public class Account {

    @Persistent(mappedBy="account")
    @Join
    Collection<Address> addresses;
}
```



**双向 FK**

- Address.account_id

```java
public class Account {
    @Persistent(mappedBy="account")
    Collection<Address> addresses;
}

public class Address {
    @Column(name="ACCOUNT_ID")
    Account account;
}
```



**共享 JoinTable**

- 多个关联关系共享一个 JoinTable

- JoinTable会再增加一列 ADDRESS_TYPE

  ```java
  public class Account {
  
      @Persistent
      @Join(table="ACCOUNT_ADDRESSES", columns={@Column(name="ACCOUNT_ID_OID")})
      @Element(columns={@Column(name="ADDRESS_ID_EID")})
      // JoinTable会再增加一列 ADDRESS_TYPE
      @SharedRelation(column="ADDRESS_TYPE", value="work") 
      Collection<Address> workAddresses;
  
      @Persistent
      @Join(table="ACCOUNT_ADDRESSES", columns={@Column(name="ACCOUNT_ID_OID")})
      @Element(columns={@Column(name="ADDRESS_ID_EID")})
      // JoinTable会再增加一列 ADDRESS_TYPE
      @SharedRelation(column="ADDRESS_TYPE", value="home") 
      Collection<Address> homeAddresses;
  
      ...
  }
  ```



**共享 FK**

- Address.account_id_oid

- Address.address_type 额外一列

  ```java
  public class Account {
  
      @Persistent
      @SharedRelation(column="ADDRESS_TYPE", value="work")
      Collection<Address> workAddresses;
  
      @Persistent
      @SharedRelation(column="ADDRESS_TYPE", value="home")
      Collection<Address> homeAddresses;
  
      ...
  }
  ```

  





#### 简单对象 Collection<Simple>



**简单对象集合的 JoinTable**

- 使用额外的关联表 直接存储简单对象集合

- ACCOUNT_ADDRESSES.account_id_oid

- ACCOUNT_ADDRESSES.address --> 直接存储，而没有独立的 Address表

  ```java
  public class Account {
      @Persistent
      @Join
      @Element(column="ADDRESS")
      Collection<String> addresses; // String集合
  }
  ```

  

**AtrributeConverter ！！！** 

>  不光可以用在Collection!!

- 将集合转换成一列存储

  ```java
  public class Account {
  
      @Persistent
      @Convert(CollectionStringToStringConverter.class)
      @Column(name="ADDRESSES")
      Collection<String> addresses;
  }
  ```



#### Map<PC, PC> ：不常用

**JoinTable**

- 四个表

  - Account
  - Name
  - Address
  - Account_Addresses: 
    - account_id_oid
    - Name_id_kid
    - Address_id_vid

  ```java
  @PersistenceCapable
  public class Account {
      @Join
      Map<Name, Address> addresses;
  }
  ```



#### Map<Simple, PC> ：不常用

**JoinTable**

- Account_Adresses

  - Account_id_oid
  - Address_id_vid
  - STRING_KEY

  ```java
  public class Account {
      @Join
      Map<String, Address> addresses;
  }
  ```



**单向 FK**

- ADDRESS.account_id_oid

  ```java
  public class Account{
      @Key(mappedBy="alias")
      Map<String, Address> addresses;
  }
  
  public class Address{
      String alias; // Use as key
  }
  ```

  



#### Map<PC, Simple> ：不常用

**JoinTable**

- 表结构类似 Map<Simple, PC>，只不过string_key存储的是值

  ```java
  public class Account{
      @Join
      Map<Address, String> addresses;
  }
  ```



**单向FK**

- ACCOUNT 没有额外字段

- ADDRESS.bus_phone / account_id_oid

  ```java
  public class Account {
     @Value(mappedBy="businessPhoneNumber")
     Map<Address, String> phoneNumbers;
  }
  
  public class Address {
      String businessPhoneNumber; // Use as value
  }
  ```

  



#### Map<Simple, Simple>

**JoinTable**

- 关联表

  - account_id_oid
  - STRING_KEY
  - STRING_VALUE

  ```java
  @PersistenceCapable
  public class Account{
      @Join
      Map<String, String> addresses;
  
  }
  ```



**AttributeConverter**

- Map 转换成String 存储为一列







### N-1

**单向 FK**

- 1 个 User 对应 N 个 Account

- ACCOUNT.user_id

- 表关系和 单向1-1 完全一样 

  ```java
  public class Account {
  
      @Column(name="USER_ID")
      User user;
  }
  ```



**单向 JoinTable**

- Account_User

  - Account_id_oid
  - User_id_eid

  ```java
  public class Account {
  
      @Persistent(table="ACCOUNT_USER")
      @Join
      User user;
  }
  ```



### M-N

**Set**

- JoinTable - Set

```java
public class Product {
    @Persistent(table="PRODUCTS_SUPPLIERS")
    @Join(column="PRODUCT_ID")
    @Element(column="SUPPLIER_ID")
    Set<Supplier> suppliers;
}

public class Supplier {
    @Persistent(mappedBy="suppliers")
    Set<Products> products;
}
```













