# Spring  源码剖析

# 说在前面

- **本章相关代码及笔记地址**：飞机票🚀
- **🌍**Github：[🚀Java超神之路：【🍔Java全生态技术学习笔记，一起超神吧🍔】](https://github.com/EayonLee/JavaGod)
- **🪐**CSDN：[🚀Java超神之路：【🍔Java全生态技术学习笔记，一起超神吧🍔】](https://blog.csdn.net/qq_20492277/article/details/114269863)

# 目录
- [Spring  源码剖析](#spring--源码剖析)
- [说在前面](#说在前面)
- [目录](#目录)
- [一. Bean完整的生命周期](#一-bean完整的生命周期)
  - [1.1 BeanDefinition（Bean定义信息）](#11-beandefinitionbean定义信息)
  - [1.2 BeanFactoryPostProcessor（BeanFactory的后置处理器）](#12-beanfactorypostprocessorbeanfactory的后置处理器)
  - [1.3 Bean实例化、填充属性、Aware接口](#13-bean实例化填充属性aware接口)
  - [1.4 BeanPostProcessor接口 / init-method](#14-beanpostprocessor接口--init-method)
  - [1.5 destory-method销毁方法-完整的Bean声明周期结束](#15-destory-method销毁方法-完整的bean声明周期结束)
- [二. Bean的生命周期 源码解析](#二-bean的生命周期-源码解析)
  - [2.1 准备工作](#21-准备工作)
  - [2.2 ClassPathXmlApplicationContext方法](#22-classpathxmlapplicationcontext方法)
  - [2.3 验证](#23-验证)
    - [2.3.1 refresh()：方法](#231-refresh方法)
    - [2.3.2 freshBeanFactory()：创建bean工厂并加载xml文件到BeanDefinition](#232-freshbeanfactory创建bean工厂并加载xml文件到beandefinition)
    - [2.3.3 finishBeanFactoryInitialization()：完成上下文bean工厂的初始化](#233-finishbeanfactoryinitialization完成上下文bean工厂的初始化)
    - [2.3.4 getBean()：获取bean](#234-getbean获取bean)
    - [2.3.5 createBean()：创建Bean](#235-createbean创建bean)
    - [2.3.6 popelateBean()：Bean的属性赋值](#236-popelatebeanbean的属性赋值)
    - [2.3.7 invokeAwareMethods()：执行bean所实现Aware接口方法](#237-invokeawaremethods执行bean所实现aware接口方法)
    - [2.3.8 beanPostProcessors的before方法](#238-beanpostprocessors的before方法)
    - [2.3.9 init-method：执行初始化方法](#239-init-method执行初始化方法)
    - [2.3.10 beanPostProcessors的after方法](#2310-beanpostprocessors的after方法)
  - [2.4 总结](#24-总结)
- [三. 循环依赖](#三-循环依赖)
  - [3.1 什么是循环依赖？](#31-什么是循环依赖)
  - [3.2 测试循环依赖](#32-测试循环依赖)
    - [3.2.1 准备实体](#321-准备实体)
    - [3.2.2 XML文件创建Bean并属性注入](#322-xml文件创建bean并属性注入)
    - [3.2.3 测试及结果分析](#323-测试及结果分析)
  - [3.2 循环依赖的几种场景](#32-循环依赖的几种场景)
  - [3.3 单例的setter注入如何解决循环依赖](#33-单例的setter注入如何解决循环依赖)
    - [3.3.1 为什么正常情况下Spring不会出现循环依赖问题](#331-为什么正常情况下spring不会出现循环依赖问题)
    - [3.3.2 Spring默认针对单例setter注入是如何解决循环依赖的](#332-spring默认针对单例setter注入是如何解决循环依赖的)
    - [3.3.3 什么是三级缓存](#333-什么是三级缓存)
    - [3.3.4 源码验证](#334-源码验证)
  - [3.4 为什么需要三级缓存，二级行不行?](#34-为什么需要三级缓存二级行不行)
- [四. JDK动态代理源码分析](#四-jdk动态代理源码分析)
  - [4.1 JDK动态代理的实现](#41-jdk动态代理的实现)
  - [4.2 源码分析](#42-源码分析)
  - [4.3 总结](#43-总结)

# 一. Bean完整的生命周期

## 1.1 BeanDefinition（Bean定义信息）

![image-20210819164327964](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819164327964.png)

一般我们会通过XML配置文件、注解、配置类去定义bean。当我们去加载配置文件或者配置类的时候首先会去通过BeanDefinitionReader接口的实现类去读取bean的一些定义。比如我们通过XML配置文件去进行bean定义，则会通过XMLBeanDefinitionReader进行解析并将bean的定义信息、依赖关系等保存至BeanDefinition中。

## 1.2 BeanFactoryPostProcessor（BeanFactory的后置处理器）

![image-20210819164409397](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819164409397.png)

**为什么要有BeanFactoryPostProcessor？它的作用是什么？**

一般情况我们将bean的一些定义信息保存至BeanDefinition中之后，就可以通过BeanFactory中的反射将bean进行实例化了。

但是根据我们开发经验，我们有时候会将数据源的一些配置信息（DataSource）通过XML或者配置类进行定义。并且会将属性值通过 ${} 占位符的方式去引入外部文件赋值，如下图：

![image-20210819164438580](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819164438580.png)

但是我们通过BeanDefinitionReader解析配置文件保存至BeanDefinition这个过程中并没有去解析`` ${}`` 占位符中的值，BeanDefinition中保存的还是``${jdbc.url}``这样的属性值。那么这个时候我们肯定是不能直接通过BeanFactory去实例化的，而是通过BeanFactoryPostProcessor去进行后置处理，解析占位符，再通过BeanFactory去实例化。

## 1.3 Bean实例化、填充属性、Aware接口

通过如上步骤我们将bean的定义信息全部解析完毕保存至BeanDefinition中并通过BeanFactoryPostProcessor进行增强后，就可以进行实例化和初始化。

实例化没啥好说就是通过反射去new对象，但是此时的bean中的属性值都是空的，赋值操作是由初始化来完成的。

**初始化又分为两步**：

- 属性赋值
- 执行bean所实现Aware接口的方法。

![image-20210819164521220](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819164521220.png)

属性赋值很简单我们就不说了，主要看看什么叫执行bean所实现的Aware接口方法。

**执行bean所实现的Aware接口方法**：

Aware接口的意义在于方便通过Spring中的Bean对象来获取其在容器中对应的相关属性值

**定义一个对象并实现Aware接口**

```java
public class Teacher implements EnvironmentAware , BeanNameAware {
    /**
     * bean的一些基础属性（业务层面的属性）
     */
    private String teacherName;
    private Integer age;

    /**
     * bean在容器中的属性
     */
    private Environment environment;
    private String beanName;

    /**
     * 通过EnvironmentAware设置该Bean在容器中的环境
     * @param environment
     */
    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    public Environment getEnvironment() {
        return environment;
    }

    /**
     * 设置该Bean再容器中的名称
     * @param s
     */
    @Override
    public void setBeanName(String s) {
        this.beanName = s;
    }

    public String getBeanName() {
        return beanName;
    }
}
```

**获取bean在容器中的属性值**

```java
@Test
public void test_aware(){
    ApplicationContext ac = new AnnotationConfigApplicationContext(JdbcConfiguration.class);//加载配置类
    Teacher bean = ac.getBean("teacher", Teacher.class);
    //Aware接口的意义在于方便通过Spring中的Bean对象来获取其在容器中对应的相关属性值
    String beanName = bean.getBeanName();//bean的名称
    Environment environment = bean.getEnvironment();//环境
    System.out.println(beanName);
    System.out.println(environment);
}
```

![image-20210819164555973](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819164555973.png)

## 1.4 BeanPostProcessor接口 / init-method

![image-20210819164617452](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819164617452.png)

- 如果Bean实现了**BeanPostProcess**接口，Spring将调用它们的**postProcessBeforeInitialization**方法

- - （作用是在Bean实例创建成功后对进行增强处理，如对Bean进行修改，增加某个功能）

- 如果Bean声明了**init-method方法**则会进行调用，完成初始化

- 如果Bean实现了**BeanPostProcess**接口，Spring将调用它们的**postProcessAfterInitialization**方法

- - （和执行before方法一样都是进行增强 ，只不过一个在初始化之前，一个在初始化方法之后)

## 1.5 destory-method销毁方法-完整的Bean声明周期结束

![image-20210819164643740](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819164643740.png)

如果该Bean声明了**destory-method**方法则会进行调用销毁。但是对于我们高级语言java来说，我们日常很少会声明销毁，都是通过垃圾回收进行解决。



# 二. Bean的生命周期 源码解析

## 2.1 准备工作

**定义一个对象**

```java
public class Teacher{

    private String teacherName;
    private Integer age;

    ...省略get/set方法.....
}
```

**通过XML文件创建Bean及注入属性**

```java
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

    <bean id="teacher" class="com.eayon.entity.Teacher">
        <property name="teacherName" value="张三"></property>
        <property name="age" value="24"></property>
    </bean>
    
</beans>
```

**测试加载Bean**

```java
@Test
public void test_load_bean(){
    ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");//加载XML
    Teacher teacher = ac.getBean("teacher", Teacher.class);
    System.out.println(teacher.getTeacherName());
    System.out.println(teacher.getAge());
}
```

## 2.2 ClassPathXmlApplicationContext方法

```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
    super(parent);
    //configLocations参数也就是1.6.1准备工作中的创建Bean的XML文件
    //将XML配置文件设置到类的成员属性中方便后面的读取和加载
    this.setConfigLocations(configLocations);
    if (refresh) {
        //refresh方法刷新Spring应用的上下文，这个方法里面包含了所有Bean生命周期的方法
        this.refresh();
    }
}
```

## 2.3 验证

本节``refresh``方法我们只去找创建bean的方法（createBean），其他的暂时我们不去研究

### 2.3.1 refresh()：方法

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized(this.startupShutdownMonitor) {
        /**
        * （不重要 知道是准备工作即可）
        * prepareRefresh()方法 做容器刷新前的准备工作：
        * 1、设置容器的启动时间
        * 2、设置活跃状态为true
        * 3、设置关闭状态为false
        * 4、获取Environment对象，并加载当前系统的属性值到Environment对象中
        * 5、准备监听器和事件的集合对象，默认为空的集合
       */    
        this.prepareRefresh();
        
        //（重要）创建BeanFactory工厂，并加载XML配置文件中的属性到当前工厂中的BeanDefinition
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        
        //（不重要）对上一步创建的beanFactory做一些初始化属性填充的准备工作
        this.prepareBeanFactory(beanFactory);

        try {
            //（不重要）源码中该方法内部为空，是留给它子类做一些扩展用的。
            this.postProcessBeanFactory(beanFactory);
            
            //（重要）invoke是执行的意思 那该方法也就是执行beanFactoryPostProcessors（beanFactory的后置处理器）
            //我们之前所说的将xml文件的属性值的占位符解析成实际的值等扩展操作都是在这里执行。
            this.invokeBeanFactoryPostProcessors(beanFactory);
            
            //（有点重要）注册beanPostProcessors，这里只是注册（准备出来）
            //按之前的流程图来说本来应该实例化对象了，但是实例化之后的初始化操作中需要beanPostProcessors，所以我们在实例化之前先将他准备出来
            //注意：这里是beanPostProcessors和上面的beanFactoryPostProcessors不一样，所以这里只是注册，真正的调用是在getBean()的时候调用
            this.registerBeanPostProcessors(beanFactory);
            
            //（不重要）为上下文初始化message源，也就是不同语言的消息体，国际化处理
            this.initMessageSource();
            
            //(有点重要)初始化事件监听多路广播器
            //如果我们现在bean生命周期不同的阶段做不同的事情我们需要通过观察者模式：监听器、监听事件、多播器来实现，所以这里先进性初始化
            this.initApplicationEventMulticaster();
            
            //（不重要）一个空方法，留给子类来初始化其他bean的
            this.onRefresh();
            
            //（有点重要）注册监听器，也属于实例化bean的一些前期准备工作
            this.registerListeners();
            
            //（重要）实例化所有的单例bean（非懒加载的bean）
            this.finishBeanFactoryInitialization(beanFactory);
            
            //最后一步：发布相应的事件。
            this.finishRefresh();
        } catch (BeansException var9) {
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
            }

            this.destroyBeans();
            this.cancelRefresh(var9);
            throw var9;
        } finally {
            this.resetCommonCaches();
        }

    }
}
```

### 2.3.2 freshBeanFactory()：创建bean工厂并加载xml文件到BeanDefinition

``ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();``

创建BeanFactory工厂，并加载XML配置文件中的属性到当前工厂中的BeanDefinition

![image-20210819165049320](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819165049320.png)

**prepareBeanFactory()**

进入到该prepareBeanFactory方法中可以找到下图的``refreshBeanFactory``方法，该方法中就是通过``loadBeanDefinitions(beanFactory)``方法去加载BeanDefinition。

![image-20210819165107806](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819165107806.png)

**loadBeanDefinitions()**

那么就是通过``XmlBeanDefinitionReader()``方法解析xml文件中的定义并保存到BeanDefinition中

![image-20210819170633093](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819170633093.png)

### 2.3.3 finishBeanFactoryInitialization()：完成上下文bean工厂的初始化

实例化所有**非懒加载的单例bean**

```java
/**
* 完成此上下文的bean工厂的初始化，
* 初始化所有剩余的单例bean。
*/
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
   // （不用看）为此上下文初始化转换服务。
   if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
         beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
      beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
   }

   //  （不用看）如果beanFactory之前没有注册嵌入值解析器，则注册默认的嵌入值解析器，主要用于注解属性值的解析
   if (!beanFactory.hasEmbeddedValueResolver()) {
      beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
   }

   // （不用看）尽早初始化loadTimeAware bean，以便尽早注册它们的转换器
   String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
   for (String weaverAwareName : weaverAwareNames) {
      getBean(weaverAwareName);
   }

   //（不用看）禁止使用临时类加载器进行类型匹配
   beanFactory.setTempClassLoader(null);

   //（不用看）冻结所有bean的定义，说明注册的bean定义将不被修改或任何进一步的处理。
   beanFactory.freezeConfiguration();

   //（重要）这一步开始去真正的去实例化剩下的所有非懒加载的单例对象
   beanFactory.preInstantiateSingletons();
}
```

**preInstantiateSingletons()**

实例化之前再做一些判断和前期准备工作

```java
public void preInstantiateSingletons() throws BeansException {
   if (logger.isTraceEnabled()) {
      logger.trace("Pre-instantiating singletons in " + this);
   }

   //将所有beanDefinition的名字创建成一个集合，其实也就是bean的名称
   List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

   //遍历该集合。初始化所有非懒加载的单例bean
   for (String beanName : beanNames) {
       //合并父类beanDefinition
      RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
      //条件判断：是否是抽象的、是否是单例、是否是懒加载
      if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
          //判断该bean是否实现了FactoryBean接口
         if (isFactoryBean(beanName)) {
             //FACTORY_BEAN_PREFIX 是定义的常量 & 
             //也就是根据 & + beanName来获取具体的bean
            Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
            // 判断bean是否等于 FactoryBean
            if (bean instanceof FactoryBean) {
               final FactoryBean<?> factory = (FactoryBean<?>) bean;
               //判断这个beanFactory是否急切的需要实例化的标识
               boolean isEagerInit;
               if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                  isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                              ((SmartFactoryBean<?>) factory)::isEagerInit,
                        getAccessControlContext());
               }
               else {
                  isEagerInit = (factory instanceof SmartFactoryBean &&
                        ((SmartFactoryBean<?>) factory).isEagerInit());
               }
               //如果急切实例化则通过beanName获取实例化bean
               if (isEagerInit) {
                  getBean(beanName);
               }
            }
         }
         else {
             //如果beanName对应的bean不是FactoryBean类型，或者说没有实现FactoryBean接口，只是一个普通bean，则执行getBean()
             //getBean()是去看容器中存不存在该beanName的bean实例，有直接获取，没有则创建
            getBean(beanName);
         }
      }
   }

   //遍历beanNames，触发所有SmartInitializingSingleton的后初始化回调
   for (String beanName : beanNames) {
       //获取beanName对应的bean实例
      Object singletonInstance = getSingleton(beanName);
      if (singletonInstance instanceof SmartInitializingSingleton) {
         final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
         if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
               smartSingleton.afterSingletonsInstantiated();
               return null;
            }, getAccessControlContext());
         }
         else {
            smartSingleton.afterSingletonsInstantiated();
         }
      }
   }
}
```

### 2.3.4 getBean()：获取bean

**getBean()**

```java
public Object getBean(String name) throws BeansException {
    //此方法才是真正实例化bean的方法，前面都是前期准备工作，也是依赖注入的主要方法
    //Spring源码中，do开头的方法才是真正干活的方法
   return doGetBean(name, null, null, false);
}
```

**doGetBean()**

此方法才是真正去调用（仅仅是调用）实例化bean的方法，前面都是前期准备工作，同时该方法也是依赖注入的主要方法

ps：Spring源码中，do开头的方法才是真正干活的方法

此方法较长，包含需多循环依赖的东西，可以暂时不看，后面会单独将，只需要找到一个**createBean()**方法即可

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    //beanName转换
   final String beanName = transformedBeanName(name);
   Object bean;

   // 提前检查单例缓存中是否有手动注册的单例对象，此处跟循环依赖有关
   Object sharedInstance = getSingleton(beanName);
   if (sharedInstance != null && args == null) {
      if (logger.isTraceEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' that is not fully initialized yet - a consequence of a circular reference");
         }
         else {
            logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
         }
      }
      //返回对象的实例，当我们的bean实现了FactoryBean接口，需要获取具体的对象的时候就需要此方法来进行获取
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // 当对象都是单例的时候会尝试解决循环依赖的问题，但是原型模式下如果存在循环依赖的情况，那么直接抛出异常
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // 获取父容器
      BeanFactory parentBeanFactory = getParentBeanFactory();
      //如果beanDefinitionMap中也就是在所有已经加载的类中不包含beanName，那么就尝试从父容器中获取
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         String nameToLookup = originalBeanName(name);
         if (parentBeanFactory instanceof AbstractBeanFactory) {
            return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                  nameToLookup, requiredType, args, typeCheckOnly);
         }
         else if (args != null) {
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else if (requiredType != null) {
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
         else {
            return (T) parentBeanFactory.getBean(nameToLookup);
         }
      }
        //如果不是做类型检查，那么表示要创建bean，此处在集合中做一个记录
      if (!typeCheckOnly) {
         markBeanAsCreated(beanName);
      }

      try {
          //此处做了BeanDefinition对象的转换，当我们从xml文件中加载beanDefinition对象的时候，封装的对象是GenericBeanDefinition
          //此处要做类型转换，如果是子类bean的话，会合并父类的相关属性
         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         //如果存在依赖的bean的话，那么则优先实例化依赖的bean
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
             //如果存在依赖，则需要递归实例化依赖的bean
            for (String dep : dependsOn) {
               if (isDependent(beanName, dep)) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               //注册各个bean的依赖关系，以便进行销毁
               registerDependentBean(dep, beanName);
               try {
                  getBean(dep);
               }
               catch (NoSuchBeanDefinitionException ex) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
               }
            }
         }

         //（本小节重点，前面可以不看）创建单例模式bean的实例对象
         if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
               try {
                   //这里去创建bean
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }
        //创建原型模式的bean实例对象
         else if (mbd.isPrototype()) {
            Object prototypeInstance = null;
            try {
               beforePrototypeCreation(beanName);
               prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
               afterPrototypeCreation(beanName);
            }
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

//===============================省略一堆无用代码====================================

   return (T) bean;
}
```

### 2.3.5 createBean()：创建Bean

**createBean()**

之前我们说过，spring源码中真正干活的都是带do的方法，那么我们只需要在当前方法中找到``doCreateBean``即可。

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   //锁定class，根据设置的class属性或者根据className来解析class
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   //验证及准备覆盖的方法
   try {
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // 给BeanPostProcessors一个机会来返回代理来替代真正的实例
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
       //（此小节重点方法）真正实际创建bean的调用
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
         logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

**doCreateBean()**

ok！见到亲人了，终于找到真正干活的人了

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // 这个beanWrapper是用来持有创建出来的bean对象的
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
       //如果是单例对象，从factoryBean实例缓存中移除当前bean定义信息
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
       //（重要，通过反射创建实例）根据执行bean使用对应的策略创建新的实例，如：工厂方法、构造函数主动注入、简单初始化
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   //创建完成后的实例
   final Object bean = instanceWrapper.getWrappedInstance();
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // 允许后置处理器postPorcessor修改合并的bean定义
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
             //应用MergedBeanDefinitionPostProcessor
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

  //==========省略一堆无用代码===========================

   return exposedObject;
}
```

**createBeanInstance(）**

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
   //确认需要创建的bean实例的类可以实例化
   Class<?> beanClass = resolveBeanClass(mbd, beanName);

   if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
   }

   Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
   if (instanceSupplier != null) {
      return obtainFromSupplier(instanceSupplier, beanName);
   }

    //如果工厂方法不为空则使用工厂方法初始化策略
   if (mbd.getFactoryMethodName() != null) {
      return instantiateUsingFactoryMethod(beanName, mbd, args);
   }

   // Shortcut when re-creating the same bean...
   boolean resolved = false;
   boolean autowireNecessary = false;
   if (args == null) {
      synchronized (mbd.constructorArgumentLock) {
          //一个类中有多个构造函数，每个构造函数都有不同的参数，所以调用前需要现根据参数锁定构造函数或对应的工厂方法
         if (mbd.resolvedConstructorOrFactoryMethod != null) {
            resolved = true;
            autowireNecessary = mbd.constructorArgumentsResolved;
         }
      }
   }
   //如果已经解析过则使用解析好的构造函数方法，不需要再次锁定
   if (resolved) {
      if (autowireNecessary) {
          //（重要）构造函数自动注入
         return autowireConstructor(beanName, mbd, null, null);
      }
      else {
          //（重要）使用默认构造函数构造
         return instantiateBean(beanName, mbd);
      }
   }

   // 需要根据参数解析构造函数
   Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
         mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
      return autowireConstructor(beanName, mbd, ctors, args);
   }

   // Preferred constructors for default construction?
   ctors = mbd.getPreferredConstructors();
   if (ctors != null) {
       //（重要）构造函数自动注入
      return autowireConstructor(beanName, mbd, ctors, null);
   }

   //（重要）使用默认构造函数构造
   return instantiateBean(beanName, mbd);
}
```

**instantiateBean()**

那我们主要来看这个使用默认构造函数构造的方法即可

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
   try {
      Object beanInstance;
      final BeanFactory parent = this;
      if (System.getSecurityManager() != null) {
         beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
               getInstantiationStrategy().instantiate(mbd, beanName, parent),
               getAccessControlContext());
      }
      else {
          //getInstantiationStrategy()：获取实例化的策略
          //instantiate()：进行实例化
          //得到实例化后的bean，所以我们再来看看这个instantiate()方法是如何进行实例化
         beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
      }
      BeanWrapper bw = new BeanWrapperImpl(beanInstance);
      initBeanWrapper(bw);
      return bw;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
   }
}
```

**instantiate()**

```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
   //如果需要覆盖或者动态替换的方法则使用cglig进行动态代理，因为可以在创建代理的同时将动态方法织入类中，但是如果没有需要动态改变的方法，为了方便直接反射就可以了
   if (!bd.hasMethodOverrides()) {
       //此处获取到指定的构造器对bean进行实例化
      Constructor<?> constructorToUse;
      synchronized (bd.constructorArgumentLock) {
         constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
         if (constructorToUse == null) {
            final Class<?> clazz = bd.getBeanClass();
            if (clazz.isInterface()) {
               throw new BeanInstantiationException(clazz, "Specified class is an interface");
            }
            try {
               if (System.getSecurityManager() != null) {
                  constructorToUse = AccessController.doPrivileged(
                        (PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
               }
               else {
                   //根据beanClass获取到构造器
                  constructorToUse = clazz.getDeclaredConstructor();
               }
               bd.resolvedConstructorOrFactoryMethod = constructorToUse;
            }
            catch (Throwable ex) {
               throw new BeanInstantiationException(clazz, "No default constructor found", ex);
            }
         }
      }
      //（重要）使用BeanUtils工具类根据bean的构造器进行反射实例化并返回
      return BeanUtils.instantiateClass(constructorToUse);
   }
   else {
      // Must generate CGLIB subclass.
      return instantiateWithMethodInjection(bd, beanName, owner);
   }
}
```

**BeanUtils.instantiateClass(constructorToUse)**

```java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
   Assert.notNull(ctor, "Constructor must not be null");
   try {
      ReflectionUtils.makeAccessible(ctor);
      if (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass())) {
         return KotlinDelegate.instantiateClass(ctor, args);
      }
      else {
         Class<?>[] parameterTypes = ctor.getParameterTypes();
         Assert.isTrue(args.length <= parameterTypes.length, "Can't specify more arguments than constructor parameters");
         Object[] argsWithDefaultValues = new Object[args.length];
         for (int i = 0 ; i < args.length; i++) {
            if (args[i] == null) {
               Class<?> parameterType = parameterTypes[i];
               argsWithDefaultValues[i] = (parameterType.isPrimitive() ? DEFAULT_TYPE_VALUES.get(parameterType) : null);
            }
            else {
               argsWithDefaultValues[i] = args[i];
            }
         }
         //（重要）通过bean的构造器进行实例化，到这里实例化bean的操作就结束了
         return ctor.newInstance(argsWithDefaultValues);
      }
   }
   catch (InstantiationException ex) {
      throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
   }
   catch (IllegalAccessException ex) {
      throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
   }
   catch (IllegalArgumentException ex) {
      throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
   }
   catch (InvocationTargetException ex) {
      throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
   }
}
```

这个时候我们创建完bean回到``doCreateBean()``方法查看创建的bean可以发现，仅仅是进行了实例化，并没有进行属性赋值

![image-20210819171239341](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819171239341.png)

### 2.3.6 popelateBean()：Bean的属性赋值

在刚刚的``doCreateBean()``方法中我们已经获取到了实例化的bean，并且发现是没有进行属性赋值的，所以我们还在当前``doCreateBean()``方法中找到``popelateBean()``属性赋值方法。

**再来看看doCreateBean()方法**

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // 这个beanWrapper是用来持有创建出来的bean对象的
   BeanWrapper instanceWrapper = null;
   if (mbd.isSingleton()) {
       //如果是单例对象，从factoryBean实例缓存中移除当前bean定义信息
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   if (instanceWrapper == null) {
       //（重要，通过反射创建实例）根据执行bean使用对应的策略创建新的实例，如：工厂方法、构造函数主动注入、简单初始化
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }
   //创建完成后的实例bean
   final Object bean = instanceWrapper.getWrappedInstance();
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // 允许后置处理器postPorcessor修改合并的bean定义
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
             //应用MergedBeanDefinitionPostProcessor
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

    //判断当前bean是否需要提前曝光：单例&允许循环依赖&当前bean正在创建中，检测循环依赖
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
          isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
       if (logger.isTraceEnabled()) {
          logger.trace("Eagerly caching bean '" + beanName +
                "' to allow for resolving potential circular references");
       }
       //为避免后期循环依赖，可以在bean初始化完成前将创建实例的ObjectFactory加入工厂
       addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
    
    // 初始化bean
    Object exposedObject = bean;
    try {
        //（重要）对bean的属性进行填充，将各个属性值注入，其中。可能存在依赖于其他bean的属性，则会递归初始化依赖的bean
        //只需要知道这里进行bean的一些普通属性填充即可
       populateBean(beanName, mbd, instanceWrapper);
       //（重要）执行初始化逻辑
       exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    
    //=====================省略一些无用代码=========================

   return exposedObject;
}
```

![image-20210819171325907](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819171325907.png)

该bean只去填充了一些普通属性，对于实现的Aware接口的方法不在此处进行调用。

### 2.3.7 invokeAwareMethods()：执行bean所实现Aware接口方法

那么对于bean实现的Aware接口的方法在何处进行调用？其实就在``populateBean()``方法的下面，``initializeBean()``当中，我们可以来看一下

**initializeBean()**

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
       //（重要）执行bean所实现Aware接口的方法
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
       //如果bean实现了BeanPostProcessors接口则调用该before方法进行一些额外的扩展增强操作
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
       //（重要）执行初始化方法
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
       //（重要）如果bean实现了BeanPostProcessors接口则调用该after方法进行一些额外的扩展增强操作
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

![image-20210819171416527](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819171416527.png)

执行完``invokeAwareMethods()``方法后我们发现，已经执行了bean所实现的Aware接口中的方法，并完成属性的赋值。

所以也证实了bean所实现的Aware接口中的方法的调用是在bean属性填充操作后去执行的。

**PS**：environmen为什么没有调用赋值？是因为spring源码在前期的准备工作者屏蔽了environment，他应该在``beanPostProcessors``的``before``方法中去调用。

### 2.3.8 beanPostProcessors的before方法

before方法的全称其实叫：**applyBeanPostProcessorsBeforeInitialization（）**

before方法也在``initializeBean()``方法中，就在``invokeAwareMethods方法``的下面

如果bean实现了beanPostProcessors接口则会执行``before方法``做一些扩展增强操作，另外environmentAware也会在此处进行调用赋值

**initializeBean()**

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
       //（重要）执行bean所实现Aware接口的方法
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
       //（重要）如果bean实现了BeanPostProcessors接口则调用该before方法进行一些额外的扩展增强操作
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
       //（重要）执行初始化方法
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
       //（重要）如果bean实现了BeanPostProcessors接口则调用该after方法进行一些额外的扩展增强操作
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

我们就不进before方法当中去看了，里面无非就是获取bean所实现的BeanPostProcessors接口方法然后去执行。

### 2.3.9 init-method：执行初始化方法

初始化方法的全称其实叫：**invokeInitMethods()**

``init-method方法``也在``initializeBean()``方法中，就在``before方法``的下面

**initializeBean()**

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
       //（重要）执行bean所实现Aware接口的方法
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
       //（重要）如果bean实现了BeanPostProcessors接口则调用该before方法进行一些额外的扩展增强操作
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
       //（重要）执行初始化方法
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
       //（重要）如果bean实现了BeanPostProcessors接口则调用该after方法进行一些额外的扩展增强操作
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

我们就不进``init-method方法``当中去看了，里面无非就是执行用户自定义的``init-method方法``并且执行

### 2.3.10 beanPostProcessors的after方法

after方法的全称其实叫：**applyBeanPostProcessorsAfterInitialization（）**

after方法也在``initializeBean()``方法中，就在``invokeInitMethods方法``的下面

如果bean实现了beanPostProcessors接口则会执行``after方法``做一些扩展增强操作

**initializeBean()**

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
       //（重要）执行bean所实现Aware接口的方法
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
       //（重要）如果bean实现了BeanPostProcessors接口则调用该before方法进行一些额外的扩展增强操作
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
       //（重要）执行初始化方法
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
       //（重要）如果bean实现了BeanPostProcessors接口则调用该after方法进行一些额外的扩展增强操作
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

我们就不进after方法当中去看了，里面无非就是获取bean所实现的BeanPostProcessors接口方法然后去执行。

**至此我们获得一个完整的bean对象**

## 2.4 总结

总结我们还是按照之前画的流程图来总结吧

![image-20210819171723890](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819171723890.png)

1. 我们首先都会在xml中定义bean，或者直接通过注解，这里我们之说xml的情况。

2. 首先会通过**BeanDefinitionReader**（根据不同的bean定义方式来选择响应的实现类）来解析xml文件并保存至**BeanDefinition**中。

3. 然后由于实际开发中我们经常会在xml配置文件定义bean时使用占位符的方式引入外部属性值，所以此时保存到BeanDefinition的属性值会存在为解析的占位符的情况。这种情况我们肯定不能将他直接通过BeanFactory去反射实例化。所以我们会通过BeanFactoryPostProcessor来对bean进行实例化前的扩展增强，这其中就包含了解析占位符的操作。

4. 得到完整的一个BeanDefinition后我们就可以通过**反射的方式实例化bean**，注意：实例化的都是懒加载的bean。

5. 接着进入初始化步骤：

6. 1. bean的**属性填充**
   2. 如果bean实现了**Aware接口**中的方法则在属性填充后进行方法调用并赋值
   3. 如果bean实现了**BeanPostProcessor接口的before方法**则进行调用增强
   4. 如果bean定义了**init-method方法**则进行调用此初始化方法
   5. 如果bean实现了**BeanPostProcessor接口的after方法**则进行调用增强

7. 获取到**完整的bean实例**

8. 如果bean定义了**destory-method销毁方法**则还会进行销毁，但一般我们都通过java的垃圾回收解决此问题



# 三. 循环依赖

## 3.1 什么是循环依赖？

**循环依赖分为三种**：自身依赖于自身、互相循环依赖、多组循环依赖。

![image-20210819171809288](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819171809288.png)

但无论循环依赖的数量有多少，循环依赖的本质是一样的。就是你的完整创建依赖于我，而我的完整创建也依赖于你，但我们互相没法解耦，最终导致依赖创建失败。

所以 Spring 提供了除了构造函数注入和原型注入外的，setter循环依赖注入解决方案。那么我们也可以先来尝试下这样的依赖，如果是我们自己处理的话该怎么解决。

## 3.2 测试循环依赖

### 3.2.1 准备实体

准备两个实体A和B，A中注入B实体属性，B中注入A实体属性

```java
//实体A
public class A {
    private B b;
    
    public void setB(B b) {
        this.b = b;
    }
}

//实体B
public class B {
    private A a;
    
    public void setA(A a) {
        this.a = a;
    }
}
```

### 3.2.2 XML文件创建Bean并属性注入

```java
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

    <bean id="a" class="com.eayon.entity.A">
        <property name="b" ref="b"></property>
    </bean>

    <bean id="b" class="com.eayon.entity.B">
        <property name="a" ref="a"></property>
    </bean>
</beans>
```

### 3.2.3 测试及结果分析

按刚刚``3.1章节``我们分析什么是循坏依赖时的想法，这时候我们加载XML文件他会去实例化A和B，然后抛出循环依赖的异常，那我们来测试一下

![image-20210819171937666](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819171937666.png)

我们发现，它最后正常的实例化出了A和B实例，并没有发生循环依赖。这是为什么呢？我们可以看下一节 ``3.3单例的setter注入如何解决循环依赖``

## 3.2 循环依赖的几种场景

![image-20210819172017667](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172017667.png)

## 3.3 单例的setter注入如何解决循环依赖

### 3.3.1 为什么正常情况下Spring不会出现循环依赖问题

还记得我们在最开始跟踪Spring源码分析bean的生命周期时有一个``preInstantiateSingletons()``方法，该方法再去调用``getBean()``方法之前有一个判断，判断当前bean是否是抽象的、单例的、懒加载的。所以我们实例化的bean都是单例非懒加载的。

![image-20210819172126743](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172126743.png)

**重要**：

那么再回头看一下3.2.2小节我们给bean属性注入的方式是set方法。

那么其实在Spring中对于单例set注入的bean是默认解决了循环依赖问题的。解决方法是 实例提前暴露+三层缓存

但是前提必须是单例的setter注入，如果是多例setter注入或者是单例/多例构造器注入都是无法解决的。

### 3.3.2 Spring默认针对单例setter注入是如何解决循环依赖的

针对于之前我们了解的bean生命周期得出。bean的实例化操作和属性赋值（初始化）操作是分开的。我们先实例化得到一个bean的半成品然后去属性赋值，或者说注入其他实体。

那么我们首先加载实体A，第一次肯定从缓存中获取不到，所以去创建实例，然后提前暴露添加到三级缓存，当我们去初始化A属性注入时，发现需要实例化B，然后我们再从缓存中获取不到B，则取创建B实例，然后将B暴露添加到三级缓存，当我们去初始化属性注入时发现是实体A，我们则从三级缓存中获取到了实体A，并添加到了二级缓存。至此实体B初始化完成，得到一个完整的bean，最后将B添加到一级缓存。紧接着回到实体A，成功完成实体A的属性注入，初始化完成得到一个完整的bean，并将A添加到一级缓存。至此bean加载结束。完美解决了循环依赖的问题。

![image-20210819172206521](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172206521.png)

### 3.3.3 什么是三级缓存

**三级缓存的定义**

![image-20210819172227057](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172227057.png)

**为什么三级缓存保存的是ObjectFactory对象？直接保存实例对象不行？**

ObjectFactory对象其实是getEarlyBeanReference函数表达式，执行该表达式可以获取到bean早期的bean引用，并判断该bean是否需要代理，简单地说就是，如果一个bean我们定义了他需要代理，那么通过三级缓存的函数表达式执行获取到的就是代理对象，如果不需要代理，获取到的就是普通对象。

**一级缓存**：存储的是完整实例

**二级缓存**：存储的是半成品实例，也就是还未属性赋值的实例

**三级缓存**：存储的是getEarlyBeanReference()函数表达式，该表达式可以获取到bean早期的bean引用，并判断该bean是否需要代理，最终返回代理对象或普通对象

**为什么要三级缓存，二级行不行？**

所以说我们只用二级缓存行不行其实取决你到底需不需要代理对象。

如果需要代理，我们则在创建过程中其实是创建了两个bean，一个是普通bean，一个是代理bean。**所以他就会在通过调用getEarlyBeanReference()函数表达式想要用你这个bean的时候，将代理bean覆盖掉普通bean，保证你这个bean的全局唯一**。如果说你不需要代理，那可以，你可以不需要三级缓存，二级缓存足以

### 3.3.4 源码验证

我们先把之前的流程图拿过来对照着看源码

![image-20210819172300357](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172300357.png)

**首先第一步从一级缓存中获取实例**

在``doGetBean()``方法中调用了``getSingleton()``方法从缓存中（这里单指一级缓存）获取bean。获取到bean就不去创建直接返回了。

![image-20210819172322779](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172322779.png)

**getSingleton()方法**

在方法中可看到他会根据beanName先从一级缓存中获取，如果一级缓存中不存在该bean并且该bean正在被创建的话才会去二级缓存中查找。

而此时我们的bean是首次创建，所以一级缓存里肯定没有，并且bean还没有进行创建，所以当从一级缓存中获取不到bean后直接返回一个空的object

![image-20210819172343807](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172343807.png)

**创建实例 A**

紧接着上面从一级缓存中未获取到bean返回空后，我们再来看看**doGetBean()** 方法的后半部分

经过一系列判断及准备操作之后通过mbd（beanDefinition）判断是否是单例的。

如果是单例的则调用**getSingleton()去创建bean**，其中该方法有两个参数，以格式beanName，另一个是函数表达式，我们就理解传递了**createBean()**方法过去。

该``getSingletion() ``方法主要逻辑就是，**再次从一级缓存查找，找不到则调用createBean()方法去创建bean**。

![image-20210819172359843](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172359843.png)

**getSingleton()**

方法的上半部分如下图

![image-20210819172426038](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172426038.png)

**doCeateBean()**

好！现在我们知道了他要去创建bean了，那我们就来看看``doCreateBean()``方法怎么执行的，

我们发现实例化A成功并返回了。但是注意现在还没有进行初始化，所以还没有B的依赖注入

![image-20210819172444018](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172444018.png)

我们再来看看**doCreateBean()**方法的下半部分

我们之前提到过，创建完bean后会将这个半成品添加到三级缓存，那么就是通过**addSingletonFactory()**实现的

![image-20210819172508987](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172508987.png)

**addSingletonFactory()**

那么该方法其实就是去将创建出来的半成品bean添加到三级缓存

![image-20210819172522491](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172522491.png)

**注意**：存入三级缓存中的value就是singletonFactory这个函数表达式。也就是上一步doCreateBean()方法中的 getEarlyBeanReference()方法。

getEarlyBeanReference方法用于获取代理对象，其实底层是通过AbstractAutoProxyCreator类的getEarlyBeanReference生成代理对象。

**为什么三级缓存保存的是ObjectFactory对象？直接保存实例对象不行？**

首先通过上面我们知道了``getEarlyBeanReference函数表达式``就是传递到``addSingletonFactory()``中singletonFactory参数。该表达式作用是用于获取代理对象，那其实三级缓存存的是``getEarlyBeanReference函数表达式`` 也可以理解为bean的代理对象。那为什么不能直接保存实例对象呢？因为假如你想对添加到三级缓存中的实例对象进行增强，直接用实例对象是行不通的。所以我们需要保存代理对象。

**populateBean() 属性赋值-依赖注入**

上面我们将半成品bean添加到三级缓存后，我们返回``doCreateBean方法``。将bean添加到三级缓存后，这时需要给bean进行初始化，也就是属性赋值依赖注入了。

关键的方法就在``populateBean()``。

![image-20210819172559868](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172559868.png)

那我们进入``populateBean()``方法看一下，由于该方法前面都是一些判断和预备操作，我们直接找到该方法最后面调用``applyPropertyValues()``方法查看如何进行属性注入

![image-20210819172626511](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172626511.png)

**applyPropertyValues()**

那么这时候我们其实是给A对象去进行属性赋值，那么需要注入的属性就是B，需要注入的属性值也就是B对象再运行时bean的引用

那么再次调用``resolveValueIfNecessary()``方法进行属性值处理

![image-20210819172650664](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172650664.png)

**resolveValueIfNecessary()**

![image-20210819172704304](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172704304.png)

**resolveReference()**

很明显，这个时候我们需要给A实体的属性b赋值，所以他这里调用了``getBean()``方法去容器中找B实例，我们知道``getBean()``方法会调用``doGetBean()``方法，``doGetBean()``方法中会去一级缓存中查找bean，找不到则去创建。那我们继续往下看看**doGetBean()**

**doGetBean()**

![image-20210819172740493](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172740493.png)

既然缓存没有那就实例化b呗，继续看该方法的后半段，他就去调用了``getSingleton()``方法 再次从一级缓存中获取b实例，获取不到就去调用``createBean()``方法创建

**getSingleton()**

![image-20210819172805889](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172805889.png)

**doCreateBean()**

我们不看``createBean()``了 直接看``doCreateBean()``

![image-20210819172822455](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172822455.png)

**addSingletonFactory()**

![image-20210819172836587](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172836587.png)

**populateBean()**

创建完b实例的半成品并添加到三级缓存之后，我们返回``doCreateBean()``方法找到给实例b进行属性赋值操作。

![image-20210819172851492](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172851492.png)

那进入方法看一下，找到``applyPropertyValues()``方法

![image-20210819172910520](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172910520.png)

**applyPropertyValues()**

![image-20210819172925167](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172925167.png)

**resolveValueIfNecessary()**

![image-20210819172939493](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172939493.png)

**resolveReference()**

![image-20210819172953142](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819172953142.png)

**doGetBean()**

![image-20210819173005437](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173005437.png)

**getSingleton()**

![image-20210819173017370](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173017370.png)

该函数表达式其实就是``getEarlyBeanReference()``方法，然后获取到A实例，注意这个A实例此时是个半成品 并没有给b属性赋值

**doGetBean()**

这时我们回到给B实例属性赋值的方法

![image-20210819173037168](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173037168.png)

**getSingleton()**

现在我们已经将B实例创建好也初始化好了，现在我们返回到了``getSingleton()``方法，现在需要将B这个完成的对象添加到一级缓存

![image-20210819173051610](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173051610.png)

添加到一级缓存

![image-20210819173110987](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173110987.png)

**doCreateBean()**

这时我们返回到创建A实例的``doCreateBean()``方法，此时实例A的属性赋值也全部完成

![image-20210819173132696](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173132696.png)

**getSingleton()**

现在我们已经将A实例创建好也初始化好了，现在我们返回到了getSingleton()方法，现在需要将A这个完成的对象添加到一级缓存

![image-20210819173146212](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173146212.png)

![image-20210819173155325](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173155325.png)

由于后面太乱了 截图太多不方便，所以就不继续走源码了，但是大概的逻辑大家应该都清楚了。

我们将实例A的属性值b赋值完后添加到一级缓存之后，我们就可以从一级缓存获取到实例A，最后完成整个对象的创建。

## 3.4 为什么需要三级缓存，二级行不行?

**一级缓存**：存储的是完整实例

**二级缓存**：存储的是半成品实例，也就是还未属性赋值的实例

**三级缓存**：存储的是``getEarlyBeanReference()``函数表达式，该表达式可以获取到bean早期的bean引用，并判断该bean是否需要代理，最终返回代理对象或普通对象

``getEarlyBeanReference``函数表达式，执行该表达式可以获取到bean早期的bean引用，并判断该bean是否需要代理，简单地说就是，如果一个bean我们定义了他需要代理，那么通过三级缓存的函数表达式执行获取到的就是代理对象，如果不需要代理，获取到的就是普通对象。

所以说我们只用二级缓存行不行其实取决你到底需不需要代理对象。

如果需要代理，我们则在创建过程中其实是创建了两个bean，一个是普通bean，一个是代理bean。**所以他就会在通过调用getEarlyBeanReference()函数表达式想要用你这个bean的时候，将代理bean覆盖掉普通bean，保证你这个bean的全局唯一**。如果说你不需要代理，那可以，你可以不需要三级缓存，二级缓存足以



# 四. JDK动态代理源码分析

## 4.1 JDK动态代理的实现

**需要动态代理的接口**

```java
/**
 * 需要动态代理的接口
 */
public interface Movie {

    void player();

    void speak();
}
```

**需要动态代理的接口的真实实现**

```java
/**
 * 需要动态代理接口的真实实现
 */
public class RealMovie implements Movie {

    @Override
    public void player() {
        System.out.println("看个电影");
    }

    @Override
    public void speak() {
        System.out.println("说句话");
    }
}
```

**动态代理处理器**

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
     * 对真实实现的目标方法进行增强
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

**创建代理对象**

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
        //第一个参数：真实实现的类加载器
        //第二个参数：真实实现类它所实现的所有接口的数组
        //第三个参数：动态代理处理器
        Movie movie = (Movie) Proxy.newProxyInstance(realMovie.getClass().getClassLoader(),
                realMovie.getClass().getInterfaces(),
                handler);
        movie.player();

    }

}
```

**结果**

![image-20210819173404132](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173404132.png)

由于设置 ``sun.misc.ProxyGenerator.saveGeneratedFiles`` 的值为``true``,所以代理类的字节码内容保存在了项目根目录下，文件名为``$Proxy0.class``

![image-20210819173435065](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173435065.png)

**生成的代理对象字节码文件**

```java
public final class $Proxy0 extends Proxy implements Movie {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m4;
    private static Method m0;
    
    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("com.eayon.dynamic.Movie").getMethod("player");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m4 = Class.forName("com.eayon.dynamic.Movie").getMethod("speak");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    /**
     * 重写被代理接口的方法
     * 因为生成的代理对象会实现被代理接口，所以我们在外部可以直接通过代理对象嗲偶哦那个被代理接口中的方法
     */
    public final void speak() throws  {
        try {
            //当外部通过代理对象调用被代理接口的方法时，其实是通过invocationHandler中的invoke()方法去调用的。
            //这个h就是invocationHandler（我们之前创建的MyInvocationHandler代理处理器）
            //this就是当前这个Proxy0代理对象
            //m4则具体要调用的方法
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
    
    //与上面的speak方法同理，都是重写的被代理接口中的方法
    public final void player() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
    
    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }
}
```

**代理对象的特点**

1、代理类继承了Proxy类并且**代理对象和真实实现一样都实现了要代理的接口**

2、重写了equals、hashCode、toString

3、有一个静态代码块，通过反射或者代理类的所有方法

4、通过invoke执行代理类中的目标方法doSomething

## 4.2 源码分析

从上述代码中不难看出，创建代理对象的关键代码为：

```java
//获取动态代理对象
//第一个参数：真实实现的类加载器
//第二个参数：真实实现类它所实现的所有接口的数组
//第三个参数：动态代理处理器
Movie movie = (Movie) Proxy.newProxyInstance(realMovie.getClass().getClassLoader(),
        realMovie.getClass().getInterfaces(),
        handler);
```

然后当执行如下代码的时候，也就是当代理对象调用真实实现的方法时，会自动跳转到动态代理处理器的invoke方法来进行调用。

```java
movie.player();
```

这是为什么呢？

那其实归根结底都在``Proxy.newProxyInstance() ``方法创建代理对象的源码中，我们一起来看看做了些什么

**Proxy.newProxyInstance()**

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    //判断代理处理器是否为空，为空则抛出空指针异常
    Objects.requireNonNull(h);
    
    //将真实实现类它所实现的所有接口的数据进行拷贝
    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    //生成接口的代理对象的字节码文件（主要方法）
    Class<?> cl = getProxyClass0(loader, intfs);

    /*
     * 使用自定义的InvocationHandler作为参数，调用构造函数获取代理对象示例
     */
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }

        //通过代理对象的字节码文件获取代理对象的构造器
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        //通过代理对象的构造器调用newInstance()反射获取代理对象实例
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

由上述代码我们发现，主要是通过``getProxyClass0()``方法去获取或者创建代理对象的字节码文件，通过代理对象字节码文件获取其构造器并通过反射生成代理对象实例。

那现在的重点就是如何获取或者创建代理对象的字节码文件，我们继续往下。

**getProxyClass0(loader, intfs)**

那其实真正生成代理对象字节码文件的是这个方法，他会传入真实实现的类加载器和他所实现的接口数组。

```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    //限制真实实现所实现的接口数量不能大于65535个
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // 首先从缓存中获取该接口对于的代理对象，如果有则返回，没有则通过ProxyClassFactory创建
    return proxyClassCache.get(loader, interfaces);
}
```

**ProxyClassFactory**

缓存中获取我们比较好理解，但是我们并没有在上述方法中发现proxyClassFactory

我们可以点击进入``proxyClassCache.get()``方法看看是如何从缓存中获取的

```java
public V get(K key, P parameter) {
    Objects.requireNonNull(parameter);

    expungeStaleEntries();
    
    // 这里我们就理解成将真实实现的类加载器作为缓存key即可
    Object cacheKey = CacheKey.valueOf(key, refQueue);

    // 从缓存中获取代理对象
    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    if (valuesMap == null) {
        ConcurrentMap<Object, Supplier<V>> oldValuesMap
            = map.putIfAbsent(cacheKey,
                              valuesMap = new ConcurrentHashMap<>());
        if (oldValuesMap != null) {
            valuesMap = oldValuesMap;
        }
    }

    // 缓存中不存在则根据subKeyFactory.apply(key, parameter)方法进行创建
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    Supplier<V> supplier = valuesMap.get(subKey);

    .......省略无用代码........
}
```

**subKeyFactory.apply(key, parameter)**

我们点击进入apply方法发现其实是BiFunction接口，我们找到它的实现

![image-20210819173614848](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173614848.png)

此时我们发现，我们进入的这个apply方法所在的位置是Proxy类下ProxyClassFactory这个静态内部类中

所以当缓存中没有相应的代理对象，则会调用ProxyClassFactory类的``apply``方法来创建代理类。

**ProxyClassFactory.apply()**

```java
private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
    // 生成代理对象的字节码文件名的前缀，用于组装文件名
    private static final String proxyClassNamePrefix = "$Proxy";

    // 生成代理对象字节码文件名的计数器，用于组装文件名(计数器默认从0开始)
    private static final AtomicLong nextUniqueNumber = new AtomicLong();

    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        for (Class<?> intf : interfaces) {
            //校验类加载器是否能通过接口名称加载该类
            Class<?> interfaceClass = null;
            try {
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
            
            //校验该类是否是接口类型
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            
            //校验接口是否重复
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }

        String proxyPkg = null;     // 代理对象包名
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        /*
         * 用于生成代理对象需要使用的包名
         * 非public接口，代理类的包名与接口的包名相同
         */
        for (Class<?> intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }

        if (proxyPkg == null) {
            // 如果代理接口是public修饰的，则使用默认的com.sun.proxy package作为包名
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }

        /*
         * 为代理对象生成字节码文件名
         * 文件名格式：proxyName = 之前生成的包名 + $Proxy + 当前计数器的值(计数器默认从0开始)
         * 比如 proxyName = com.sun.proxy.$Proxy0
         */
        long num = nextUniqueNumber.getAndIncrement();
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        /*
         * 真正生成代理对象字节码文件的地方
         */
         //生成代理对象字节码数组
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            // 将代理对象字节码数组生成字节码文件，并使用类加载器将代理对象的字节码文件加载到JVM内存中
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

我们可以看出它是通过``ProxyGenerator.generateProxyClass() ``先生成代理对象字节码数组，

然后通过``defineClass0()``方法将代理对象的字节码数组生成字节码文件，并将该字节码通过类加载器加载到JVM中。

但是``defineClass0()``方法底层是通过``native``调用的``C++``，我们看不了，知道有这个事就行

![image-20210819173659056](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173659056.png)

**ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags)**

这个方法随便看看就行，不用过多理解

```java
public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
    ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
    final byte[] var4 = var3.generateClassFile();
    // 是否要将生成代理对象的字节码文件保存到磁盘中
    // 该步骤也就是为什么之前我们在测试生成代理对象的时候使用System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");来将代理对象字节码文件保存下来
    if (saveGeneratedFiles) {
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                try {
                    int var1 = var0.lastIndexOf(46);
                    Path var2;
                    if (var1 > 0) {
                        Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar));
                        Files.createDirectories(var3);
                        var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
                    } else {
                        var2 = Paths.get(var0 + ".class");
                    }

                    Files.write(var2, var4, new OpenOption[0]);
                    return null;
                } catch (IOException var4x) {
                    throw new InternalError("I/O exception saving generated file: " + var4x);
                }
            }
        });
    }

    return var4;
}
```

## 4.3 总结

![image-20210819173737409](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173737409.png)

![image-20210819173728693](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210819173728693.png)

创建代理对象的核心方法就是**Proxy.newProxyInstance()**。该方法首先会调用**getProxyClass0() 从缓存中获取或者创建代理对象字节码文件**，拿到代理对象字节码文件后调用**getConstructor()方法获取代理对象的构造器**，最后**通过cons.newInstance() 根据代理对象的构造器反射生成代理对象实例**并返回。

我们回过头来看getProxyClass0()方法，该方法**首先判断真实实现所实现的接口数量是否超限**，没有超限则proxyClassCache.get()**从缓存中获取代理实例**。如果缓存中没有，则去**创建代理对象**。那么是如何创建的呢？首先会根据：**包名 + $proxy0 + 当前计数器的值(默认从0开始) 生成代理对象的名称**，其次根据名称和代理对象需要实现的被代理接口去**生成代理对象字节码数组**。拿到字节码数组之后，就可以通过**调用的native方法去生成代理对象字节码文件**，最后进行返回。