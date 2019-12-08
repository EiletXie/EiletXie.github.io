---
title: Ribbon在SpringCloud是怎么使用的？
date: 2019-08-20 11:25:39
tags:
- 分布式
- Ribbon
categories:
- JAVA
toc: true
---



# Ribbon

Ribbon是一款SpringCloud处理客户端http与tcp访问的负载均衡策略的工具



## Ribbon如何与springcloud组件结合？

在Spring中使用 RestTemplate 获取 http请求时，为了简约

我们常会 @LoadBalanced 注解 标记 RestTemplate

让SpringCloud为 RestTemplate 设置负载均衡策略。

那么它的如何设置的呢？

<!--more-->

该注解是这样解释的

*Annotation to mark a RestTemplate bean to be configured to use a LoadBalancerClient*



标记一个RestTemplate的bean为被 LoadBalancerClient所配置

我们查看这个 LoadBalancerClient 类

简单的截个图

{% asset_img b1.jpg loadBalance %}

这里的接口解释为： 表示客户端负载均衡器，其中它继承了 ServiceInstanceChooser接口

{% asset_img b2.jpg ServiceInstanceChooser %}

从 ServiceInstanceChooser的这个接口名称和类解释上我们不难看出，这个接口的设计就是为了从负载均衡器中选择一个 具体的服务实例

### 接口结构图：

{% asset_img b3.jpg loadBalance-interface%}


### 3个核心方法：

1. 父类 ServiceInstanceChooser的 choose方法：选择一个具体的服务实例
2. LoadBalancerClient 中 execute，这里传参不同有两个方法，但实际上功能是一致的，都是使用LoadBalancer中的ServiceInstance执行指定的请求。
3. URI reconstructURI(ServiceInstance instance, URI original) ：该方法创建一个真实的主机地址和端口，因为我们访问时给定的地址是用服务名访问的 http://myservice/path/to/service 

​      它会将我们给定的服务名进行转成成一个可用的http请求url

这些都是接口方法，那么具体



## 它是怎么实现的，是如何做到自动装配负载均衡的呢？

我们查看 负载均衡包 package org.springframework.cloud.client.loadbalancer 下的相关类

{% asset_img b4.jpg loadBalance-package%}

查看一下它的配置类 LoadBalancerAutoConfiguration.class

{% asset_img b5.jpg loadBalance-configure%}

看到没有，配置类上的3个注解，它将 RestTemplate,和 LoadBalancerClient以及 负载均衡的相关配置信息注入进来处理。

参数中 先注入了所有RestTemplate，然后在下面的初始化中 采用单例模式在遍历的过程中返回具体的实例，这里可以看出默认是循环遍历，也就是轮播的机制。


## 那当我们在application，或者其他地方自定义负载均衡策略时，它又是怎么处理的呢？

在LoadBalancerAutoConfiguration类中有一段这样的代码

{% asset_img b6.jpg loadBalance-factory %}

它初始获取了Spring容器中的负载均衡请求转换列表，有一个类 LoadBalancerRequestTransformer ，这个类是用来干嘛的呢


### LoadBalancerRequestTransformer 

{% asset_img b7.jpg loadBalance-transform %}

类解释为：允许应用程序转换给定的负载均衡{@link HttpRequest} 所选{@link ServiceInstance}

这下懂了把，它会获取所有配置中用户给定的负载均衡策略，再根据 给定的策略定制 负载均衡请求工厂。



但是其实我还是很迷糊的，我需要知道


## Ribbon对于RestTemplate的请求的拦截处理是咋样的？

发现 LoadBalancerAutoConfiguration类中 包含了一个 静态的内部类 

它是一个负载均衡的拦截器配置类LoadBalancerInterceptorConfig


###  LoadBalancerInterceptorConfig

{% asset_img b8.jpg interceptConfig %}

包含了两个方法，这两个方法解决了我的疑问。

第一个方法，返回了一个负载均衡的拦截器，我们发现它传入了客户端和 LoadBalancerRequestFactory

```java
LoadBalancerInterceptor ribbonInterceptor(
LoadBalancerClient loadBalancerClient,
LoadBalancerRequestFactory requestFactory)
```

在前面 LoadBalancerRequestFactory 是在装配的时候就已经填入了 用户自定义的负载均衡策略了的！

return new LoadBalancerRequestFactory(loadBalancerClient, transformers)；

同时设计者将这些参数组合成了一个 LoadBalancerInterceptor类。

看一下这个负载均衡拦截类的设计，

{% asset_img b9.jpg loadBalance-intercept %}

看到有两个构造器了吗？

第二个构造器应该是当检测到用户如果没有指定负载均衡策略的时候跑的。

第一个构造器带 factory的是用户定制负载均衡策略跑的。

{% asset_img funny.gif funny %}

It's funny！！

第二个方法：

```java
public RestTemplateCustomizer restTemplateCustomizer(

​      final RetryLoadBalancerInterceptor loadBalancerInterceptor)；
```

这一步是将上一步的拦截了http请求塞入了定制的负载均衡策略的拦截器作为规则重新修改

RestTemplate，将其返回。



其实还有一个RetryInterceptorAutoConfiguration类，其做法负载均衡拦截器类的基本一致，并且未注入类资源使用。我不能查看RetryTemplate源码就google了一下。



### RetryTemplate.class:

可重试操作封装在RetryCallback接口的实现中，并使用提供的execute方法之一执行。

默认情况下，如果抛出异常的任何异常或子类，则重试操作。可以使用setRetryPolicy（RetryPolicy）方法更改此行为。

此类是线程安全的，适用于执行操作和执行配置更改时的并发访问。因此，可以动态更改重试次数，以及使用的BackOffPolicy，并且不会影响正在进行的可重试操作。



这是一个用于并发访问时做可重试操作的东东，应该是拦截http请求时报错后使用这个东西，这里就不深入研究了。

{% asset_img b10.jpg loadBalance-retry %}

这个时候梳理一下Ribbon在Springcloud中的大体工作流程。

{% asset_img b11.jpg work-img %}



## Ribbon整个流程处理的类图中类之间的关联情况是啥样的？

我将比较核心的类的关联关系理了一下。

{% asset_img b12.jpg class-link %}

这里梳理了一下工作流和类的关系，个人暂时就探索到这里了，项目中遇到什么对Ribbon的疑问再到这篇文章下面补充。

> 注：由于这里是自己暂时摸索了一下源码，有错误的地方麻烦各位指正一下。后续会看书进行再次疏通