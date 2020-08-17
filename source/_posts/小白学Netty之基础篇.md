---
title: 小白学Netty之基础篇
date: 2020-08-17 21:43:06
tags:
- Netty
categories:
- JAVA
toc: true
---





### 前言

​              最近几个月除了新公司的业务熟悉，在业余时间看了相关的netty、flink的视频和书籍，但是Netty这个东西背后涉及 网络、并发、以及操作系统原理等，我即使花了时间看视频，看Netty实战，总结成思维导图依旧没有那种对一门技术的掌握感，为此我决定将其总结成一系列的技术博客来加深自己对Netty的理解，就叫 《小白学Netty》吧。至于Flink，目前我平常开发的内容并未涉及Flink，我这边暂时不深入学习，后续有机会再开展。

<br/>

这第一篇文章我从Netty的起源、Netty的作用，以及Netty的大体设计来做一个概述。

<!--more--> 

### 1、 什么是**Netty**

​     Netty是一款异步的事件驱动的网络应用程序框架，支持快速地开发可维护的高性能的面向协议的服务器和客户端

<br/>

### 2、Netty的起因

​    早期的网络编程是 阻塞IO的交互，在早期的Java API（Java.net)只支持由本地系统套接字库提供的所谓的阻塞函数。

**举个阻塞IO的栗子：**

```java
ServerSocket serverSocket = new ServerSocket(portNumber);
Socket clientSocket = serverSocket.accept();            
BufferedReader in = new BufferedReader(                    
        new InputStreamReader(clientSocket.getInputStream()));
PrintWriter out =
        new PrintWriter(clientSocket.getOutputStream(), true);
String request, response;
while ((request = in.readLine()) != null) {                 
    if ("Done".equals(request)) {                         
        break;
    }
}
response = processRequest(request);                        
out.println(response);  
```

这里需要说明 serverSocket的 accept()方法会一直阻塞到一个连接生成，随后会返回一个新的Socket用于客户端和服务器之间的通信。

{% asset_img 图1-1.png 图1-1%}



那么看代码，我们可以知道，这种阻塞IO的写法一次只能处理一个连接，如果要同时管理过个并发客户端，需要为每个新的客户端对应的Socket连接创建一个新的Thread来处理。

<br/>

#### 这种做法的缺点是什么？

1、任何时刻都会可能存在大量线程处于闲置状态，不停的在等待输入或输出数据准备就绪，这是一种资源的浪费。

2、需要为每个线程的调用栈都分配内存，其默认值大小区间为 64KB 到 1MB，取决于操作系统

3、第三点涉及操作系统的原理，CPU执行大量的线程时，上下文切换开销比较大，任务执行速度远慢于 CPU核数 * 2左右的线程数（这个感兴趣的小伙伴可以深入去了解为什么会这样？）



那会有小伙伴说了2002年起Java不是支持 NIO 非阻塞IO吗？

对的，在NIO中 java.nio.channels.Selector 的非阻塞IO实现的关键。

它是使用事件通知API来确定在一组 **非阻塞套接字** 中有哪些已经就绪能够进入IO相关的操作，因为任何时间检查任意的读写操作的完成状态，所以一个**单一的线程便可以处理多个并发的连接**

{% asset_img 图1-2.png 图1-2%}

<br/>

#### NIO与 阻塞IO 比较

- 使用较少的线程处理许多的连接，从而减少了内存管理和上下文切换所带来的开销
- 当没有IO操作需要处理的时候，线程也可以被用于其他任务



这里可能大家对第二点不够理解，我这里再做说明

> NIO 是一种**同步非阻塞**的 IO 模型。
>
> 同步是指线程不断轮询 IO 事件是否就绪，非阻塞是指线程在等待 IO 的时候，可以同时做其他任务。
>
> 同步的核心就是 Selector，Selector 代替了线程本身轮询 IO 事件，避免了阻塞，同时减少了不必要的线程消耗；
>
> 非阻塞的核心就是通道Channel 和缓冲区 Buffer，当 IO 事件就绪时，可以通过写到缓冲区，保证 IO 的成功，而无需线程阻塞式地等待。



<br/>

那么Netty比 原生的NIO又强大在于哪里呢？

Java是一门面向对象的语言，通过抽象来隐藏其背后NIO的复杂性，基于操作系统和网络协议等多方面因素考虑，将NIO进行了封装，提供更加简单的API,更强大的性能，以及更强的安全性。

{% asset_img 图1-3.png 图1-3%}

<br/>

### Netty的核心组件

1、Channel

2、回调

3、Future

4、事件 和 ChannelHandler



<br/>

#### **Channel**

   Channel，你可以把它看做是 传入（入站） 或者 传出（出站）数据的载体，类似 我们常见的水在水管中流动，数据在Channel中传输。

<br/>

#### **回调**

   一个回调其实就是一个方法，一个指向已经被提供给另外一个方法的方法的引用，这使得接受回调的方法 可以在适当的时候 调用前者。其实就是用来操作完成后通知相关方，Netty在内部使用回调来处理事件，当一个回调被触发，相关的事件会被 一个 ChannelHandler 对应的实现来处理。

<br/>

#### **Future**

  Future提供了另外一种在操作完成时通知应用程序的方式。 这个对象可以看作是一个异步操作的结果占位符。

  它将在未来的某个时刻完成，并提供对其结果的访问。

  JDK提供的 java.util.concurrent.Future是只能通过 手动检查其是否完成 或者 一直阻塞直到操作完成，这是比较笨的。

  Netty提供了 ChannelFuture 可以执行异步操作，监听器对应的回调方法 operationComplete() 会在对应操作完成时被调用

  Netty是完全是异步和事件驱动的



**事件和ChannelHandler**

​     Netty通过使用不同的事件来通知我们状态的改变或者操作的状态。

<br/>

### NIO Reactor模型



{% asset_img 图1-4.png  图1-4%}

<br/>

#### Netty Reactor模型架构干了什么事情？

1、 Netty将一个网络通信的处理对应的多个步骤 accept、read、decode、process、encode、write、send，转换成了多个Task，每个Task对应一个事件，这样的好处在于 细粒度，每个事件都可以异步处理。

2、每当一个Task完成，即一个事件完成后会通知该 Reactor架构，Reactor调用对应网络事件的ChannelHandler来处理

<br/>

### **知识点思维导图**

初略了总结了一下Netty的常用API，后续针对核心的API做源码讲解和流程阐述。

{% asset_img 图1-5.png  图1-5%}