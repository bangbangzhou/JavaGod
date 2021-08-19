# Spring AOP 概念及实战

# 说在前面

- **本章相关代码及笔记地址**：飞机票🚀
- **🌍**Github：**[**🚀Java超神之路：【🍔Java全生态技术学习笔记，一起超神吧🍔】](https://github.com/EayonLee/JavaGod)
- **🪐**CSDN：**[**🚀Java超神之路：【🍔Java全生态技术学习笔记，一起超神吧🍔】](https://blog.csdn.net/qq_20492277/article/details/114269863)

# 目录
- [Spring AOP 概念及实战](#spring-aop-概念及实战)
- [说在前面](#说在前面)
- [目录](#目录)
- [一. AOP基础概念](#一-aop基础概念)
  - [1.1 AOP?](#11-aop)
  - [1.2 AOP中的一些常用概念](#12-aop中的一些常用概念)
  - [1.3 通知类型](#13-通知类型)
- [二. AOP底层实现](#二-aop底层实现)
  - [2.1 AOP底层原理](#21-aop底层原理)
  - [2.2 代理概述](#22-代理概述)
  - [2.3 静态代理实现](#23-静态代理实现)
  - [2.3 JDK动态代理实现](#23-jdk动态代理实现)
- [三. Spring AOP的操作](#三-spring-aop的操作)
  - [3.1 什么是AspectJ？](#31-什么是aspectj)
    - [3.1.1 在Spring框架中，一般都是基于AspectJ来实现AOP的相关操作](#311-在spring框架中一般都是基于aspectj来实现aop的相关操作)
    - [3.1.2 基于AspectJ实现AOP操作](#312-基于aspectj实现aop操作)
  - [3.2 准备工作](#32-准备工作)
    - [3.2.1 引入Spring AOP相关依赖](#321-引入spring-aop相关依赖)
    - [3.2.2 切入点表达式了解](#322-切入点表达式了解)
  - [3.3 AOP操作 - 基于AspectJ注解](#33-aop操作---基于aspectj注解)
    - [3.3.1 创建被增强类及方法](#331-创建被增强类及方法)
    - [3.3.2 创建切面类（写方法增强逻辑的地方）](#332-创建切面类写方法增强逻辑的地方)
    - [3.3.3 加载bean到Spring容器并开启Aspect](#333-加载bean到spring容器并开启aspect)
    - [3.3.4 测试机结果](#334-测试机结果)
    - [3.3.5 通知的执行顺序](#335-通知的执行顺序)
    - [3.3.6 设置增强类的加载优先级](#336-设置增强类的加载优先级)

# 一. AOP基础概念

## 1.1 AOP?

**AOP面向切面编程**，可以``不修改源代码进行方法增强``，AOP是OOP（面向对象编程）的延续，主要用于日志记录、性能统计、安全控制、事务处理等方面。它是基于``代理设计模式``，而代理设计模式又分为静态代理和``动态代理``，静态代理比较简单就是一个接口，分别由一个真实实现和一个代理实现，而动态代理分为基于接口的``JDK动态代理``和基于类的``cglib的动态代理``，咱们正常都是面向接口开发，所以AOP使用的是基于接口的JDK动态代理。

## 1.2 AOP中的一些常用概念

- **切面(Aspect)**：AOP核心就是切面，它将多个类的通用行为封装成可重用的模块，该模块含有一组API提供横切功能。比如，一个日志模块可以被称作日志的AOP切面。根据需求的不同，一个应用程序可以有若干切面。在Spring AOP中，切面通过带有@Aspect注解的类实现。
- **连接点(Join Point)**：哪些方法需要被AOP增强，这些方法就叫做连接点。
- **通知(Advice)**：AOP在特定的切入点上执行的增强处理，有``before``, ``after``,`` afterReturning``, ``afterThrowing``, ``around``
- **切入点（Pointcut）**：实际真正被增强的方法，称为切入点。

## 1.3 通知类型

通知(advice)是你在你的程序中想要应用在其他模块中的横切关注点的实现。Advice主要有以下5种类型：

- **前置通知(Before Advice)**：在连接点之前执行的Advice，不过除非它抛出异常，否则没有能力中断执行流。使用``@Before`` 注解使用这个Advice。

- **返回之后通知(After Retuning Advice)**：在连接点正常结束之后执行的Advice。例如，如果一个方法没有抛出异常正常返回。通过 ``@AfterReturning`` 关注使用它。

- **抛出（异常）后执行通知(After Throwing Advice)**：如果一个方法通过抛出异常来退出的话，这个Advice就会被执行。通过 ``@AfterThrowing`` 注解来使用。

- **后置通知(After Advice)**：无论连接点是通过什么方式退出的(正常返回或者抛出异常)都会执行在结束后执行这些Advice。通过 ``@After`` 注解使用。

- **围绕通知(Around Advice)**：围绕连接点执行的Advice，就你一个方法调用。这是最强大的Advice。通过``@Around`` 注解使用。



# 二. AOP底层实现

## 2.1 AOP底层原理

它是基于代理设计模式，而代理设计模式又分为静态代理和动态代理，静态代理比较简单就是一个接口，分别由一个真实实现和一个代理实现，而动态代理分为基于接口的JDK动态代理和基于类的CGLIB的动态代理。

**第一种 有接口情况，使用 JDK 动态代理**

创建接口实现类代理对象，增强类的方法

![image-20210819142955341](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819142955341.png)

JDK动态代理是去创建一个UserDao接口的实现类的代理对象，该接口实现类的代理对象会调用该接口的真实实现，并且在代理对象中调用真实实现类的前后做方法增强

**第二种 没有接口情况，使用 CGLIB 动态代理**

创建子类的代理对象，增强类的方法

![image-20210819143018532](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819143018532.png)

CGLIB动态代理是去创建一个User类子类的代理对象，该子类的代理对象会去调用父类User中的方法，并且在子类代理对象调用其父类方法签后去做增强

## 2.2 代理概述

代理就是在不修改源代码的情况下使得原本不具备某种行为能力的类、对象具有该种行为能力，实现对目标对象的功能扩展

**代理的应用场景**

- 事务处理
- 权限管理
- 日志收集
- AOP切面
- .........

**Java的代理分为静态代理和动态代理**

静态代理的局限性：只能代理某一类型接口的实例，不能代理任意接口任意方法的操作。

静态代理只能代理固定或单一接口的方法，也就是说不能做到任何类任何方法的代理。

## 2.3 静态代理实现

**Movie 接口的实现**

```java
/**
 * 委托类的父接口
 */
public interface Movie {
    void player();
}
```

**实现了Movie 接口的 真实实现类（委托类）**：

```java
public class RealMovie implements Movie {
    @Override
    public void player() {
         System.out.println(">>>>>>>> 您正在观看《士兵突击》");
    }
}
```

**实现了Movie 接口的 代理实现类**：

```java
public class Cinema implements Movie {

    RealMovie realMovie;

    public Cinema(RealMovie realmovie) {
        this.realMovie = realmovie;
    }
    
    public void player() {
        //对目标方法进行方法增强
        System.out.println("|||||||||||||||||||||||电影开始前,卖爆米花");
        
        //执行真实实现的目标方法
        realMovie.player();
        
        //对目标方法进行方法增强
        System.out.println("----------------------电影结束了，打扫卫生");
    }
    
    
}
```

**具体的调用如下**：

```java
public class ProxyTest {
    public static void main(String[] args) {
        //创建电影院（静态代理）
        Cinema cinema = new Cinema(new RealMovie());
        cinema.player();
    }
}
```

**使用静态代理的好处：**

使得真实角色处理的业务更加纯粹，不再去关注一些公共的事情。

公共的业务由代理来完成---实现业务的分工。

公共业务发生扩展时变得更加集中和方便。

**缺点**：每一个代理类都必须实现一遍真实 实现类（也就是realMovie）的接口，如果接口增加方法，则代理类也必须跟着修改。其次，每一个代理类对应一个真实实现类（委托类），如果真实实现（委托类）非常多，则静态代理类就非常臃肿，难以胜任。

## 2.3 JDK动态代理实现

动态代理有别于静态代理，是根据代理的对象，动态创建代理类。这样，就可以避免静态代理中代理类接口过多的问题。动态代理是通过反射来实现的，借助Java自带的java.lang.reflect.Proxy,通过固定的规则生成。

其步骤如下：

1. 创建一个需要动态代理的接口，即Movie接口
2. 创建一个需要动态代理接口的真实实现，即RealMovie类
3. 创建一个动态代理处理器，实现``InvocationHandler接口``，并重写invoke方法去增强真实实现中的目标方法
4. 在测试类中，生成动态代理的对象。

**第一二步骤，和静态代理一样，不过说了。第三步，代码如下**：

```java
/**
 * 动态代理处理类
 */
public class MyInvocationHandler implements InvocationHandler {

    //需要动态代理接口的真实实现类
    private Object object;

    //通过构造方法去给需要动态代理接口的真实实现类赋值
    public MyInvocationHandler(Object object) {
        this.object = object;
    }


    /**
     * 对真实实现（被代理对象）的目标方法进行增强
     * 当代理对象调用真实实现类的方法时，就会执行动态代理处理器中的该invoke方法
     *
     * @param proxy  生成的代理对象
     * @param method 代理对象调用的方法
     * @param args   调用的方法中的参数
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //方法增强
        System.out.println("卖爆米花");

        //object是真实实现，args是调用方法的参数
        //当代理对象调用真实实现的方法，那么这里就会将真实实现和方法参数传递过去，去调用真实实现的方法
        method.invoke(object,args);

        //方法增强
        System.out.println("扫地");
        return null;
    }
}
```

**第四步，创建动态代理的对象**

```java
public class DynamicProxyTest {
    
    public static void main(String[] args) {
        // 保存生成的代理类的字节码文件
        //由于设置sun.misc.ProxyGenerator.saveGeneratedFiles 的值为true,所以代理类的字节码内容保存在了项目根目录下，文件名为$Proxy0.class
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");

        //需要动态代理接口的真实实现
        RealMovie realMovie = new RealMovie();
        //动态代理处理类
        MyInvocationHandler handler = new MyInvocationHandler(realMovie);
        //获取动态代理对象
        //第一个参数：真实实现（被代理对象）的类加载器
        //第二个参数：真实实现类（被代理对象）它所实现的所有接口的数组
        //第三个参数：动态代理处理器
        Movie movie = (Movie) Proxy.newProxyInstance(realMovie.getClass().getClassLoader(),
                realMovie.getClass().getInterfaces(),
                handler);
        movie.player();
    }

}
```

**结果**

![image-20210819143321623](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819143321623.png)

由于设置 sun.misc.ProxyGenerator.saveGeneratedFiles 的值为true,所以代理类的字节码内容保存在了项目根目录下，文件名为$Proxy0.class

![image-20210819143343829](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819143343829.png)



# 三. Spring AOP的操作

## 3.1 什么是AspectJ？

### 3.1.1 在Spring框架中，一般都是基于AspectJ来实现AOP的相关操作

AspectJ并不是Spring的组成部分，它是独立的AOP框架（不需要Spring也能独立使用），

所以我们一般把AspectJ和Spring框架一起使用，去进行一些AOP操作。

### 3.1.2 基于AspectJ实现AOP操作

- 基于XML配置文件实现
- 基于注解方式实现（常用、方便）

## 3.2 准备工作

### 3.2.1 引入Spring AOP相关依赖

在原有的依赖的基础上添加jar包（包括AspectJ）

![image-20210819143538837](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819143538837.png)

### 3.2.2 切入点表达式了解

**切入点表达式的作用**：用于表达对哪个类里面的哪个方法（切入点）进行增强

**语法结构**：

```java
举例 1：对 com.eayon.dao.BookDao 类里面的 add 进行增强 execution(* com.dao.dao.BookDao.add(..))
举例 2：对 com.eayon.dao.BookDao 类里面的所有的方法进行增强execution(* com.dao.dao.BookDao.* (..))
举例 3：对 com.eayon.dao 包里面所有类，类里面所有方法进行增强execution(* com.dao.dao.*.* (..))
```

**PS**：上面举例三个都是去切具体的某个方法、类。我们也可以去切到某个包下所有的方法，也可以去切某包下带有某注解的方法等等。

**常见的切入点表达式示例**

- 所有方法执行

  ```java
  execution(public * *(..))
  ```

- 名称以"set"开头的所有方法执行

  ```java
  execution(* set*(..))
  ```

- AccountService接口中的所有方法执行

  ```java
  execution(* com.eayon.service.AccountService.*(..))
  ```

- service包下所有方法执行

  ```java
  execution(* com.eayon.service..*.*(..))
  ```

- service包下的所有连接点（仅在Spring AOP中执行方法）

  ```java
  within(com.eayon.service..*)
  ```

- 代理实现AccountService接口的任何连接点（仅在Spring AOP中执行方法）

  ```java
  this(com.eayon.service.AccountService)
  ```

- 所有带有@checkLogin注解的方法或类

  ```java
  @annotation(com.eayon.annotation.checkLogin)
  ```

## 3.3 AOP操作 - 基于AspectJ注解

### 3.3.1 创建被增强类及方法

```java
package com.eayon.aop;
import org.springframework.stereotype.Component;

/**
 * @Description AOP被增强类
 */
@Component//通过IOC中的注解将该类实例化到Spring容器
public class Car {

    //汽车前进方法
    public void forward(String carName){
        System.out.println(carName + "牌汽车前进了");
    }

    //汽车后退方法
    public void backoff(String carName){
        System.out.println(carName + "牌汽车后退了");
    }
}
```

### 3.3.2 创建切面类（写方法增强逻辑的地方）

```java
package com.eayon.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component// 通过IOC中的注解将该类实例化到Spring容器
@Aspect// 声明切面类，并为本类生成代理对象
public class CarAspect {

    /**
     * 相同切入点抽取
     */
    @Pointcut(value = "execution(* com.eayon.aop.Car.*(..))")
    public void pointCut(){

    }

    /**
     * 前置通知
     * @Before注解表示作为前置通知
     */
    //@@Before(value = "execution(* com.eayon.aop.Car.*(..))")
    @Before(value = "pointCut()")//抽取切入点
    public void before(){
        System.out.println("Before...");
    }

    /**
     * 后置通知（返回通知）
     */
    //@AfterReturning
    @AfterReturning(value = "pointCut()")
    public void afterReturning(){
        System.out.println("AfterReturning...");
    }

    /**
     * 异常通知
     */
    //
    @AfterThrowing(value = "pointCut()")
    public void afterThrowing(){
        System.out.println("AfterThrowing...");
    }

    /**
     * 最终通知
     */
    //@After
    @After(value = "pointCut()")
    public void after(){
        System.out.println("After...");
    }


    /**
     * @Around 代表环绕通知  value代表切入点，即Car类中的所有方法
     * 同理 你想用其他通知只需要变更注解就可以了 value都是一个意思
     * @Before 前置通知注解
     * @After 后置通知注解
     * @AfterThrowing 抛出（异常）后执行通知注解
     * @AfterReturning 返回之后通知注解
     *
     * @return
     */
    //@Around
    @Around(value = "pointCut()")
    public Object before(ProceedingJoinPoint point) throws Throwable {
        //获取切点方法上的参数
        Object[] args = point.getArgs();

        //进行方法增强  修改参数值
        if(null != args && args.length > 0){
            //原来的参数值
            Object carName = args[0];
            System.out.println("Around 原来的参数值" + carName);

            //更换参数
            args[0] = "奔驰";
        }

        //继续执行切点方法 并使用更换后的参数
        return point.proceed(args);
    }
}
```

### 3.3.3 加载bean到Spring容器并开启Aspect

**XML方式**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
                        http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">
    <!--开启组件扫描   扫描com.eayon包下所有带有注解（@Component @Service等）的类 然后去实例化-->
    <context:component-scan base-package="com.eayon"></context:component-scan>

    <!-- 开启Aspect生成代理对象  也就是扫描带有@Aspect注解的类并生成代理对象-->
    <aop:aspectj-autoproxy></aop:aspectj-autoproxy>
</beans>
```

**配置类方式**

```java
package com.eayon.conf;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

/**
 * @Description Spring配置类 用于扫描带有注解的类将其实例化到IOC容器
 * 开启Aspect生成代理对象  也就是扫描带有@Aspect注解的类并生成代理对象
 */
@Configuration//声明当前类是配置类并加载到IOC容器
@ComponentScan(basePackages = {"com.eayon"})//开启组件扫描   扫描com.eayon包下所有带有注解（@Component @Service等）的类 然后去实例化
@EnableAspectJAutoProxy(proxyTargetClass = true)//开启Aspect生成代理对象  也就是扫描带有@Aspect注解的类并生成代理对象
public class SpringConf {
    
}
```

### 3.3.4 测试机结果

```java
package com.eayon.demo;

public class AopTest {

    @Test
    public void test_car_aop() {
        // 加载配置类实例化所有bean、并开启Aspect生成代理对象  也就是扫描带有@Aspect注解的类并生成代理对象
        // ApplicationContext context = new ClassPathXmlApplicationContext("spring_conf.xml");//加载配置文件，效果一样
        ApplicationContext context = new AnnotationConfigApplicationContext(SpringConf.class);//加载配置类，效果一样
        Car car = context.getBean("car", Car.class);
        car.forward("奥迪");
    }
}
```

![image-20210819144011637](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819144011637.png)

### 3.3.5 通知的执行顺序

``环绕通知 -> 前置通知 -> 目标方法 -> 后置通知 -> 最终通知``

抛出异常通知随时可能执行，根据异常触发决定

### 3.3.6 设置增强类的加载优先级

有多个增强类对同一个方法进行增强，可设置增强类的加载优先级。

**举例**：比如上面有一个CarAspect增强类对User类中的方法进行增强，现在有一个CarAspect2增强类也对User类中的方法进行增强，那么哪个肯定是哪个增强类先被加载，则先执行哪个增强类。所以我们可以通过在增强类上面添加注解 ``@Order(数字类型值)``进行设置类的加载优先级，数字类型值越小优先级越高

```java
@Order(1)//加载优先级
@Component // 通过IOC中的注解将该类实例化到Spring容器
@Aspect // 声明切面类，并为本类生成代理对象
public class CarAspect {
     .....省略.....   
}
```