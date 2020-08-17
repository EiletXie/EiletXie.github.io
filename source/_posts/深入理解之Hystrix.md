---
title: 深入理解之Hystrix
date: 2020-03-09 21:06:59
tags:
- hystrix
categories:
- springcloud
toc: true

---



Hystrix是 SpringCloud中的一个组件，它能起到了断路器的作用。

## Hystrix的基本介绍

### 它解决了什么问题？

​       当服务之间调用可能因为某个服务的堵塞或服务器问题使得其他需要消费该服务的其他服务请求处于等待，由于服务之间是相互调用的，会导致大量请求进程挂起进而导致整个系统CPU飙升进而崩溃的局面。

​       而断路器是作用是做到 当一个服务单元发生故障，通过故障监控，向调用方返回一个错误响应，而不是长时间等待，使得线程不会因故障长时间不释放，避免故障在服务间蔓延。



<br/>
<!--more-->

### hystrix基本配置

hystrix默认超时时间 2000毫秒





原生组件 Hystrix

```xml
<dependency>
  <groupId>com.netflix.hystrix</groupId>
  <artifactId>hystrix-core</artifactId>
  <version>1.5.8</version>
</dependency>
```



当然 SpringCloud 也集成了Hystrix

```xml
<dependency>     <groupId>org.springframework.cloud</groupId>     <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency> 
```



<br/>

在服务消费者的主类OrderApplication上使用 @EnableCircuitBreaker注解开启断路器功能

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
@ComponentScan(basePackages = "com.eiletxie")
public class OrderApplication {

   public static void main(String[] args) {
      SpringApplication.run(OrderApplication.class, args);
   }
}
```



其实可以直接使用 @SpringCloudApplication 注解来修饰主类，因为它包含上面的这3个注解

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public @interface SpringCloudApplication {
}
```



<br/>

### 设置函数降级策略

**@HystrixCommand(fallbackMethod = "降级函数名")**

我们可以在 controller层或者service层对方法增加 @HystrixCommand 注解来指定回调方法，也就是常说的降级策略

```java
   @HystrixCommand(fallbackMethod = "fallback")
   @GetMapping("/getProductInfoList")
   public String getProductInfoList(@RequestParam("number") Integer number) {

      RestTemplate restTemplate = new RestTemplate();
      return restTemplate.postForObject("http://127.0.0.1:8005/product/listForOrder",
            Arrays.asList("157875196366160022"),
            String.class);
   }

   private String fallback() {
      return "太拥挤了, 请稍后再试~~";
   }
```



### 服务熔断参数设置

当然你也可以对函数调用的具体相关参数做熔断的条件判断

Hystrix默认超时时间为2000毫秒

```java
@HystrixCommand(commandProperties = {
      @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),              //设置熔断
      @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),    //请求数达到后才计算
      @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000"), //休眠时间窗
      @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60"),  //错误率
})
```

<br/>

接下来就是重头戏了

我们通过官方给的流程图来了解 请求调用其他服务时 Hystrix如何工作的。

## Hystrix工作流程

{% asset_img  hystrix_flow.jpg  Hystrix流程图 %}

我们根据流程图来分析每一个节点

### 1、 创建 HystrixCommand 或 HystrixObservableCommand 对象

这两个对象是用来干嘛的呢？ 

​       用来 封装请求（对其他服务调用的请求），它包含了所有需要传递的参数。这两个Command对象分别针对不同的应用场景，都使用到了设计模式中的命令模式。

> - HystrixCommand ： 用从依赖的服务返回单个操作结果的时候。
> - HystrixObservableCommand: 用在依赖的服务返回多个操作结果的时候。

<br/>

observable可观察的

这里可以先看菜鸟教程的命令模式简介 https://www.runoob.com/design-pattern/command-pattern.html

#### 命令模式

命令模式（Command Pattern）是一种数据驱动的设计模式，它属于行为型模式。

将一个请求封装为一个对象，从而使你可用不同的请求对客户端进行参数化，对请求排队或记录请求日志，以及支持可撤销的操作。

简单来说： 命令模式将来自客户端的请求封装成一个对象，从而让你使用不同的请求对客户端进行参数化，它的最大好处是 行为请求者 与 行为实现者 之间 的解耦。

**个人写了一篇对于命令模式的描述,感兴趣的朋友可以看看 ->**   [**命令模式**](http://www.eiletxie.cn/2019/12/25/命令模式/)

<br/>

**为什么这里用到了命令模式呢？**

个人理解：  我们思考一下，在controller或service层中，服务消费者调用了 服务提供者的相关接口，同时我们使用了hystrix，也就是类似 @HystrixCommand(fallbackMethod = "fallback") 这种注解来对请求做了一定的限定，要求 这个请求（请求服务提供方的API）的调用过程要符合在我们给定的参数范围内，例如： 请求时间不能超过3秒，失败后重复请求次数不能超过5次、以及失败后执行哪个回调函数。 我们通过命令模式这种设计思想，将每个服务调用的请求封装成命令。当服务消费端 执行该命令，command命令底层再调用相关的服务提供方，让服务之间实现了解耦。

​         并且因为将请求封装成命令，那么它可以做到记录日志（将命令执行次数与我们设定的参数做对比）、撤销操作，以及高并发时用队列存放起来排队执行。

<br/>

### 2、命令执行

从上图中，我们可以看到一共存在 4种命令的执行方式，Hystrix在执行时会根据创建的Command对象和具体情况来选择一个执行。

- HystrixCommand实现的两种执行方式

> execute()： 同步执行，从依赖的服务返回一个单一的结果对象，或是在发生错误的时候抛出异常。
> queue(): 异步执行，直接返回一个Future对象，其中包含了服务执行结束时要返回的单一结果对象。
> R value = command.execute();
> Future fValue = command.queue();

- HystrixObservableCommand的两种执行方式

> observe(): 返回 Observable 对象，它代表操作的多个结果，它是一个 Hot Observale
> toObserve(): 同样返回Observable 对象，它也代表操作的多个结果，但是它返回的是一个 Cold Observable

值得一提的是，Hystrix的底层实现大量使用了 RXJava，为了后续的讲解，这里大概讲一下 RXJava主要用到的 观察者-订阅者模式

 观察者-订阅者模式： 一看到观察-订阅，就让我们想到了 消息中间件的Topic订阅。

<br/>

#### 观察者模式 

在软件设计中是一个对象，维护一个依赖列表，当任何状态发生改变自动通知它们。

例如： B站中你关注了几个up主，当关注的up主更新时，会第一时间通知所有关注了他的人。

这里的 up主是 Subject，而作为观众的你是 Observe

**思考： 这个状态变化的监测和通知的过程是怎么实现的？**

​      其实不难，在一些教程中，它们的实现思路就是 订阅的主题 subject中有一个 List<Observer>列表来存放所有的观察者Observer的实现类，而Observer实现了一个统一的接口，该接口规定要实现一个 update方法。

​        所以当subject主题状态更新，只需要遍历观察者列表，执行update接口方法来调用具体的实现类的状态通知。

<br/>

#### 发布-订阅设计模式

与观察者模式基本差不多，区别在哪呢？ 中间多了一层中间消息件一样的东西

例如： 你订阅了腾讯新闻的财经频道，然后你可以收到很多作者写的新闻，但是作者是将新闻发布到财经频道，你是在频道中看到新闻，你和作者之间没有关联。

在发布-订阅模式，消息的发送方，叫做**发布者（publishers）**，消息不会直接发送给特定的接收者，叫做**订阅者**。

{% asset_img  pub_sub.jpg  发布-订阅 %}

<br/>

**总结一下两模式的区别**

- **观察者模式**大多数时候是**同步**的，比如当事件触发，Subject就会去调用观察者的方法。而**发布-订阅**模式大多数时候是**异步的**（使用消息队列）。
- 在**观察者**模式中，观察者是知道Subject的，Subject一直保持对观察者进行记录。然而，在**发布订阅**模式中，发布者和订阅者**不知道对方的存在**。它们只有通过消息代理进行通信。
- 在**发布订阅**模式中，组件是松散耦合的，正好和观察者模式相反



代入hystrix来看，使用了hystrix的服务消费方开启了一个观察监控，监控请求对应的服务提供方的响应，如果正常响应则直接获取响应数据来处理消费方的业务逻辑。如果失败就根据对应用户设置的请求参数判断是否开启断路器。



**这时候来看看 Hystrix中 Hot** **Observable  和 Cold Observable  这两个概念**

- Hot Observable 是无论 不管“事件源” 是否有 订阅者，都会创建后对事件进行发布，所以它应该对应的是 发布订阅模式,它只能看到部分过程。
-  Cold Observable  则是在 没有“订阅者”的时候并不会发布事件，而是进行等待，直到有“订阅者”之后才发布事件，这样它可以查看到整个过程，它这种同步的方式对应的是 观察者模式。

<br/>

### 3、结果是否被缓存

 若当前命令的请求缓存功能是否被开启的，并且该命令缓存命中，那么缓存的结果会立即以Observable 对象的形式返回。

这一步是比较好理解的，由于可能发生网络中断或者用户重复点击等请求数据重复获取的情况，将命令对象存放在 线程池中，先检验是否缓存命中，再发起网络请求，效率会有一定提升，但是由于请求的参数和时间差异,应用面较小。





### 4、断路器是否打开

   在命令结果没有缓存命中的时候，Hystrix在执行命令前需要检查断路器是否为打开状态：

- 如果断路器是打开的，那么Hystrix不会执行命令，而是转接到fallback处理逻辑（防止服务之间请求堆积）
- 如果断路器是关闭的，那么Hystrix会跳到第5步，检查是否有可用资源来执行命令。

<br/>

### 5、线程池/请求队列/信号量是否占满

​       当与命令相关的线程池和请求队列，或者信号量（不使用线程池的时候）已经被占满，那么 Hystrix 也不会执行命令，而是转接到 fallback处理逻辑。

值得一提的是，这里的线程池不是我们认为的一个大的服务消费者的线程池，而是针对每个请求对应的依赖服务创建的专有线程池。 并且 Hystrix为了保证不会因为某个依赖服务的问题影响其他依赖服务而采用了 **“舱壁模式”** 来隔离每个依赖的服务。

<br/>

### 6、HystrixObservableCommand.construct() 或 HystrixCommand.run() 

Hystrix 会根据我们编写的方法来决定采用什么的方式去请求依赖服务。

- HystrixCommand.run()：返回单一的结果，或者抛出异常。
- HystrixObservableCommand.construct() ： 返回一个Observable对象来发射多个结果，或通过 onError发送错误通知。



​     如果run()或 construct() 方法执行时间超过了 命令设置的超时阈值，当前处理的线程将抛出一个 TimeoutException。在这种情况下，Hystrix会转接到fallback处理逻辑。

​    如果命令正常执行返回了结果，那么Hystrix会记录一些相关的日志并采集监控报告之后将该结果返回。这就是我们看到了那个 Hystrix-DashBoard，也称为仪表盘，豪猪监控界面了 。

<br/>

### 7、计算断路器的健康度

​     Hystrix会将“成功”、“失败”、“拒绝”、“超时”等信息报告给断路器，而断路器会维护一组计数器来统计这些数据。

​     断路器会使用这些统计数据来决定是否将断路器打开，来对某个依赖服务的请求进行 ”熔断/短路“，直到恢复期结束。若在恢复期结束后，根据统计数据判断如果还未到健康指标，就会再次”熔断/短路“

<br/>

### 8、fallback处理

   当命令执行失败时，Hystrix会进入 fallback尝试回调处理，该操作被称为 “服务降级”，引起服务降级处理的情况有下面几种

- 第4步： 当前命令处于 “熔断/短路”状态，并且断路器是打开的状态。
- 第5步： 当前命令的线程池、请求队列或者信号量被占满的时候。
- 第6步：HystrixObservableCommand.construct() 或 HystrixCommand.run() 抛出 异常的时候



 需要说明的是：在服务降级的逻辑中，我们需要一个通用的响应结果，并且这个结果的处理逻辑应该是从缓存或一些静态逻辑中获取，大部分情况指的就是服务自身的方法，而不是通过网络请求来获取，如果是通过网络请求，那么该请求也必须被 HystrixCommand或HystrixObservableCommand申明，形成一个级联的 降级策略，但最终的降级逻辑一定不是通过网络请求的处理，而是一个能稳定返回结果的处理逻辑。



​    当我们没有为命令实现降级逻辑或者降级处理逻辑中抛出了异常，Hystrix依然会返回一个 Observable 对象，但是它不会产生任何结果，而是通过onError方法通知命令立即中断请求，并通过onError方法将引发命令失败的异常发送给调用者。



​    当降级执行发现失败的时候，Hystrix会根据不同的执行方法做出不同的处理（注意： 是**降级处理的过程中发生异常**）。

- execute(): 抛出异常
- queue(): 正常返回Future对象，但是当调用 get() 来获取结果的时候会抛出异常。
- observe(): 正常返回 Observable对象，当订阅它的时候，将立即通过调用 onError方法来通知终止请求。
- toObservable(): 正常返回 Observable对象，当订阅它的时候，将立即通过调用 onError方法来通知终止请求。

<br/>

### 9、返回成功的响应

​    当Hystrix命令执行成功之后，它会将处理结果直接返回或是以Observable的形式返回。而具体以哪种方式返回取决于之前第2步中我们所提到的对命令的4种不同执行方式，下图总结了这4种调用方式之间的依赖关系。

{% asset_img success_return.jpg 成功的响应 %}

- toObservable(): 返回最原始的 Observable() ,必须通过订阅它才会真正触发命令的执行流程。
- observe(): 在 toObservable() 产生原始 Observable 之后立即订阅它，让命令能够马上开始异步执行，并返回一个Observable对象，当调用它的subscribe时，将重新产生结果和通知给订阅者。
- queue(): 将toObservable() 产生的原始Observable通过 toBlocking()方法转换成 BlockingObservable对象，并调用它的toFuture()方法返回异步的Future对象
- execute(): 在queue()产生异步结果 Future对象之后，通过调用 get() 方法阻塞并等待结果的返回。

也就是说 返回的是一个 Observable形式的对象，然后根据我们最初命令执行的方法（queue、execut、observe）来进行返回格式的转换。



说完了Hystrix的工作流程，是时候聊聊能断水断电，牛逼的断路器了。

<br/>

## 断路器原理

断路器那么厉害，它是如何决策熔断和记录信息的呢？

### HystrixCircuitBreaker 接口分析

我们查看一下 HystrixCircuitBreaker 接口的代码 （这是 Springcloud 2.0.6.RELEASE版，与其他版本差不多）

```java
public interface HystrixCircuitBreaker {
    boolean allowRequest();

    boolean isOpen();

    void markSuccess();

    void markNonSuccess();

    boolean attemptExecution();
    
    public static class NoOpCircuitBreaker implements HystrixCircuitBreaker{...};
    
    public static class HystrixCircuitBreakerImpl implements HystrixCircuitBreaker {...};
    
    public static class Factory {...};
}
```

可以看出接口定义较为简单，比较核心的三个抽象方法

- allowRequest(): 每个Hystrix命令的请求都通过它判断是否被执行
- isOpen(): 返回当前断路器是否打开
- markSuccess(): 代表请求执行成功，用来关闭断路器



另外还有三个静态类

**Factory**：

​       维护了一个Hystrix命令与HystrixCircuitBreaker的关系集合：ConcurrentHashMap<String, HystrixCircuitBreaker> circuitBreakersByCommand，其中String类型的key通过 HystrixCommandKey定义。每一个Hystrix命令需要有一个key来标识，同时一个Hystrix命令也会在该集合中找到它对应的断路器 HystrixCircuitBreaker实例。



**NoOpCircuitBreaker** **：**

   定义了一个什么都不做的断路器实现，它允许所有请求，并且断路器状态始终闭合。



**HystrixCircuitBreakerImpl** **：**

   断路器接口 HystrixCircuitBreaker 的实现类，在该类中定义了断路器的几个核心对象

- HystrixCommandProperties properties: 断路器对应 HystrixCommand实例的属性对象
- HystrixCommandMetrics metrics： 用来让 HystrixCommand记录各类度量指标的对象
-  AtomicLong circuitOpened: 断路器是否打开的标志
- spring cloud微服务实战书中说 1.3x版本有 AtomicLong circuitOpenedorLastTestedTime: 断路器打开或是上一次测试的时间

而2.0版本修改成了下面两个属性

- AtomicReference<Status> status： 断路器状态标识
- AtomicReference<Subscription> activeSubscription ： Subscription中包含了断路器开启的时间，以及对断路器的状态记录





下面我们来分析**HystrixCircuitBreakerImpl** 来查看下面3个核心方法的实现

- isOpen(): 判断断路器的打开/关闭状态，详细逻辑如下：
- allowRequest(): 判断请求是否被允许

可以看到断路器直接通过this.properties判断对应强制打开或关闭属性是否被设置

```java
 public boolean isOpen() {
    if ((Boolean)this.properties.circuitBreakerForceOpen().get()) {
        return true;
    } else if ((Boolean)this.properties.circuitBreakerForceClosed().get()) {
        return false;
    } else {
        return this.circuitOpened.get() >= 0L;
    }
}

public boolean allowRequest() {
    if ((Boolean)this.properties.circuitBreakerForceOpen().get()) {
        return false;
    } else if ((Boolean)this.properties.circuitBreakerForceClosed().get()) {
        return true;
    } else if (this.circuitOpened.get() == -1L) {
        return true;
    } else {
        return ((HystrixCircuitBreaker.HystrixCircuitBreakerImpl.Status)this.status.get()).equals(HystrixCircuitBreaker.HystrixCircuitBreakerImpl.Status.HALF_OPEN) ? false : this.isAfterSleepWindow();
    }
}
```

- markSuccess():从if判断可以得知，markSuccess在 断路器“半开路”或“关闭”状态下判断

​              当Hystrix请求重新调用成功时，它取消了上次请求的数据记录，并重置当前的度量指标对象和关闭断路器,这里我们可以推断出 markSuccess方法应该是 请求失败后再次重连成功后的断路器状态处理

```java
public void markSuccess() {
    if (this.status.compareAndSet(HystrixCircuitBreaker.HystrixCircuitBreakerImpl.Status.HALF_OPEN, HystrixCircuitBreaker.HystrixCircuitBreakerImpl.Status.CLOSED)) {
        this.metrics.resetStream();
        Subscription previousSubscription = (Subscription)this.activeSubscription.get();
        if (previousSubscription != null) {
            previousSubscription.unsubscribe();
        }

        Subscription newSubscription = this.subscribeToStream();
        this.activeSubscription.set(newSubscription);
        this.circuitOpened.set(-1L);
    }
}
```

再看一下 Netfix Hystrix官方文档中对于断路器的详细执行逻辑图

### 断路器的详细执行逻辑

{% asset_img circuit_flow.jpg 断路器流程 %}

参考博文： https://segmentfault.com/a/1190000012439580

断路器打开和关闭有如下几种情况：

- 假设断路中的请求满足了一定的阈值（HystrixCommandProperties.circuitBreakerRequestVolumeThreshold()）
- 假设错误发生的百分比超过了设定的错误发生的阈值HystrixCommandProperties.circuitBreakerErrorThresholdPercentage()
- 断路器状态由CLOSE变换成OPEN
- 如果断路器打开，所有的请求都会被断路器所熔断。
- 一定时间之后HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds()，下一个的请求会被通过（处于半打开状态），如果该请求执行失败，断路器会在睡眠窗口期间返回OPEN，如果请求成功，断路器会被置为关闭状态，重新开启1步骤的逻辑。

<br/>



## 舱壁模式

​     如果了解过 Docker的小伙伴，肯定听过“舱壁模式”，它实现了进程的隔离，使得容器之间不会互相收到影响。Hystrix采用舱壁模式来隔离相互之间的依赖关系，它会为每个依赖服务创建一个独立的线程池，这样就算某个依赖服务出现延迟高的情况，也不会影响到其他的依赖服务。

{% asset_img container_divorse.jpg  舱壁模式 %}

### 线程和线程池

客户端（第三方包、网络调用等）会在单独的线程执行，会与调用的该任务的线程进行隔离，以此来防止调用者调用依赖所消耗的时间过长而阻塞调用者的线程。

{% asset_img thread_pool.jpg  线程池调用服务 %}

您可以在不使用线程池的情况下防止出现故障，但是这要求客户端必须能够做到快速失败（网络连接/读取超时和重试配置），并始终保持良好的执行状态。

Netflix设计Hystrix，并且选择使用线程和线程池来实现隔离机制，有以下几个原因：

- 很多应用会调用多个不同的后端服务作为依赖。
- 每个服务会提供自己的客户端库包。
- 每个客户端的库包都会不断的处于变更状态。
- 每个客户端库包都可能包含重试、数据解析、缓存等等其他逻辑。
- 对用户来说，客户端库往往是“黑盒”的，对于实现细节、网络访问模式。默认配置等都是不透明的。
- 大部分的网络访问是同步执行的。

- 客户端代码中也可能出现失败和延迟，而不仅仅是在网络调用中。

<br/>

- **使用线程池的好处**

通过线程在自己的线程池中隔离的好处是：

- - 该应用程序完全可以不受失控的客户端库的威胁。即使某一个依赖的线程池已满也不会影响其他依赖的调用。
  - 应用程序可以低风险的接受新的客户端库的数据，如果发生问题，它会与出问题的客户端库所隔离，不会影响其他依赖的任何内容。
  - 当失败的客户端服务恢复时，线程池将会被清除，应用程序也会恢复，而不至于使得我们整个Tomcat容器出现故障。
  - 如果一个客户端库的配置错误，线程池可以很快的感知这一错误（通过增加错误比例，延迟，超时，拒绝等），并可以在不影响应用程序的功能情况下来处理这些问题（可以通过动态配置来进行实时的改变）。
  - 如果一个客户端服务的性能变差，可以通过改变线程池的指标（错误、延迟、超时、拒绝）来进行属性的调整，并且这些调整可以不影响其他的客户端请求。
  - 除了隔离的优势之外，拥有专用的线程池可以提供内置的请求任务的并发性，可以在同步客户端上构建异步门面。

简而言之，由线程池提供的隔离功能可以使客户端库和子系统性能特性的不断变化和动态组合得到优雅的处理，而不会造成中断，保证了应用的健壮性



当然线程池隔离的方案好处多多，但是也多了额外的开销和延迟，但Netfix官方对于线程做了相关测试，以结果打消我们的顾忌。

<br/>

- **线程池的缺点**

线程池最主要的缺点就是增加了CPU的计算开销，每个命令都会在单独的线程池上执行，这样的执行方式会涉及到命令的排队、调度和上下文切换。

Netflix在设计这个系统时，决定接受这个开销的代价，来换取它所提供的好处，并且认为这个开销是足够小的，不会有重大的成本或者是性能影响。

<br/>

- **线程成本**

Hystrix在子线程执行construct()方法和run()方法时会计算延迟，以及计算父线程从端到端的执行总时间。所以，你可以看到Hystrix开销成本包括（线程、度量，日志，断路器等）。

### 使用线程池的性能测试

下图显示了一个HystrixCommand在单个API实例上每秒执行60个请求（每个服务器每秒执行大约350个线程执行总数）：

{% asset_img compare.jpg  性能比较测试 %}

注意： 这个图在怎么看呢？ 主要是看下方对于每条线的颜色注解，它统计了服务器处理的3个峰值，每种颜色的线有两根，分别对应使用线程池 User开头，与未使用线程池做了对比

我们可以看到只有当服务提供方的处理达到顶点时，两种使用方式之间才有9ms的延迟差，几乎可以忽略不计。

| 服务器峰值 | 未使用线程池 | 使用线程池 | 延迟差 |
| ---------- | ------------ | ---------- | ------ |
| 中位数     | 2ms          | 2ms        | 0ms    |
| 90%        | 5ms          | 8ms        | 3ms    |
| 99%        | 28ms         | 37ms       | 9ms    |

<br/>

## 请求合并

​      微服务架构中的依赖通常是通过远程调用实现的，而远程调用中最常见的问题就是通信消耗与连接数占用。因为通信次数的增加，总的通信时间消耗将会不太理想。同时hystrix创建的依赖服务的线程池资源有限，将会出现排队等待与响应延迟的情况。为了优化这两个问题，Hystrix推出了 **HystrixCollapser** 来实现 请求的合并，以减少消耗和线程数的占用。

​      HstrixCollapser 实现了 在 HystixCommand 之前放置一个合并处理器，将处于一个很短的时间窗（默认10毫秒）内的针对同一个依赖服务的多个请求进行整合并批量方式发起请求的功能（当然服务提供方也需要有相应的批量实现接口）。



简单点说： 假设消费方（单用户与多用户）短时间内对服务提供方UserService 进行了大量的请求调用，那么消费方会将这些 HystrixCommand打包在一块，一次发送，一次获取数据，而不是每次发送一次请求获取一次数据，大大节约了通信开销。

如下图：

{% asset_img hystrix_collasper.jpg  请求合并 %}

<br/>

### 为什么使用请求合并

事情请求合并来减少执行并发HystrixCommand请求所需要的线程数和网络连接数。请求合并以自动方式执行的，不需要代码层面上进行批处理请求的编码。

- 全局上下文（所有的tomcat线程）

理想的合并方式是在全局应用程序级别来完成的，以便来自任何用户的任何Tomcat线程的请求都可以一起合并。

例如，如果将HystrixCommand配置为支持任何用户请求获取影片评级的依赖项的批处理，那么当同一个JVM中的任何用户线程发出这样的请求时，Hystrix会将该请求与其他请求一起合并添加到同一个JVM中的网络调用。

请注意，合并器会将一个HystrixRequestContext对象传递给合并的网络调用，为了使其成为一个有效选项，下游系统必须处理这种情况。

- 用户请求上下文（单个tomcat线程）

如果将HystrixCommand配置为仅处理单个用户的批处理请求，则Hystrix仅仅会合并单个Tomcat线程的请求。

例如，如果一个用户想要加载300个影片的标签，Hystrix能够把这300次网络调用合并成一次调用。

- 对象建模和代码的复杂性

有时候，当你创建一个对象模型对消费的对象而言是具有逻辑意义的，这与对象的生产者的有效资源利用率不匹配。

例如，给你300个视频对象，遍历他们，并且调用他们的getSomeAttribute()方法，但是如果简单的调用，可能会导致300次网络调用（可能很快会占满资源）。

有一些手动的方法可以解决这个问题，比如在用户调用getSomeAttribute()方法之前，要求用户声明他们想要获取哪些视频对象的属性，以便他们都可以被预取。

或者，您可以分割对象模型，以便用户必须从一个位置获取视频列表，然后从其他位置请求该视频列表的属性。

<br/>

### 请求合并之HystrixCollapser 

让我们看一下 HystrixCollapser 类的实现,除了一些getset，比较核心的是下面这3个方法

```java
public abstract class HystrixCollapser<BatchReturnType, ResponseType, RequestArgumentType> 
    implements HystrixExecutable<ResponseType>, HystrixObservable<ResponseType> {
        ...
     public abstract RequestArgumentType getRequestArgument();
      
      protected abstract HystrixCommand<BatchReturnType> createCommand(Collection<HystrixCollapser.CollapsedRequest<ResponseType, RequestArgumentType>> var1);
      
      protected abstract void mapResponseToRequests(BatchReturnType var1, Collection<HystrixCollapser.CollapsedRequest<ResponseType, RequestArgumentType>> var2);
}
```

从HystrixCollapser抽象类的定义中可以看到，它指定了三个不同的参数类型

- BatchReturnType: 合并后批量请求的返回类型。
- ResponseType： 单个请求返回的类型
- RequestArgumentType: 请求参数类型

这3个参数类型，我们可以从这些调用方法中查看到 是我们指定了传入的参数类型，和批量返回的参数类型是什么。

<br/>

那么请求合并具体怎么使用呢?

**请求合并的实现类**

请求合并的实现包含了两个主要实现类，一个是合并请求类，一个是批量处理类，合并请求类的作用是收集一定时间内的请求，将他们传递的参数汇总，然后调用批量处理类，通过向服务调用者发送请求获取批量处理的结果数据，最后在对应的request方法体中依次封装获取的结果，并返回回去



实战代码，待补充。。。

这里推荐一个 多个请求合并请求 用户服务的例子

https://www.jianshu.com/p/8f5d1fd6d601

