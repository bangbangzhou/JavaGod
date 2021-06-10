# Sentinel服务降级熔断及限流

## 说在前面

>**本章相关代码及笔记地址：**[**飞机票🚀**](https://github.com/EayonLee/JavaGod)
>
>🌍Github：[🚀Java超神之路：【🍔Java全生态技术学习笔记，一起超神吧🍔】](https://github.com/EayonLee/JavaGod)<br>
>🪐CSDN：[🚀Java超神之路：【🍔Java全生态技术学习笔记，一起超神吧🍔】](https://blog.csdn.net/qq_20492277/article/details/114269863)

## 目录
- [Sentinel服务降级熔断及限流](#sentinel服务降级熔断及限流)
  - [说在前面](#说在前面)
  - [目录](#目录)
  - [一. 微服务级联故障服务雪崩](#一-微服务级联故障服务雪崩)
  - [二. Sentinel概要](#二-sentinel概要)
    - [2.1 简介](#21-简介)
  - [三. Sentinel部署及接入](#三-sentinel部署及接入)
    - [3.1 Sentinel-Dashboard控制台部署](#31-sentinel-dashboard控制台部署)
      - [3.1.1 下载](#311-下载)
      - [3.1.2 部署](#312-部署)
    - [3.2 项目如何接入Sentinel](#32-项目如何接入sentinel)
      - [3.2.1 添加依赖](#321-添加依赖)
      - [3.2.2 配置文件](#322-配置文件)
      - [3.2.3 测试](#323-测试)
  - [四. Sentinel基础使用](#四-sentinel基础使用)
    - [4.1 流控规则](#41-流控规则)
      - [4.1.1 基础使用介绍](#411-基础使用介绍)
      - [4.1.2 实现QPS限流](#412-实现qps限流)
      - [4.1.3 自定义限流响应结果](#413-自定义限流响应结果)
      - [4.1.4 自定义限流跳转页面](#414-自定义限流跳转页面)
      - [4.1.5 流控模式：直接流控](#415-流控模式直接流控)
      - [4.1.6 流控模式：关联流控](#416-流控模式关联流控)
      - [4.1.7 流控模式：链路流控](#417-流控模式链路流控)
      - [4.1.8 流控效果：Warm Up预热模式](#418-流控效果warm-up预热模式)
      - [4.1.9 流控效果：排队等待](#419-流控效果排队等待)
    - [4.2 降级规则](#42-降级规则)
      - [4.2.1 慢调用比例](#421-慢调用比例)
      - [4.2.2 异常比例](#422-异常比例)
      - [4.2.3 异常数](#423-异常数)
    - [4.3 热点规则](#43-热点规则)
    - [4.4 系统保护规则](#44-系统保护规则)
    - [4.5 授权规则](#45-授权规则)
  - [五. Sentinel Dashboard控制台通信原理](#五-sentinel-dashboard控制台通信原理)
  - [六. Sentinel两种保护应用的方式](#六-sentinel两种保护应用的方式)
    - [6.1 （默认）拦截所有controller的请求url路径](#61-默认拦截所有controller的请求url路径)
    - [6.2 通过注解保护应用**](#62-通过注解保护应用)
  - [七. Sentinel整合微服务调用](#七-sentinel整合微服务调用)
    - [7.1 Sentinel整合RestTemplate](#71-sentinel整合resttemplate)
    - [7.2 Sentinel整合Feign](#72-sentinel整合feign)
      - [7.2.1 方式一：fallback限流降级处理类](#721-方式一fallback限流降级处理类)
      - [7.2.2 方式二：fallbackFactory工厂](#722-方式二fallbackfactory工厂)
  - [八. Sentinel持久化](#八-sentinel持久化)
    - [8.1 pull模式持久化到本地文件](#81-pull模式持久化到本地文件)
    - [8.2 push模式持久化到Nacos（生产推荐）](#82-push模式持久化到nacos生产推荐)

## 一. 微服务级联故障服务雪崩

在分布式系统里，许多服务之间通过远程调用实现信息交互，调用时不可避免会出现调用失败，比如超时、异常等原因导致调用失败，Sentinel能够保证在一个服务出问题的情况下，不会导致整体服务失败，避免级联故障（服务雪崩），以提高分布式系统的弹性；

**服务雪崩举例**：

比如电商中的用户下订单，我们有两个服务，一个下订单服务，一个减库存服务，当用户下订单时调用下订单服务，然后下订单服务又调用减库存服务，如果减库存服务响应延迟或者没有响应，则会造成下订单服务的线程挂起等待，如果大量的用户请求下订单，或导致大量的请求堆积，引起下订单服务也不可用，如果还有另外一个服务依赖于订单服务，比如用户服务，它需要查询用户订单，那么用户服务查询订单也会引起大量的延迟和请求堆积，导致用户服务也不可用。

所以在微服务架构中，很容易造成服务故障的蔓延，引发整个微服务系统瘫痪不可用。

**常用的容错方案或思想**：

- 超时，设置比较短的超时时间，调用不成功，很短时间就释放线程，避免大量线程堵塞等待，导致服务cpu、内存等资源飙高；(快速失败)
- 限流，超过设置的阈值就拒绝，比如评估系统的QPS是3000，那么就可以设置限流阈值是2800；
- 仓壁保护，就是一艘船不是一个船舱，而是把一个船舱划分为多个船舱，某个船舱进水了，其他船舱都不受到影响；
- 断路器，熔断器也有叫断路器，他们表示同一个意思，最早来源于微服务之父 Martin Fowler 的论文 CircuitBreaker 一文，“熔断器”本身是一种开关装置，用于在电路上保护线路过载，当线路中有电器发生短路时，能够及时切断故障电路，防止发生过载、发热甚至起火等严重后果。

## 二. Sentinel概要

### 2.1 简介

Sentinel：阿里巴巴开源的轻量级的流量控制、熔断降级Java 组件，是分布式系统的流量防卫兵；

![image-20210609181159854](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181159854.png)

随着微服务的流行，服务与服务之间的调用稳定性变得越来越重要；

- 当服务访问量达到一定程度，流量扛不住的时候，该如何处理？
- 服务之间相互依赖，当服务A出现响应时间过长，影响到服务B的响应，进而产生连锁反应，直至影响整个依赖链上的所有服务，该如何处理？

这是分布式、微服务开发不可避免的问题，Sentinel以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性；

**Sentinel主要组成**

1. 核心库（Java 客户端）：Sentinel的核心库不依赖任何第三方框架/库，能够运行于所有 Java环境，同时对 Dubbo / SpringBoot / Spring Cloud 等框架也有很好的支持；
2. 控制台（Dashboard）基于 Spring Boot 开发，打包后可以直接运行，不需要额外的 Tomcat 等应用容器；

## 三. Sentinel部署及接入

### 3.1 Sentinel-Dashboard控制台部署

Sentinel 控制台是流量控制、熔断降级规则统一配置和管理的入口，它为用户提供了机器自发现、簇点链路自发现、监控、规则配置等功能。在 Sentinel 控制台上，我们可以配置规则并实时查看流量控制效果。

#### 3.1.1 下载

Sentinel Github地址：https://github.com/alibaba/Sentinel

![image-20210609181232686](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181232686.png)

进入最新版本

![image-20210609181246816](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181246816.png)

下载Sentinel-Dashboard的jar包

#### 3.1.2 部署

**注意**：sentinel控制台需要在JDK 1.8+的版本上运行。

**上传sentinel-dashboard.jar到服务器**

![image-20210609181302033](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181302033.png)

**使用如下命令启动**

```shell
java -Dserver.port=19090 -Dcsp.sentinel.dashboard.server=localhost:19090 -Dproject.name=sentinel-dashboard -jar /opt/sentinel-dashboard/sentinel-dashboard-1.8.1.jar &
```

``其中 -Dserver.port=19090 用于指定 Sentinel 控制台端口为 19090。（如果不指定端口默认为8080）``

**登录测试**

默认账号密码都是：``sentinel``

![image-20210609181349133](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181349133.png)

### 3.2 项目如何接入Sentinel

#### 3.2.1 添加依赖

给消费者添加该依赖就可以针对消费者访问进行流量规则配置

给提供者添加该依赖就可以针对消费者访问提供者时进行流量规则配置

那么我这里在消费者和提供者都加入该依赖

```yaml
<!--sentinel-服务降级熔断及限流-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

#### 3.2.2 配置文件

那么我这里在消费者和提供者都加入该配置

```yaml
#指定sentinel-dashboard控制台的连接地址
spring.cloud.sentinel.transport.dashboard=172.17.70.29:19090
```

如下只演示了添加消费者的配置，提供者同理自行配置

![image-20210609181454008](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181454008.png)

#### 3.2.3 测试

将消费者和提供者重启，然后查看sentinel-dashboard控制台

但重启项目之后其实在控制台还是看不到我们的项目信息的，**我们需要访问一下提供者和消费者的接口**，让信息加载到Sentinel中

![image-20210609181529364](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181529364.png)

由于我们刚刚访问了提供者和消费者的接口，所以在实时监控里面应该有我们的请求信息的展示，但现在是没有的

![image-20210609181542656](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181542656.png)

我们查看Sentinel的日志可以发现这可能是sentinel服务端无法通过``IP:8719``端口访问到我们项目中的sentinel客户端，从而拉取不到接口访问信息。

![image-20210609181600516](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181600516.png)

我们的项目是在本地电脑上启动的，那我们看一下注册到Sentinel中该项目的 IP地址和端口

![image-20210609181620809](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181620809.png)

我们发现端口号没问题，是``8719（每个项目启动后默认分配的端口是8719，如果该端口被占用则会自增+1 直到可用，所以另一个服务的端口是8720）``。

但是192.168.159.1 这个IP肯定不是我们本机的IP，我们可以查看一下本机IP

![image-20210609181651757](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181651757.png)

所以很明显，sentinel服务端是访问不同客户端的。

**解决：**

指定项目的Sentinel客户端IP地址

![image-20210609181708130](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181708130.png)

**查看：**

![image-20210609181720669](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181720669.png)

我们再次访问项目的接口 刷一点接口请求，然后再去查看一下实时监控是否展示数据

![image-20210609181735888](C:\Users\Eayon\AppData\Roaming\Typora\typora-user-images\image-20210609181735888.png)

**注意**：

如果是本地测试的话建议将sentinel-dashboard部署到本机，不要部署到云服务器或者虚拟机，因为sentinel-dashboard需要连接访问我们项目的IP:8719端口去拉取项目接口请求数据，如果将sentinel-dashboard部署在虚拟机或者云服务器上的话它获取到我们服务的IP可能有误，这样从云服务器或者虚拟机是访问不了本机启动服务的8719端口的。就导致虽然服务注册到了sentinel，但是没有任何请求数据的情况。

所以还是将控制台部署到本地比较方便。如果是线上的话就没有问题。因为服务器都在一个网段

**8719端口是什么？**

在我们项目引入sentinel连接dashboard之后，会默认给该项目开启一个8719端口供sentinel服务端来访问项目的sentinel客户端拉取该项目接口请求数据使用的。如果该项目部署所在服务器已存在8719端口，那么会进行自增变成8720，同理8720也被占用则继续自增，直至可用为止。所以我们需要特定的去配置该端口，只需要知道就可以

## 四. Sentinel基础使用

### 4.1 流控规则

#### 4.1.1 基础使用介绍

我们先访问一下服务的接口 刷一些请求。

[http://localhost:18083/getData](http://localhost:18084/getData)2

可以发现我对这个 ``getData2`` 接口每分钟请求成功了15次，那么我们就可以对这个 ``getData2`` 请求进行流量控制。

![image-20210609181823556](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181823556.png)

**流量控制**

![image-20210609181856709](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181856709.png)

新增后 在左侧菜单栏的流控规则中就会有针对该接口的流量控制配置

**流量控制配置参数详解**

- **资源名：**

- - 不设置的话默认使用接口名称

- **针对来源：**

- - default：代表所有的服务进行限流，不管你是哪个服务都会进行流量控制

- **阈值类型：**

- - QPS阈值：如果服务QPS阈值设置为2，那么表示1秒内超过两个请求则进行限流
  - 线程数阈值：如果服务线程阈值设置为2，那么表示1秒内超过两个线程则进行限流

- **是否集群：**

- - 集群阈值模式 - 单机均摊：该配置和上面的阈值类型是相关的，比如集群阈值模式设置为单机均摊，并且上面阈值类型为QPS阈值，阈值数为2则表示：单个服务节点每秒超过两个请求就限流
  - 集群阈值模式 - 总体均摊：如集群阈值模式设置为 总体均摊，并且上面阈值类型为QPS阈值，阈值数为2则表示：所有服务节点每秒超过两个请求就限流

- **高级 - 流控模式：**

流控模式是与下面流控效果相关的

- - 直接：字面意思  就是直接的意思，如果QPS阈值为2并且流控模式为直接 流控效果为快速失败的时候，请求了高于QPS阈值2 则会直接返回访问失败
  - 关联：关联某一个资源，举例：当前getData2资源关联了getData3资源后，getData3接口资源的单机QPS阈值达到了设置阈值，将当前资源getData2接口进行限流
  - 链路：记录指定链路上的流量

- **高级 - 流控效果**

- - 快速失败：直接返回请求失败
  - Warm Up：预热模式根据coldFactor（加载因子 默认为3）的值，根据单机阈值除于coldFactor，经过预热的时长达到设置的QPS阈值，比如设置QPS单机阈值为100，那么100/3 =33，用33作为最初的阈值，然后在10秒到达100后再开始限流；
  - 排队等待：排队模式，在QPS阈值到达后，新的请求就等待，直到超时，可以适用于突发流量的请求；

#### 4.1.2 实现QPS限流

比如我们将getData2接口进行了限流，流控规则如下

![image-20210609181916372](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181916372.png)

那么我们疯狂请求`` getData2`` 接口进行测试

![image-20210609181935162](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181935162.png)

会发现，他会返回给我们该请求被限流的警告，但是日常工作中我们肯定不会这样直接返回，需要进行处理，可以看下面如何进行处理

#### 4.1.3 自定义限流响应结果

**在服务中创建自定义限流降级信息返回处理类**

![image-20210609181955787](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609181955787.png)

```JAVA
import com.alibaba.csp.sentinel.adapter.spring.webmvc.callback.BlockExceptionHandler;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import com.alibaba.csp.sentinel.slots.block.authority.AuthorityException;
import com.alibaba.csp.sentinel.slots.block.degrade.DegradeException;
import com.alibaba.csp.sentinel.slots.block.flow.FlowException;
import com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowException;
import com.alibaba.csp.sentinel.slots.system.SystemBlockException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.HashMap;
import java.util.Map;

/**
 * @author zhengtai.li
 * @ClassName MyBlockExceptionHandler
 * @Description 自定义限流降级返回错误信息
 * @Copyright 2021 © kuwo.cn
 * @date 2021/5/24 18:50
 * @Version: 1.0
 */
@Component
public class MyBlockExceptionHandler implements BlockExceptionHandler{

    /**
     * 针对限流后返回错误提示信息
     */
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, BlockException e) throws Exception {
        //log.info("UrlBlockHandler.....................................");
        Map restObject = new HashMap();

        // 不同的异常返回不同的提示语
        if (e instanceof FlowException) {
            restObject.put("code",100);
            restObject.put("msg","接口限流了");

        } else if (e instanceof DegradeException) {
            restObject.put("code",101);
            restObject.put("msg","服务降级了");

        } else if (e instanceof ParamFlowException) {
            restObject.put("code",102);
            restObject.put("msg","热点参数限流了");

        } else if (e instanceof SystemBlockException) {
            restObject.put("code",103);
            restObject.put("msg","触发系统保护规则");

        } else if (e instanceof AuthorityException) {
            restObject.put("code",104);
            restObject.put("msg","授权规则不通过");
        }

        //返回json数据
        response.setStatus(500);
        response.setCharacterEncoding("utf-8");
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        //springmvc 的一个json转换类 （jackson）
        new ObjectMapper().writeValue(response.getWriter(), restObject);
    }

}
```



重启服务进行测试

![image-20210609182019027](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182019027.png)

#### 4.1.4 自定义限流跳转页面

同样还是如上那个实现类，只不过最后不反悔json相应内容了 而重定向页面

```java
import com.alibaba.csp.sentinel.adapter.spring.webmvc.callback.BlockExceptionHandler;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import com.alibaba.csp.sentinel.slots.block.authority.AuthorityException;
import com.alibaba.csp.sentinel.slots.block.degrade.DegradeException;
import com.alibaba.csp.sentinel.slots.block.flow.FlowException;
import com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowException;
import com.alibaba.csp.sentinel.slots.system.SystemBlockException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.HashMap;
import java.util.Map;

/**
 * @author zhengtai.li
 * @ClassName MyBlockExceptionHandler
 * @Description 自定义限流降级返回错误信息
 * @Copyright 2021 © kuwo.cn
 * @date 2021/5/24 18:50
 * @Version: 1.0
 */
@Component
public class MyBlockExceptionHandler implements BlockExceptionHandler {

    /**
     * 针对限流后返回错误提示信息
     */
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, BlockException e) throws Exception {
        //log.info("UrlBlockHandler.....................................");
        Map restObject = new HashMap();

        // 不同的异常返回不同的提示语
        if (e instanceof FlowException) {
            restObject.put("code", 100);
            restObject.put("msg", "接口限流了");

        } else if (e instanceof DegradeException) {
            restObject.put("code", 101);
            restObject.put("msg", "服务降级了");

        } else if (e instanceof ParamFlowException) {
            restObject.put("code", 102);
            restObject.put("msg", "热点参数限流了");

        } else if (e instanceof SystemBlockException) {
            restObject.put("code", 103);
            restObject.put("msg", "触发系统保护规则");

        } else if (e instanceof AuthorityException) {
            restObject.put("code", 104);
            restObject.put("msg", "授权规则不通过");
        }

        //返回json数据
        //response.setStatus(500);
        //response.setCharacterEncoding("utf-8");
        //response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        //springmvc 的一个json转换类 （jackson）
        //new ObjectMapper().writeValue(response.getWriter(), restObject);

        request.getRequestDispatcher("/index.jsp").forward(request, response);
    }

}
```

![image-20210609182051528](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182051528.png)

**创建Index页面**

![image-20210609182106091](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182106091.png)

**重启测试**

![image-20210609182115862](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182115862.png)

如果是前后端分离项目  我们可以让前端同学直接判断我们返回的状态码，由前端的同学去写前端页面和跳转的操作，一般也都是这样去搞得

#### 4.1.5 流控模式：直接流控

线程阈值设置为5

![image-20210609182145862](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182145862.png)

这里需要超过5个线程去同时访问测试，那么我们通过浏览器测试是不行的，大家可以通过JMeter去进行多线程访问测试。

#### 4.1.6 流控模式：关联流控

![image-20210609182216830](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182216830.png)

将当前`` getData2``接口资源 和 ``getData3``接口资源进行关联，当``getData3``的单机QPS阈值达到2时`` getData2``就会进行限流

#### 4.1.7 流控模式：链路流控

![image-20210609182250651](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182250651.png)

**注意：**

入口资源：``sentinel_spring_web_context`` 代表的是该流控接口 ``getData2``的上级簇点链路，如下图

![image-20210609182322579](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182322579.png)

**解释：**

``sentinel_spring_web_context``这个入口资源下会有多个接口资源，如上图的getData2和getData3

我们的流控规则为当``sentinel_spring_web_context``这个入口资源的单机QPS阈值达到 2 时会对getData2接口资源进行限流，流控效果为快速现有失败

只要是``sentinel_spring_web_context``这个入口资源下的所有接口资源的流量都会汇总到入口资源上

#### 4.1.8 流控效果：Warm Up预热模式

![image-20210609182347005](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182347005.png)

预热模式根据coldFactor（加载因子 默认为3）的值，根据单机阈值除于coldFactor，经过预热的时长达到设置的QPS阈值，比如设置QPS单机阈值为100，那么100/3 =33，用33作为最初的阈值，然后在10秒到达100后再开始限流；

#### 4.1.9 流控效果：排队等待

![image-20210609182422405](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182422405.png)

当 getData2接口资源的单机QPS阈值达到2时不会直接进行限流，而是进行排队等待500毫秒，如果超过500毫秒服务还是无法处理本次请求则进行限流处理返回错误提示。

### 4.2 降级规则

- **异常比例**：异常比例 (DEGRADE_GRADE_EXCEPTION_RATIO)是指当资源的每秒异常总数占通过量的比值超过阈值（DegradeRule 中的 count）之后，资源进入降级状态，即在接下的时间窗口（DegradeRule 中的 timeWindow，以 s 为单位）之内，对这个方法的调用都会自动地返回，异常比率的阈值范围是 [0.0, 1.0]，代表 0% - 100%；
- **异常数**：异常数 (DEGRADE_GRADE_EXCEPTION_COUNT)是指当资源近1分钟的异常数目超过阈值之后会进行熔断，注意由于统计时间窗口是分钟级别的，若timeWindow小于 60s，则结束熔断状态后仍可能再进入熔断状态；

#### 4.2.1 慢调用比例

![image-20210609182504692](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182504692.png)

**执行逻辑**

比如我对getData2资源设置降级规则，此时如果服务器在统计时长1000ms内接收到的请求超过最小请求数5次，并且有4次（该4次是通过比例阈值和最小请求数算出来的）调用时间超过最大RT200ms则触发降级，1s内对资源进行熔断，这1s内该资源无法被访问，访问后会返回我们自定义的错误响应信息或页面。

![image-20210609182522221](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182522221.png)

#### 4.2.2 异常比例

![image-20210609182535177](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182535177.png)

**执行逻辑**

如果服务器在统计时长1000ms内接收到的请求超过最小请求数5，并且有4次（该4次是通过比例阈值和最小请求数算出来的）请求异常的情况下会对该getData2进行1秒的资源熔断，1秒内该资源无法被访问，访问后会返回我们自定义的错误响应信息或页面。

![image-20210609182547791](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182547791.png)

#### 4.2.3 异常数

![image-20210609182606835](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182606835.png)

**执行逻辑**

如果服务器统计时长1000ms内接收到的请求超过最小请求数5，并且异常数超过2的话会进行1秒的熔断，1秒内该资源无法被访问，访问后会返回我们自定义的错误响应信息或页面。

![image-20210609182620820](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182620820.png)

### 4.3 热点规则

何为热点？热点即经常访问的数据，很多时候我们希望统计某个热点数据中访问频次最高的 Top K 数据，并对其访问进行限制

**比如：**

商品 ID 为参数，统计一段时间内最常购买的商品 ID 并进行限制；

用户 ID 为参数，针对一段时间内频繁访问的用户 ID 进行限制；

热点参数限流会统计传入参数中的热点参数，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流，热点参数限流可以看做是一种特殊的流量控制，仅对包含热点参数的资源调用生效；

**创建一个测试热点参数接口：**

![image-20210609182638466](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182638466.png)

该接口的路径为：``/getData5``

``@SentinelResource``注解一定要加，否则热点规则限流不生效

**注解参数详解：**

- value：该value会在Sentinel中为该接口资源 /getData5生成一个子资源 getData5，一般value属性值与接口路径保持一致（去掉斜杠）。
- fallback：当该接口进行热点规则限流后会进入到fallback降级处理类（这里是MyBlockHotFallback）中的fallback的静态方法，该方法名可自行定义
- fallbackClass：当该接口进行热点规则限流后会进入到该MyBlockHotFallback降级处理类中的名字为fallback的静态方法，该方法名可自行定义

**降级处理类：**

![image-20210609182703885](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182703885.png)

**普通测试**

我们先刷一下 /getData5接口，然后在Sentinel簇点链路中找到该接口资源 /getData5 和他的子资源 getData5 ，这个getData5资源就是注解中value的属性值

![image-20210609182730365](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182730365.png)

我们想要对该接口进行参数热点规则限流的话 只能对通过``@SentinelResource``注解生成该接口的子资源进行热点配置，也就是getData5资源 而不是 /getData5（没有斜杠的那一个）

![image-20210609182743367](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182743367.png)

当getData5资源的下标为0的参数在1秒内超过单机阈值2次的访问会进行限流

![image-20210609182754743](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182754743.png)

**高级测试**

![image-20210609182807722](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182807722.png)

当 getData5资源 下标为0的参数每秒请求超过单机阈值5会进行限流，并且如果该资源下标为0的参数值为66的话，每秒请求数大于限流阀值2也会进行限流

![image-20210609182819067](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182819067.png)

**注意**：

如果是如下参数值的阈值大于参数阈值的情况

![image-20210609182836482](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182836482.png)

比如下标0的参数值66的阈值为100，下标0的参数的总体阈值为2的话，我们请求参数为66时，它是只会走下面那个配置的，所以只有当阈值达到100的时候才会限流，到达2的时候不会限流

![image-20210609182848879](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609182848879.png)



### 4.4 系统保护规则

![image-20210609183016066](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183016066.png)

- **LOAD**：只有在 Linux 系统的机器上才会生效，可以根据当前操作系统的负载，来决定是否触发保护（可通过uptime命令查看linux的负载）；
- **RT**：当单台机器上该应用所有请求的平均响应时间达到阈值就会触发系统保护停止所有接口请求（阈值单位是毫秒）
- **线程数**：当单台机器上该应用，所有的请求消耗的线程数加起来，如果超过阈值，就停止所有接口请求（阈值单位是个）
- **入口 QPS**：当单台机器上该应用，所有接口的 QPS 加起来，如果超过阈值，就停止所有接口请求（阈值单位是个）
- **CPU 使用率**：当该应用所在机器的CPU使用率，如果超过阈值，就停止所有接口请求（阈值取值范围0.0~1.0）

发生系统规则中配置的情况的时候，会把整个应用都断掉，所有的接口对不能对外提供服务了，这个设计很少用，因为粒度太大了，用 Sentinel 一般都是做细粒度的维护，如果设置了系统规则，可能自己都不知道怎么回事，系统就用不了了；

### 4.5 授权规则

授权规则就是可以针对资源进行权限管理，比如 /getData2这个接口资源只允许某应用进行调用，或只不允许某应用调用

![image-20210609183036726](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183036726.png)

请求中携带 order应用标识的请求访问 /getData2资源时进行黑名单限制。

调用方可以在请求参数中直接携带 他自己的应用标识，也可以在请求header中传递。

当然如果调用方不传递应用参数，而提供者没有具体限制必传的话调用也是没问题的

**在 /getData2资源所属服务中添加解析应用来源接口**

![image-20210609183051433](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183051433.png)

如上服务可以从请求参数中或者请求header中获取，那么调用者也就有两种传递应用参数的方式。当然如果调用方不传递应用参数，而提供者没有具体限制必传的话调用也是没问题的

```java
import com.alibaba.csp.sentinel.adapter.spring.webmvc.callback.RequestOriginParser;
import org.apache.commons.lang3.StringUtils;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;

/**
 * @author zhengtai.li
 * @ClassName MyRequestOriginParser
 * @Description 解析来源处理器（主要用于Sentinel的授权规则）
 * @Copyright 2021 © kuwo.cn
 * @date 2021/5/27 14:40
 * @Version: 1.0
 */
@Component
public class MyRequestOriginParser implements RequestOriginParser {

    @Override
    public String parseOrigin(HttpServletRequest request) {
        //从请求参数中获取参数
        String origin = request.getParameter("origin");
        //如果请求参数中没有从请求header中获取
        if(StringUtils.isBlank(origin)){
            origin = request.getHeader("origin");
        }

        //还可以在这里限制一下  没有应用标识直接拦截或抛出异常
        /*if(StringUtils.isBlank(origin)){
            throw new IllegalArgumentException("origin参数为指定");
        }*/

        return origin;
    }
}
```



**测试**

返回的结果是我们自定义的限流响应结果，不是sentinel默认的。

![image-20210609183110724](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183110724.png)

## 五. Sentinel Dashboard控制台通信原理

![image-20210609183130952](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183130952.png)

当微服务启动之后通过``sentinel-transport-simple-http``这个依赖将该服务的一些端点信息注册到``sentinel dashboard``中

然后该服务由于引用了sentinel的依赖，注册后会在该服务中默认启动一个8719端口，但是如果该服务所在机器已经存在8719端口则会递增加1，直至到可用为止

那么微服务通过8719端口会向sentinel暴露很多http接口（可通过 服务IP:8719/api 这个地址查看微服务通过8719向sentinel暴露了哪些接口）

![image-20210609183156516](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183156516.png)

然后sentinel dashboard就会调用微服务暴露的这些接口去获取服务信息，比如我们可以看服务暴露的第一个接口如下

![image-20210609183207681](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183207681.png)

该接口可以通过资源名称获取资源信息，那么我们来试一下我们服务中的一个接口资源

![image-20210609183223537](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183223537.png)

那么sentinel dashboard就是这样与我们的服务进行通讯获取到各种信息的

**在sentinel中设置的限流规则是如何告知微服务的？**

当然也是通过微服务暴露的接口进行告知的，也就是如下 setRules接口，将在dashboard设置的规则告知给微服务，然后微服务会将该规则缓存在内存中。

然后dashboard通过 /getRules接口再去微服务中获取该服务的一些降级限流规则，这也说明了为什么我们通常重启服务后，该服务之前设置的规则就会不见。

（通常我们都会将sentinel设置的规则信息持久化到Nacos）

![image-20210609183232354](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183232354.png)

## 六. Sentinel两种保护应用的方式

### 6.1 （默认）拦截所有controller的请求url路径

Sentinel为springboot程序提供了一个starter依赖，由于sentinel starter依赖默认情况下就会为所有的HTTP接口提供限流埋点，所以在springboot 中的Controller都可以受到Sentinel的保护；当然，我们还需要在Sentinel Dashboard中配置限流的保护规则。

**实现原理：**

他的实现原理就是，底层通过``com.alibaba.csp.sentinel.adapter.spring.webmvc.AbstractSentinelInterceptor``这个拦截器对所有的Controller接口进行了拦截

![image-20210609183319933](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183319933.png)

再拦截方法 preHandle 中主要通过如下方法进行判断触发了哪种行为（流量、参数热点、降级、权限、系统规则）

![image-20210609183350231](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183350231.png)

那到底是如何进行判断的呢？我们可以进入entry方法看一下，

我们可以发现该方法会抛出 ``BlockException`` 异常，那么该方法也就是通过拦截到你这个controller接口得到接口资源名称，然后根据sentinel配置的规则进行判断你触发了哪一项（流量、参数热点、降级、权限、系统规则）规则，然后报该项限流规则所对应的异常

![image-20210609183405453](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183405453.png)

我们可以看一下 BlockExecption 的几种子异常，分别对应了 流量、参数热点、降级、权限、系统规则

![image-20210609183419554](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183419554.png)

再根据不同的降级规则做不同的降级处理。

**当然我们也可以通过配置关闭它对所有Controller接口拦截保护**

\#关闭sentinel对controller的url的保护    false关闭保护  默认true开启保护 spring.cloud.sentinel.filter.enabled=false

### 6.2 通过注解保护应用**

在前面我们有自定义限流响应结果，但是他的影响度是全局的，也就是说，只要给某个接口设置了限流规则，当限流之后都会统一的走这个自定义限流响应。

同理，如果不设置自定义限流响应，就会走默认的响应结果。

那么对于我们有特殊需求的接口，对于限流之后，有与众不同的响应的话，我们就需要通过注解的方式单独为此接口声明 fallback 异常处理逻辑

![image-20210609183435767](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183435767.png)

**fallbackClass 限流处理类**

![image-20210609183450123](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183450123.png)

**Sentinel限流规则配置**

必须要为 getData7 这个不带 斜杠的资源进行流控设置，如果对 /getData7资源进行流控，走的是全局统一默认的限流处理。只有对getData7资源进行流控配置，才会走我们刚刚自定义的限流处理类中的处理方法

![image-20210609183519023](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183519023.png)

![image-20210609183532424](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183532424.png)

**测试**

最终是进入我们单独为该接口配置的限流处理类

![image-20210609183542692](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183542692.png)

**访问其他接口资源测试**

进入的是自定义的全局限流处理类（就算没有自定义限流处理类，也会进入全局默认的处理类）

![image-20210609183553003](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183553003.png)

**@SentinelResource注解中的参数详解：**

| **属性**           | **作用**                                                     | **是否必须** |
| ------------------ | ------------------------------------------------------------ | ------------ |
| value              | 资源名称                                                     | 是           |
| entryType          | entry类型，标记流量的方向，取值IN/OUT，默认是OUT             | 否           |
| blockHandler       | 处理BlockException的函数名称。函数要求：1. 必须是 public2.返回类型与原方法一致3. 参数类型需要和原方法相匹配，并在最后加 BlockException 类型的参数。4. 默认需和原方法在同一个类中。若希望使用其他类的函数，可配置 blockHandlerClass ，并指定blockHandlerClass里面的方法。 | 否           |
| blockHandlerClass  | 存放blockHandler的类。对应的处理函数必须static修饰，否则无法解析，其他要求：同blockHandler。 | 否           |
| fallback           | 用于在抛出异常的时候提供fallback处理逻辑。fallback函数可以针对所有类型的异常（除了 exceptionsToIgnore 里面排除掉的异常类型）进行处理。函数要求：1. 返回类型与原方法一致2. 参数类型需要和原方法相匹配，Sentinel 1.6开始，也可在方法最后加 Throwable 类型的参数。3.默认需和原方法在同一个类中。若希望使用其他类的函数，可配置 fallbackClass ，并指定fallbackClass里面的方法。 | 否           |
| fallbackClass      | 存放fallback的类。对应的处理函数必须static修饰，否则无法解析，其他要求：同fallback。 | 否           |
| defaultFallback    | 用于通用的 fallback 逻辑。默认fallback函数可以针对所有类型的异常（除了 exceptionsToIgnore 里面排除掉的异常类型）进行处理。若同时配置了 fallback 和 defaultFallback，以fallback为准。函数要求：1. 返回类型与原方法一致2. 方法参数列表为空，或者有一个 Throwable 类型的参数。3. 默认需要和原方法在同一个类中。若希望使用其他类的函数，可配置 fallbackClass ，并指定 fallbackClass 里面的方法。 | 否           |
| exceptionsToIgnore | 指定排除掉哪些异常。排除的异常不会计入异常统计，也不会进入fallback逻辑，而是原样抛出。 | 否           |
| exceptionsToTrace  | 需要trace的异常                                              | Throwable    |

## 七. Sentinel整合微服务调用

### 7.1 Sentinel整合RestTemplate

**在application.yaml中加入如下配置（默认开启，可以不配置）**

```yaml
# 开启sentinel对restemplate的支持  true开启（默认）  false关闭
resttemplate:
  sentinel:
    enabled: true
```

**在主启动类实例化RestTemplate方法上加注解**

```java
@Bean
//如果不加限流处理类就走默认的
@SentinelRestTemplate(//blockHandler = "block", blockHandlerClass = RestTemplateBlockHandler.class,//限流用blockHandler
        fallback = "fallback", fallbackClass = RestTemplateBlockHandler.class)//降级用fallback
@LoadBalanced//负载均衡的去调用
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

``注解中如果不配置 限流/降级 处理类就会走默认的``

``并且 如果触发了sentinel限流规则会走blockHandler参数配置的限流处理类``

``如果触发了sentinel降级规则会走fallback参数配置的降级处理类``

``并且不能同时使用，会启动失败``

![image-20210609183651197](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183651197.png)

**限流和降级处理类**

```java
import com.alibaba.cloud.sentinel.rest.SentinelClientHttpResponse;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;

/**
 * @author zhengtai.li
 * @ClassName RestTemplateBlockHandler
 * @Description restTemplate降级和降级处理类
 * @Copyright 2021 © kuwo.cn
 * @date 2021/5/29 15:42
 * @Version: 1.0
 */
public class RestTemplateBlockHandler {

    /**
     * 限流处理方法
     * resttemplate限流后的处理方法
     * 该方法的返回值一定是SentinelClientHttpResponse
     *
     * @return
     */
    public static SentinelClientHttpResponse blockA(HttpRequest request,
                                                       byte[] body,
                                                       ClientHttpRequestExecution execution,
                                                       BlockException ex) {

        System.err.println("fallback: " + ex.getClass().getCanonicalName());
        return new SentinelClientHttpResponse("限流了我的北鼻~~~~~~");
    }

    /**
     * 降级处理方法
     * @return
     */
    public static SentinelClientHttpResponse fallback(HttpRequest request,
                                                       byte[] body,
                                                       ClientHttpRequestExecution execution,
                                                       BlockException ex) {

        System.err.println("fallback: " + ex.getClass().getCanonicalName());
        return new SentinelClientHttpResponse("降级了我的北鼻~~~~~~");
    }
}
```

![image-20210609183714420](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183714420.png)

**测试接口**

![image-20210609183724789](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183724789.png)

**Sentinel配置**

![image-20210609183736778](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183736778.png)

只测试针对nacos-provider提供者的getData资源进行流控

![image-20210609183859636](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183859636.png)

**访问测试**

![image-20210609183813438](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183813438.png)



### 7.2 Sentinel整合Feign

**在application.yaml中加入如下配置（默认关闭）**

```yaml
#true开启sentinel对feign的支持，false则关闭（默认）
feign.sentinel.enabled=true
```



#### 7.2.1 方式一：fallback限流降级处理类

**Feign调用的Service**

name:该接口对应提供者注册到Nacos的服务命

fallback：该Service中的接口如果发生限流降级熔断的处理类

configuration：feign配置类，主要用户实例化fallback处理类

![image-20210609183938630](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183938630.png)

**fallback限流降级处理类**

该getData2方法就是针对于Feign调用Service下的哪个接口去专门做限流降级处理

![image-20210609183954846](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609183954846.png)

**Feign的fallback实例化配置类**

主要就是用于实例化fallback处理类的

![image-20210609184011197](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184011197.png)

**Feign调用测试接口**

![image-20210609184036578](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184036578.png)

**Sentinel配置**

![image-20210609184054471](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184054471.png)

![image-20210609184107288](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184107288.png)

**访问测试**

![image-20210609184119232](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184119232.png)

#### 7.2.2 方式二：fallbackFactory工厂

**Feign调用的Service**

name:该接口对应提供者注册到Nacos的服务命

fallbackFactory：该Service中的接口如果发生限流降级熔断的处理工厂，里面会包含该Service下所有接口的限流方法

configuration：feign配置类，主要用户实例化fallbackFactory

![image-20210609184138436](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184138436.png)

**fallbackFactory**

![image-20210609184153451](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184153451.png)

**Feign的fallbackFactory实例化配置类**

![image-20210609184208599](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184208599.png)

**Feign调用测试接口**

![image-20210609184220112](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184220112.png)

**Sentinel配置**

![image-20210609184233282](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184233282.png)

![image-20210609184245846](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184245846.png)

**访问测试**

![image-20210609184254579](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184254579.png)

## 八. Sentinel持久化

**默认：**默认情况下我们在dashboard配置的规则会通过微服务暴露sentinel接口，推送给微服务，然后微服务将规则缓存在内存中，dashboard是不提供存储的。

所以我们重启服务之后，规则就会丢失，所以我们需要进行持久化

**所有持久化操作都是在微服务端进行配置的，不是在sentinel dashboard配置的**

### 8.1 pull模式持久化到本地文件

![image-20210609184314903](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184314903.png)

**原理**

dashboard推送的规则信息会由微服务中的sentinel客户端保存到服务内存中，然后写入本地文件进行持久化。当服务重启后，内存中的规则会丢失，但是他会去本地文件中读取，然后重新加载到内存中。

**在项目中编写规则持久化类**

![image-20210609184327317](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184327317.png)

```java
import com.alibaba.csp.sentinel.command.handler.ModifyParamFlowRulesCommandHandler;
import com.alibaba.csp.sentinel.datasource.*;
import com.alibaba.csp.sentinel.init.InitFunc;
import com.alibaba.csp.sentinel.slots.block.authority.AuthorityRule;
import com.alibaba.csp.sentinel.slots.block.authority.AuthorityRuleManager;
import com.alibaba.csp.sentinel.slots.block.degrade.DegradeRule;
import com.alibaba.csp.sentinel.slots.block.degrade.DegradeRuleManager;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRule;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRuleManager;
import com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowRule;
import com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowRuleManager;
import com.alibaba.csp.sentinel.slots.system.SystemRule;
import com.alibaba.csp.sentinel.slots.system.SystemRuleManager;
import com.alibaba.csp.sentinel.transport.util.WritableDataSourceRegistry;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.TypeReference;

import java.io.File;
import java.io.IOException;
import java.util.List;

/**
 * 本地文件持久化方式的-规则持久化配置类
 */
public class FileDataSourceInit implements InitFunc {

    @Override
    public void init() throws Exception {
        //持久化文件保存路径，可自行定义，windows和linux肯定不一样
        String ruleDir = System.getProperty("user.home") + "/sentinel/rules";

        //各种规则文件
        String flowRulePath = ruleDir + "/flow-rule.json";
        String degradeRulePath = ruleDir + "/degrade-rule.json";
        String paramFlowRulePath = ruleDir + "/param-flow-rule.json";
        String systemRulePath = ruleDir + "/system-rule.json";
        String authorityRulePath = ruleDir + "/authority-rule.json";

        //创建保存持久化规则文件目录
        this.mkdirIfNotExits(ruleDir);

        //创建持久化规则文件路径
        this.createFileIfNotExits(flowRulePath);
        this.createFileIfNotExits(degradeRulePath);
        this.createFileIfNotExits(paramFlowRulePath);
        this.createFileIfNotExits(systemRulePath);
        this.createFileIfNotExits(authorityRulePath);

        // 流控规则：可读数据源
        ReadableDataSource<String, List<FlowRule>> flowRuleRDS = new FileRefreshableDataSource<>(
                flowRulePath,
                flowRuleListParser
        );
        // 将可读数据源注册至FlowRuleManager
        // 这样当规则文件发生变化时，就会更新规则到内存
        FlowRuleManager.register2Property(flowRuleRDS.getProperty());
        // 流控规则：可写数据源
        WritableDataSource<List<FlowRule>> flowRuleWDS = new FileWritableDataSource<>(
                flowRulePath,
                this::encodeJson
        );
        // 将可写数据源注册至transport模块的WritableDataSourceRegistry中
        // 这样收到控制台推送的规则时，Sentinel会先更新到内存，然后将规则写入到文件中
        WritableDataSourceRegistry.registerFlowDataSource(flowRuleWDS);


        // 降级规则：可读数据源
        ReadableDataSource<String, List<DegradeRule>> degradeRuleRDS = new FileRefreshableDataSource<>(
                degradeRulePath,
                degradeRuleListParser
        );
        DegradeRuleManager.register2Property(degradeRuleRDS.getProperty());
        // 降级规则：可写数据源
        WritableDataSource<List<DegradeRule>> degradeRuleWDS = new FileWritableDataSource<>(
                degradeRulePath,
                this::encodeJson
        );
        WritableDataSourceRegistry.registerDegradeDataSource(degradeRuleWDS);


        // 热点参数规则：可读数据源
        ReadableDataSource<String, List<ParamFlowRule>> paramFlowRuleRDS = new FileRefreshableDataSource<>(
                paramFlowRulePath,
                paramFlowRuleListParser
        );
        ParamFlowRuleManager.register2Property(paramFlowRuleRDS.getProperty());
        // 热点参数规则：可写数据源
        WritableDataSource<List<ParamFlowRule>> paramFlowRuleWDS = new FileWritableDataSource<>(
                paramFlowRulePath,
                this::encodeJson
        );
        ModifyParamFlowRulesCommandHandler.setWritableDataSource(paramFlowRuleWDS);


        // 系统规则：可读数据源
        ReadableDataSource<String, List<SystemRule>> systemRuleRDS = new FileRefreshableDataSource<>(
                systemRulePath,
                systemRuleListParser
        );
        SystemRuleManager.register2Property(systemRuleRDS.getProperty());
        // 系统规则：可写数据源
        WritableDataSource<List<SystemRule>> systemRuleWDS = new FileWritableDataSource<>(
                systemRulePath,
                this::encodeJson
        );
        WritableDataSourceRegistry.registerSystemDataSource(systemRuleWDS);


        // 授权规则：可读数据源
        ReadableDataSource<String, List<AuthorityRule>> authorityRuleRDS = new FileRefreshableDataSource<>(
                authorityRulePath,
                authorityRuleListParser
        );
        AuthorityRuleManager.register2Property(authorityRuleRDS.getProperty());
        // 授权规则：可写数据源
        WritableDataSource<List<AuthorityRule>> authorityRuleWDS = new FileWritableDataSource<>(
                authorityRulePath,
                this::encodeJson
        );
        WritableDataSourceRegistry.registerAuthorityDataSource(authorityRuleWDS);
    }


    private Converter<String, List<FlowRule>> flowRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<FlowRule>>() {
            }
    );

    private Converter<String, List<DegradeRule>> degradeRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<DegradeRule>>() {
            }
    );

    private Converter<String, List<SystemRule>> systemRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<SystemRule>>() {
            }
    );

    private Converter<String, List<AuthorityRule>> authorityRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<AuthorityRule>>() {
            }
    );

    private Converter<String, List<ParamFlowRule>> paramFlowRuleListParser = source -> JSON.parseObject(
            source,
            new TypeReference<List<ParamFlowRule>>() {
            }
    );

    private void mkdirIfNotExits(String filePath) throws IOException {
        File file = new File(filePath);
        if (!file.exists()) {
            file.mkdirs();
        }
    }

    private void createFileIfNotExits(String filePath) throws IOException {
        File file = new File(filePath);
        if (!file.exists()) {
            file.createNewFile();
        }
    }

    private <T> String encodeJson(T t) {
        return JSON.toJSONString(t);
    }
}
```



**配置SPI**

``resources\META-INF\services\com.alibaba.csp.sentinel.init.InitFunc``

文件名称是固定的，内容就是持久化配置类的路径

![image-20210609184352646](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184352646.png)

**测试**

**第一步：重启服务**

**第二步：创建sentinel限流规则**

![image-20210609184409038](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184409038.png)

此时该规则会被dashboard推送给服务，由服务缓存到内存中，并写入到本地的持久化文件

**第三步：查看规则持久化文件**

我这是在windows电脑上测试的，linux其实也一样去找目录就可以了

如果你是用docker部署项目，一定要将容器中的持久化目录挂载到外面，否则重启服务会生成新的容器，那么规则还是会丢失

![image-20210609184425479](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184425479.png)



### 8.2 push模式持久化到Nacos（生产推荐）

不仅可以推送到Nacos也可以推送到``Zookeeper、apollo、redis、consul``

**添加依赖**

```yaml
<!--sentinel数据持久化Nacos数据源-->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

**application.yaml配置持久化数据源**

```yaml
spring:
  cloud:
    sentinel:
        datasource:
          ds1:
            nacos:
              server-addr: ${spring.cloud.nacos.discovery.server-addr} #nacos地址
              dataId: ${spring.application.name}-rule #该服务sentinel规则配置的dataId名称
              groupId: sentinel-rule #所在组
              data-type: json #类型  一定是json
              rule-type: flow #限流 这个不配应该也可以
              namespace: 1a6b7567-f008-43dc-8c61-f7b805611eae #所属命名空间
```

**在Nacos中配置Sentinel规则**

项目配置sentinel数据源为Nacos后，在Sentinel创建规则并不会发送到Nacos中存储，需要我们手动在Naocs配置，然后他会定时去Nacos中同步配置到内存中。

![image-20210609184510201](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184510201.png)

![image-20210609184519898](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184519898.png)

我们可以通过Sentinel的审查元素查看创建的规则请求体中的 规则json，然后复制过来就可以了

**测试**

重启服务，查看是否存在该配置

![image-20210609184532928](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184532928.png)

只不过很麻烦，我们只能通过在Nacos去配置Sentinel规则，因为Sentinel配置的规则不会同步到Nacos中

实际中我们一般都会在Sentinel去创建规则，然后打开审查元素，查看sentinel发送的请求体，然后复制请求体中的规则json，粘贴到nacos中

![image-20210609184547799](https://cdn.jsdelivr.net/gh/EayonLee/IMG-Cloud@master/data/image-20210609184547799.png)

如果想Sentinel创建修改规则之后直接推送到Nacos需要修改sentinel源码，可以自行百度