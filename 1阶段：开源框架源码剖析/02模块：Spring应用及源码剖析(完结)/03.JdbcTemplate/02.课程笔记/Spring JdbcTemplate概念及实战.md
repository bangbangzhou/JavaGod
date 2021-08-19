# Spring JdbcTemplate概念及实战

# 说在前面

- **本章相关代码及笔记地址**：飞机票🚀
- **🌍**Github：**[**🚀Java超神之路：【🍔Java全生态技术学习笔记，一起超神吧🍔】](https://github.com/EayonLee/JavaGod)
- **🪐**CSDN：**[**🚀Java超神之路：【🍔Java全生态技术学习笔记，一起超神吧🍔】](https://blog.csdn.net/qq_20492277/article/details/114269863)

# 目录
- [Spring AOP  JdbcTemplate概念及实战](#spring-aop--jdbctemplate概念及实战)
- [说在前面](#说在前面)
- [目录](#目录)
- [一. Spring JDBC概述](#一-spring-jdbc概述)
  - [1.1 什么是JDBC Template](#11-什么是jdbc-template)
  - [1.2 Spring JDBC能做什么](#12-spring-jdbc能做什么)
- [二. 使用JdbcTemplate](#二-使用jdbctemplate)
  - [2.1 新建Module，引入依赖](#21-新建module引入依赖)
  - [2.2 配置数据库连接池 / JdbcTemplate](#22-配置数据库连接池--jdbctemplate)
    - [2.2.1 XML方式](#221-xml方式)
    - [2.2.2 配置类方式](#222-配置类方式)
  - [2.3 数据库表](#23-数据库表)
  - [2.4 实体类](#24-实体类)
  - [2.5 Service](#25-service)
  - [2.6 Dao](#26-dao)
    - [2.6.1 Dao接口](#261-dao接口)
    - [2.6.2 Dao实现类](#262-dao实现类)
  - [2.7 测试](#27-测试)
- [三. Spring事务管理](#三-spring事务管理)
  - [3.1 什么是事务？](#31-什么是事务)
    - [3.1.1 事务的定义](#311-事务的定义)
    - [3.1.2 事务的四大特征（ACID）](#312-事务的四大特征acid)
    - [3.1.3 Spring事务管理](#313-spring事务管理)
      - [3.1.3.1 编程式事务管理](#3131-编程式事务管理)
      - [3.1.3.2 声明式事务管理](#3132-声明式事务管理)
  - [3.2 搭建事务操作环境](#32-搭建事务操作环境)
    - [3.2.1 Service](#321-service)
    - [3.2.2 Dao](#322-dao)
    - [3.2.3 测试](#323-测试)
    - [3.2.4 事务问题引入](#324-事务问题引入)
  - [3.3 事务管理](#33-事务管理)
    - [3.3.1 声明式事务管理的实现方式](#331-声明式事务管理的实现方式)
    - [3.3.2 PlatformTransactionManager事务管理器](#332-platformtransactionmanager事务管理器)
    - [3.3.3 声明式事务管理注解方式配置 - XML文件](#333-声明式事务管理注解方式配置---xml文件)
    - [3.3.4 声明式事务管理注解方式配置 - 配置类](#334-声明式事务管理注解方式配置---配置类)
    - [3.3.5 开启事务管理](#335-开启事务管理)
    - [3.3.6 测试](#336-测试)
    - [3.3.7 @Transactional注解中的参数](#337-transactional注解中的参数)
  - [3.4 事务的7中传播行为](#34-事务的7中传播行为)
  - [3.5 事务的隔离级别](#35-事务的隔离级别)
    - [3.5.1 并发情况下事务会出现的问题](#351-并发情况下事务会出现的问题)
    - [3.5.2 隔离级别](#352-隔离级别)


# 一. Spring JDBC概述

## 1.1 什么是JDBC Template

它是 Spring 框架中提供的一个对象，是对原始 Jdbc API 对象的简单封装。除了JdbcTemplate，Spring 框架还为我们提供了很多的操作模板类。

**操作关系型数据的**：JdbcTemplate和HibernateTemplate。

**操作 nosql 数据库的**：RedisTemplate。

**操作消息队列的**：JmsTemplate。

## 1.2 Spring JDBC能做什么

| **操作项目**                 | **Spring帮你做的** | **你需要做的** |
| ---------------------------- | ------------------ | -------------- |
| 定义连接参数                 |                    | ✔              |
| 打开连接                     | ✔                  |                |
| 指定SQL语句                  |                    | ✔              |
| 声明参数并提供参数值         |                    | ✔              |
| 准备并运行该语句             | ✔                  |                |
| 设置循环以遍历结果（如果有） | ✔                  |                |
| 进行每次迭代的工作           |                    | ✔              |
| 处理任何异常                 | ✔                  |                |
| 处理交易                     | ✔                  |                |
| 关闭连接，语句和结果集       | ✔                  |                |



# 二. 使用JdbcTemplate

## 2.1 新建Module，引入依赖

![image-20210819150246938](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819150246938.png)

## 2.2 配置数据库连接池 / JdbcTemplate

### 2.2.1 XML方式

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <!--开启组件扫描   扫描com.eayon包下所有带有注解（@Component @Service等）的类 然后去实例化-->
    <context:component-scan base-package="com.eayon"/>

    <!-- 数据库连接池 -->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"
          destroy-method="close">
        <property name="url" value="jdbc:mysql:///user_db"/>
        <property name="username" value="root"/>
        <property name="password" value="1234"/>
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    </bean>

    <!--创建jdbcTemplate对象-->
    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <!--注入数据源-->
        <property name="dataSource" ref="dataSource"/>
    </bean> 
</beans>
```

### 2.2.2 配置类方式

```java
@Configuration//声明当前类是配置类并加载到IOC容器
@ComponentScan(basePackages = {"com.eayon"})//开启组件扫描   扫描com.eayon包下所有带有注解（@Component @Service等）的类 然后去实例化
public class JdbcConfiguration {

    /**
     * 数据库连接池
     *
     * @return
     */
    @Bean
    public DriverManagerDataSource getDataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql:///user_db");
        dataSource.setUsername("root");
        dataSource.setPassword("1234");
        return dataSource;
    }

    /**
     * jdbctemplate
     */
    @Bean
    public JdbcTemplate getJdbcTemplate(){
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        jdbcTemplate.setDataSource(getDataSource());
        return jdbcTemplate;
    }
}
```

## 2.3 数据库表

![image-20210819150219314](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819150219314.png)

## 2.4 实体类

```java
package com.eayon.entity;

public class User {
    private Integer uid;
    private String username;
    private Double money;

    .....省略构造方法及get/set方法....
}
```

## 2.5 Service

```java
@Service
public class UserService {

    @Autowired
    private UserDao userDao;

    public User getById(Integer uid) {
        return userDao.getById(uid);
    }

    public List<User> getAll() {
        return userDao.getAll();
    }

    public void add(User user){
        userDao.add(user);
    }

    public void update(User user){
        userDao.update(user);
    }

    public void delete(Integer uid){
        userDao.delete(uid);
    }
}
```

## 2.6 Dao

### 2.6.1 Dao接口

```java
public interface UserDao {

    User getById(Integer uid);

    List<User> getAll();

    void add(User user);

    void update(User user);

    void delete(Integer uid);
}
```

### 2.6.2 Dao实现类

```java
@Repository//实例化到Spring容器
public class UserDaoImpl implements UserDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public User getById(Integer uid) {
        //第一个参数：sql语句
        String sql = "select * from user where uid = ?";
        //第二个参数：查询结果返回对象/集合时通过RowMapper的实现类完成数据的封装
        BeanPropertyRowMapper<User> rowMapper = new BeanPropertyRowMapper<>(User.class);
        //第三个参数：sql中占位符的值
        User user = jdbcTemplate.queryForObject(sql, rowMapper, uid);
        return user;
    }

    @Override
    public List<User> getAll() {
        //第一个参数：sql语句
        String sql = "select * from user";
        //第二个参数：查询结果返回对象/集合时通过RowMapper的实现类完成数据的封装
        BeanPropertyRowMapper<User> rowMapper = new BeanPropertyRowMapper<User>(User.class);
        List<User> users = jdbcTemplate.query(sql, rowMapper);
        return users;
    }

    @Override
    public void add(User user) {
        Object[] args = {user.getUid(), user.getUsername(), user.getMoney()};
        String sql = "insert into user values(?,?,?)";
        int add = jdbcTemplate.update(sql, args);
        System.out.println("add success num:" + add);
    }

    @Override
    public void update(User user) {
        Object[] args = {user.getUsername(), user.getUid(), user.getMoney()};
        String sql = "update user set username = ? and money = ? where uid = ?";
        int update = jdbcTemplate.update(sql, args);
        System.out.println("update success num:" + update);
    }

    @Override
    public void delete(Integer uid) {
        Object[] args = {uid};
        String sql = "delete from user where uid = ?";
        int delete = jdbcTemplate.update(sql, args);
        System.out.println("delete success num:" + delete);
    }
}
```

## 2.7 测试

```java
public class JdbcDemo {

    @Test
    public void getById(){
        ApplicationContext ac = new AnnotationConfigApplicationContext(JdbcConfiguration.class);//加载配置类
        //ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");//加载配置文件
        UserService userService = ac.getBean("userService", UserService.class);
        User user = userService.getById(1);
        System.out.println(user);
    }

    @Test
    public void getAll(){
        ApplicationContext ac = new AnnotationConfigApplicationContext(JdbcConfiguration.class);//加载配置类
        //ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");//加载配置文件
        UserService userService = ac.getBean("userService", UserService.class);
        List<User> users = userService.getAll();
        System.out.println(users);
    }

    @Test
    public void add(){
        ApplicationContext ac = new AnnotationConfigApplicationContext(JdbcConfiguration.class);//加载配置类
        //ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");//加载配置文件
        UserService userService = ac.getBean("userService", UserService.class);
        User user = new User();
        user.setUid(2);
        user.setUsername("李四");
        user.setMoney(100.00);
        userService.add(user);
    }

    @Test
    public void update(){
        ApplicationContext ac = new AnnotationConfigApplicationContext(JdbcConfiguration.class);//加载配置类
        //ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");//加载配置文件
        UserService userService = ac.getBean("userService", UserService.class);
        User user = new User();
        user.setUid(2);
        user.setUsername("王五");
        user.setMoney(100.00);
        userService.update(user);
    }

    @Test
    public void delete(){
        ApplicationContext ac = new AnnotationConfigApplicationContext(JdbcConfiguration.class);//加载配置类
        //ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");//加载配置文件
        UserService userService = ac.getBean("userService", UserService.class);
        userService.delete(2);
    }
}
```



# 三. Spring事务管理

## 3.1 什么是事务？

### 3.1.1 事务的定义

事务是数据库操作最基本单元，逻辑上一组操作，要么都成功，如果有一个失败所有操作都失败。

**概述**

一个事务是由一个单元内的一个或多个SQL（操作）组成，这个单元中的每个SQL（操作）都是互相依赖的，单元作为一个整体是不可分割的。如果单元内的一个SQL不能够成功完成，整个单元就会回滚，所影响到的数据将返回到事务开始以前的状态。因此，只有事务中的所有操作都被执行成功，才能说是一个事务被执行成功

**举例**

![image-20210819150447356](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819150447356.png)

如上述例子，Micah给Maruko转账，只有在转账成功的情况下，Micah的账户余额才会减少，Maruko的账户余额增加，不存在Micah账户的余额减少了，而Maruko的账户余额却不变。要么转账成功，2边余额都改变；要么转账失败，2边余额都保持不变。

### 3.1.2 事务的四大特征（ACID）

- **原子性（Atomicity）**：事务中所有的操作都捆绑成一个原子弹元，所以对于事物所进行修改等操作时，要么全部执行，要么全部不执行。
- **一致性（Consistency）**：事务在完成时，必须所有的数据都保持一致。
- **隔离性（Isolation）**：事务与事务之间式是隔离开的，事务只缺提交之前，它的结果不应该显示给其他事务。
- **持久性（Durability）**：事务正确提交之后，结果将永远保存在数据库之中。

### 3.1.3 Spring事务管理

#### 3.1.3.1 编程式事务管理

编程式事务管理是侵入性事务管理，使用TransactionTemplate或者直接使用PlatformTransactionManager，对于编程式事务管理，Spring推荐使用TransactionTemplate。

#### 3.1.3.2 声明式事务管理

声明式事务管理建立在AOP之上，其本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，执行完目标方法之后根据执行的情况提交或者回滚。编程式事务每次实现都要单独实现，但业务量大功能复杂时，使用编程式事务无疑是痛苦的，而声明式事务不同，声明式事务属于无侵入式，不会影响业务逻辑的实现，只需要在配置文件中做相关的事务规则声明或者通过注解的方式，便可以将事务规则应用到业务逻辑中。

显然声明式事务管理要优于编程式事务管理，这正是Spring倡导的非侵入式的编程方式。唯一不足的地方就是声明式事务管理的粒度是方法级别，而编程式事务管理是可以到代码块的，但是可以通过提取方法的方式完成声明式事务管理的配置。

## 3.2 搭建事务操作环境

![image-20210819150620214](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819150620214.png)

### 3.2.1 Service

```java
@Service
public class UserService {

    @Autowired
    private UserDao userDao;

    /**
     * 转账
     *
     * @param reduceUid     扣款用户
     * @param addUid        收钱用户
     * @param transferMoney 转账金额
     */
    public void transfer_accounts(Integer reduceUid, Integer addUid, Double transferMoney) {
        //扣钱
        userDao.reduceMoney(reduceUid, transferMoney);
        //加钱
        userDao.addMoney(addUid, transferMoney);
    }
}
```

### 3.2.2 Dao

**Dao接口**

```java
public interface UserDao {
    void reduceMoney(Integer uid, Double money);

    void addMoney(Integer uid, Double money);
}
```

**Dao实现类**

```java
@Repository//实例化到Spring容器
public class UserDaoImpl implements UserDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * 扣钱
     *
     * @param uid
     * @param money
     */
    @Override
    public void reduceMoney(Integer uid, Double money) {
        String sql = "update user set money = money - ? where uid = ?";
        jdbcTemplate.update(sql, money, uid);
    }

    /**
     * 加钱
     * @param uid
     * @param money
     */
    @Override
    public void addMoney(Integer uid, Double money) {
        String sql = "update user set money = money + ? where uid = ?";
        jdbcTemplate.update(sql, money, uid);
    }
}
```

### 3.2.3 测试

```java
public class JdbcDemo {
    /**
     * 转账【事务管理】
     */
    @Test
    public void transfer_accounts() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(JdbcConfiguration.class);//加载配置类
        //ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");//加载配置文件
        UserService userService = ac.getBean("userService", UserService.class);

        //扣钱
        Integer reduceUid = 1;//扣款用户uid
        Integer addUid = 2;//加钱用户uid
        Double transferMoney = 50.00;//转账金额
        userService.transfer_accounts(reduceUid, addUid, transferMoney);
    }
}
```

### 3.2.4 事务问题引入

如上代码可以正确进行转账，但是如果我们在service层给用户成功扣款后出现了异常导致程序执行中断，没有给用户完成加钱操作，则就出现了事务的问题

![image-20210819150753019](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819150753019.png)

## 3.3 事务管理

在 Spring 进行事务管理操作，有两种方式：**编程式事务管理**和**声明式事务管理（使用）**

### 3.3.1 声明式事务管理的实现方式

声明式事务管理基于AOP

- 基于注解方式（使用）
- 基于xml配置方式

### 3.3.2 PlatformTransactionManager事务管理器

它是一个接口，这个接口针对不同的框架提供了不同事务控制的实现类

![image-20210819150849933](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819150849933.png)

### 3.3.3 声明式事务管理注解方式配置 - XML文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">


    <!--开启组件扫描   扫描com.eayon包下所有带有注解（@Component @Service等）的类 然后去实例化-->
    <context:component-scan base-package="com.eayon"/>

    <!-- 数据库连接池 -->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"
          destroy-method="close">
        <property name="url" value="jdbc:mysql:///user_db"/>
        <property name="username" value="root"/>
        <property name="password" value="1234"/>
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    </bean>

    <!--创建jdbcTemplate对象-->
    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <!--注入数据源-->
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!--创建事务管理器-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!--注入数据源-->
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!--开启事务注解-->
    <tx:annotation-driven transaction-manager="transactionManager"/>
</beans>
```

### 3.3.4 声明式事务管理注解方式配置 - 配置类

```java
@Configuration//声明当前类是配置类并加载到IOC容器
@ComponentScan(basePackages = {"com.eayon"})//开启组件扫描   扫描com.eayon包下所有带有注解（@Component @Service等）的类 然后去实例化
@EnableTransactionManagement//开启事务管
public class JdbcConfiguration {

    /**
     * 数据库连接池
     *
     * @return
     */
    @Bean
    public DriverManagerDataSource getDataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql:///user_db");
        dataSource.setUsername("root");
        dataSource.setPassword("1234");
        return dataSource;
    }

    /**
     * jdbctemplate
     */
    @Bean
    public JdbcTemplate getJdbcTemplate(){
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        jdbcTemplate.setDataSource(getDataSource());
        return jdbcTemplate;
    }

    /**
     * 创建事务管理器
     */
    @Bean
    public DataSourceTransactionManager getDataSourceTransactionManager(DataSource dataSource){
        DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
        transactionManager.setDataSource(dataSource);
        return transactionManager;
    }
}
```

### 3.3.5 开启事务管理

**我们可以在需要进行事务控制的service类加上``@Transactional``注解，这样该类下所有方法都会进行事务控制**

```java
@Service
@Transactional//事务
public class UserService {

    @Autowired
    private UserDao userDao;

    /**
     * 转账
     *
     * @param reduceUid     扣款用户
     * @param addUid        收钱用户
     * @param transferMoney 转账金额
     */
    public void transfer_accounts(Integer reduceUid, Integer addUid, Double transferMoney) {
        //扣钱
        userDao.reduceMoney(reduceUid, transferMoney);

        //模拟异常
        int i = 10 / 0;

        //加钱
        userDao.addMoney(addUid, transferMoney);
    }
}
```

**也可以精确到方法上就只针对与该方法进行事务控制**

```java
@Service
public class UserService {

    @Autowired
    private UserDao userDao;

    /**
     * 转账
     *
     * @param reduceUid     扣款用户
     * @param addUid        收钱用户
     * @param transferMoney 转账金额
     */
     @Transactional//事务
    public void transfer_accounts(Integer reduceUid, Integer addUid, Double transferMoney) {
        //扣钱
        userDao.reduceMoney(reduceUid, transferMoney);

        //模拟异常
        int i = 10 / 0;

        //加钱
        userDao.addMoney(addUid, transferMoney);
    }
}
```

### 3.3.6 测试

测试转账后出现异常时，被扣款账户的金额是否还会成功扣除

```java
/**
 * 转账【事务管理】
 */
@Test
public void transfer_accounts() {
    ApplicationContext ac = new AnnotationConfigApplicationContext(JdbcConfiguration.class);//加载配置类
    //ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");//加载配置文件
    UserService userService = ac.getBean("userService", UserService.class);

    //扣钱
    Integer reduceUid = 1;//扣款用户uid
    Integer addUid = 2;//加钱用户uid
    Double transferMoney = 50.00;//转账金额
    userService.transfer_accounts(reduceUid, addUid, transferMoney);
}
```

### 3.3.7 @Transactional注解中的参数

![image-20210819151116972](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819151116972.png)

- **propagation（事务传播行为）**：多个事务方法调用时，事务是如何执行的。

- **isolation（事务隔离级别）**：在并发情况下事务会出现一些脏读、幻读、不可重复读的问题，所以我们可以通过事物的隔离级别进行处理。

- **timeout（超时时间）**：事务需要在一定时间内进行提交，如果不提交进行回滚，默认值是 -1（无时间限制），设置时间以秒单位进行计算。

- **readOnly（是否只读）**：

- - readOnly默认值 false，表示可以查询，可以添加修改删除操作
  - 设置 readOnly值是 true，设置成 true之后，只能查询

- **rollbackFor（发生什么异常会去回滚）**：设置出现哪些异常进行事务回滚

- **noRollbackFor（发生什么异常不会回滚）**：设置出现哪些异常不进行事务回滚

## 3.4 事务的7中传播行为

事务的传播性一般用在事务嵌套的场景，比如一个事务方法里面调用了另外一个事务方法，那么两个方法是各自作为独立的方法提交还是内层的事务合并到外层的事务一起提交，这就是需要事务传播机制的配置来确定怎么样执行。

**事务的传播行为有以下七种**：

- **PROPAGATION_REQUIRED（****默认****）**：Spring默认的传播机制，能满足绝大部分业务需求，如果外层有事务则当前事务加入到外层事务一块提交回滚。如果外层没有事务则新建一个事务执行。
- **PROPAGATION_REQUES_NEW**：当前的方法每次都会新建一个事务，并在自己的事务内运行，如果有事务正在运行就将他挂起。
- **PROPAGATION_SUPPORT**：如果外层有事务，则加入外层事务，如果外层没有事务，则直接使用非事务方式执行。完全依赖外层的事务
- **PROPAGATION_NOT_SUPPORT：**该传播机制不支持事务，如果外层存在事务则挂起，执行完当前代码，则恢复外层事务，无论是否异常都不会回滚当前的代码
- **PROPAGATION_NEVER**：该传播机制不支持外层事务，即如果外层有事务就抛出异常
- **PROPAGATION_MANDATORY**：与NEVER相反，如果外层没有事务，则抛出异常
- **PROPAGATION_NESTED：**如果当前存在事务，就嵌套在当前事务内执行，如果没有就新建一个事务。

**传播规则回答了这样一个问题：一个新的事务应该被启动还是被挂起，或者是一个方法是否应该在事务性上下文中运行。**

## 3.5 事务的隔离级别

在并发情况下事务会出现一些如下2.5.1读的问题，所以我们可以通过事务的隔离级别去进行处理。可以把事务的隔离级别想象为这个事务对于事物处理数据的自私程度。

### 3.5.1 并发情况下事务会出现的问题

在一个典型的应用程序中，多个事务同时运行，经常会为了完成他们的工作而操作同一个数据。并发虽然是必需的，但是会导致以下问题：

- **脏读**：一个事务读取到另一个事务还没有提交的数据。如果这些数据稍后被回滚了，那么第一个事务读取到的数据是无效的，则为脏读。

- **不可重复读**：一个事务两个相同的查询却返回了不同的数据。通常是由于另一个并发事务在两次查询之间更新了数据。不可重复读的重点在修改

- **幻读**：和不可重复读类似。当一个事务第一次读取几行数据之后由于另一个并发事务往里面插入了一些数据。幻读就发生了。在后面的查询中，第一个事务会发现原来没有查询到的数据。幻读重点在新增或删除。

### 3.5.2 隔离级别

在理想状态下，事务之间将完全隔离，从而可以防止这些问题发生。然而，完全隔离会影响性能，因为隔离经常涉及到锁定在数据库中的记录（甚至有时是锁表）。完全隔离要求事务相互等待来完成工作，会阻碍并发。因此，可以根据业务场景选择不同的隔离级别

- **读操作未提交（ISOLATION_READ_UNCOMMITTED）**：允许读取尚未提交的更改。无法避免脏读、幻读、 不可重复读
- **读操作已提交（ISOLATION_READ_COMMITTED）**：Oracle默认级别。允许从已经提交的并发事务读取。可以避免脏读，但无法避免幻读和不可重复读。
- **可重复读（ISOLATION_REPEATABLE_READ）**：MySql默认隔离级别。可以保证多次读取的结果一致。除非数据被当前事务本身改变。可避免脏读，不可重复读，不可避免幻读。
- **串行化（ISOLATION_SERIALIZABLE）**：完全服从ACID的隔离级别。能够保证任何并发情况下的问题发生（脏读、幻读、不可重复读）但是效率很低，因为它通常是通过完全锁定当前事务所涉及的数据表来完成的。

![image-20210819151408747](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819151408747.png)