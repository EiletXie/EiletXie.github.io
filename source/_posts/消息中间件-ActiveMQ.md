---
title: 消息中间件-ActiveMQ
date: 2019-08-25 14:20:35
tags:
- ActiveMQ
categories:
- JAVA
toc: true
---



# 消息中间件-ActiveMQ

## ActiveMQ 安装

和tomcat差不多，都需要Java环境

{% asset_img install.jpg mq安装 %}

<!--more-->

这里在公司的虚拟机上

{% asset_img  start.jpg  %}

**grep -v grep**

反转查找，即 查找不包含grep的进程

#### **查看端口**

- **netstat -anp | grep 61616**
- **lsof -i:61616**

{% asset_img   netstat.jpg  %}



#### **带日志启动**

{% asset_img   startwithlog.jpg  %}



#### 访问：

后台端口 61616

前端访问端口 8161



### **一个电商系统的模块调用图**

{% asset_img  shopmodule.jpg  %}

## JMS开发步骤

{% asset_img  jmsStep.jpg  %}

### 个人总结JMS基础开发步骤

1、 创建一个 ConnectFactory ,带 url

2、 通过factory创建 Connection,并启动 connection

3、 创建Session

4、 通过session创建 Destination对象，Queue、Topic

5、 创建消费者 Consumer或者注册一个 JMS message listenter 或者 生产者 Producer,并设置 Destination

6、 发送或者接收 JMS message

7、 关闭JMS资源



当一个服务提供者提供6条信息，2条消费者监听该服务**队列**，则一人一半，还是按照顺序分配的



## 两种Destination模式

队列： 1 ： 1，1对1，当一个生产者队列对应了多个消费者时，消息采用轮播的方式，但是一条消息只能给一个消费者

主题： 类似微信公众号 1 ： N, 当订阅者的数量N为0时，主题生产者发送的消息会当作废消息处理。



## JMS Message的组成：

### 消息头

**核心的五个属性**

- JMSdesination ：消息发送的目的地，主要是指Queue与Topic
- JMSDeliveryMode：持久与非持久模式

​               持久模式下，一条消息发送中如果出现故障会再次发送，非持久模式下最多发送一次，出现故障消息会永久丢失

- JMSExpiration： 消息过期时间，默认永不过期
- JMSPriority： 消息优先级，要求加急消息要先于普通消息，0-4是普通消息，5-9是加急消息，默认是4级，同类型的消息如 9 不一定比7快，但一定比4快。
- JMSMessageID ： 唯一识别每个消息的表示，由MQ产生



### 消息体

**封装具体消息的数据**

**发送和接受的消息体类型必须是一致对应的**

**主要是 TextMessage,MapMessage**

- TextMessage: 普通的字符串消息，包含一个string
- MapMessage: 一个map类型的消息，key为stirng类型，而value 为java的基本类型
- BytesMessage: 二进制数组消息，包含一个byte[]
- StreamMessage: java数据流消息，用标准流操作来顺序的填充或读取
- ObjectMessage: 对象消息，包含一个可序列化的java对象



### 消息属性

如果需要除消息头字段以外的值，那么可以使用消息属性

识别/去重/重点标注等操作很有用

setProperty 类型 如 setStringProperty



## JMS的可靠性：

### 持久（Persistent): 

**默认为持久化**

- 非持久化：messageProducer.setDeliveryMode(DeliVeryMode.NON_PERSISTENT         特点： 当服务器挂掉的时候，消息丢失。
- 持久化：服务器挂掉的时候，未消费的消息仍然存在，可以继续消费。



其中主体topic持久化要想接到消息的前提是，必须在消息发送之前订阅过该主题，就跟微信公众号一样。订阅后不管消费者关闭后再次启动，关机期间服务端主题发送的消息都会再启动后接收到

### 事务：

 一旦开启事务，消息发送和接受，session需要commit才生效

### 签收：

- 非事务： 默认自动签收、可设置手动签收 Session.CLIENT_ACKONWLEDGE，重复消息
- 事务： 当设置为事务时，只要session.commit()不管签收的模式是什么，可当做自动签收。当未session.commit()时，手动签收模式是否确认签收，都无效，总会收到重复消息，因为事务优先级最高。



## ActiveMQ的Broker

一个具体的服务器MQ实例，可以被JAVA创建一个简易的MQ内核开启关闭，同时支持外部访问，不过一般还是用linux上的MQ



## ActiveMQ的传输协议

**官网介绍：**

http://activemq.apache.org/configuring-transports.html

ActiveMQ的核心协议在下面



如果不特别指定ActiveMQ的网络监听端口，那么这些端口都将使用BIO网络IO模型（openWire,STOMP,AMQP..)

所以要提升单节点的传输性能，我们需要底层配置协议

{% asset_img   transports.jpg  %}

而 ActiveMQ自身的配置文件 activemq.xml中 端口配置情况

{% asset_img  xml-port.jpg  %}

**看到没有？ 不同的协议设置了不同的网络监听端口，默认是一个端口一个协议**

URL描述信息的头部都是采用协议名称，例如：

描述 amqp协议的监听端口时，采用的URL描述格式为 “amap://....”;

但唯独描述 openwire协议描述时，URL头却采用 “tcp://...."

为什么？

**因为 ActiveMQ的默认消息协议就是 openwire 协议。**



### ActiveMQ与NIO 网络通信模型：

NIO协议与TCP协议类似，但NIO更加侧重底层的访问操作。它允许开发人员对同一资源的更多Client调用和服务端有更多负载。

从上述的描述中我们可以明白 NIO相比 TCP 在底层资源这块处理更佳



#### 适用场景：

- ​      可能存在 大量的Client连接到Broker上，一般情况下，大量的Client去连接Broker是会被操作系统限制线程数的，然而 NIO 的实现相比 TCP 可以做到 更少的线程去运行。
- ​      可能对于Broker有一个很迟钝的网络传输，NIO比TCP提供更好的性能。

### 如何让一个端口可以支持 相关网络模型 以及多协议呢？

{% asset_img  port-more.jpg  %}

从5.13版本开始，auto 关键字支持4种协议之间的互通，这里auto类似java中这个4个协议统一实现的接口一样。

官网AUTO介绍：http://activemq.apache.org/auto



#### 例如： 让 5671 端口同时支持NIO网络模型和多协议

localhost设置为 0.0.0.0

在activemq.xml中配置

```xml
<transportConnector name="auto+nio" uri="auto+nio://localhost:5671"/>
```

*重启动服务器后*

- *我们想使用自带的4种协议的时候，不用写具体详细名，可以直接使用  tcp://服务器地址:5671，它会自动根据代码去适配这四种协议中的一种。*
- *如果我们想使用 NIO进行单节点数据传输协议，我们就 nio://服务器地址:5671*



*当然生产环境可不能这么简单的配置，好歹也要设置最大连接数量，最大消息的大小设置,NIO的线程池的最大工作线程数量。*

```xml
<transportConnector name="auto+nio" uri="auto+nio://0.0.0.0:61608?maximumConnections=1000&wireFormat.maxFrameSize=104857600&org.apache.activemq.transport.nio.SelectorManager.corePoolSize=20&org.apache.activemq.transport.nio.SelectorManamaximumConnections=1000&wireFormat.maxFrameSize=104857600&org.apache.activemq.transport.nio.SelectorManager.corePoolSize=20&org.apache.activemq.transport.nio.SelectorManager.maximumPmPoolSize=50" /> 
```

官网上根据URL参数类型的作用，在不同网页介绍，没个统一参数表让我看的很迷糊。

具体URL参数信息请看官网

{% asset_img  configure-uri.jpg %}



## ActiveMQ 的持久化机制：

​       为了避免意外宕机后丢失数据，消息系统一般都会采用持久化机制来保证重启后可以恢复消息队列。

​       ActiveMQ的持久化机制有JDBC,AMQ,KahaDB和LevelDB,无论使用哪种持久化方式，消息的存储逻辑都是一样的。



### AMQ Message Store:

一种的文件存储形式，具有写入速度快和容易恢复的特点。消息存储在一个个文件中，文件默认大小为32M，如果文件中的消息全部被消费，则该文件会被标记可删除，下一个清除阶段会被删除，**适用于MQ5.3以前的版本。**



### KahaDB消息存储：

基于日志文件，从 ActiveMQ5.4版本默认使用的持久化插件。

即使是最新的5.15版本，我自己装的就是5.15.9，配置文件中仍是kahaDB

{% asset_img  persistence-kahadb.jpg  %}

它与其他持久化机制相比，文件组成十分简洁，仅有4类文件和一个lock文件锁：

{% asset_img  kahadb-component.jpg %} 

**介绍：**

#### 1、 db-<NUMBER>.log

KahaDB存储消息到预定义大小的数据记录文件中，文件命名为 db-<Number>.log。当数据文件已满时，文件会随之创建，number数值也会递增，它随之消息数量的增多，如每32M一个文件，文件名按照数字进行编号，如 db-1.log,db-2.log ,db-3.log,当不再有引用到数据文件中的任何消息时，文件会被删除或归档。

#### 2、 db.data

该文件包含了持久化的B-Tree索引，索引了消息数据记录中的消息，它是消息的索引文件，本质是B-Tree,B-Tree作为索引指向db-<Number>.log里面存储的信息。

#### 3、db.free

当前db.data文件里哪些页面是空闲的，文件具体内容是所有空闲页的ID

#### 4、db.redo

作为消息恢复，如果KahaDB消息存储在强制退出后启动，用于恢复B-Tree的索引

#### 5、lock文件锁

表示当前获取KahaDB读取权限的broker



### JDBC消息存储：

也就是说直接存储进mysql

#### jdbc配置：

第一步： 将 mysql连接包传入 activemq服务器lib目录下，我这里是 mysql-connector-java-5.1.38.jar



第二步：在配置文件activemq.xml中将持久化机制设置为 jdbc，这个数据库名称mysql-ds是自己取的,注意要加一个 # ，我这里使用的是默认的名称

```xml
<persistenceAdapter> 
  <jdbcPersistenceAdapter dataSource="#mysql-ds"/> 
</persistenceAdapter>
```

第三步：在配置文件activemq.xml中配置mysql数据库的信息,单独作为一个标签

注意： activemq自带的是连接池jar包为 dbcp2，如果是想改成 C3PO连接池或者阿里的，请记得导入相应的jar包到 activemq服务器lib目录下

```xml
 <!-- MySql DataSource Sample Setup --> 
  <bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close"> 
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/> 
    <property name="url" value="jdbc:mysql://localhost/activemq?relaxAutoCommit=true"/> 
    <property name="username" value="activemq"/> 
    <property name="password" value="activemq"/> 
    <property name="poolPreparedStatements" value="true"/> 
  </bean> 
```

当我配置好，重启时发现无进程运行，又不报错

然后发现了许多问题，如果有重启不成功的，可以查看我的一篇activemq的jdbc持久化问题解决的文章。

**队列 Queue:**

由于默认是持久化的，当我们队列点对点的发送消息，mysql的activemq_msgs表中有相关数据记录，这是因为持久化，消息保存在文件或者数据库中，当有任意消费者消费信息后会删除相应的信息。

而当我们设置为非持久化时，消息是保存到内存中。

**主题 Topic:**

主题的发布者会将发布消息持久化到activemq_msgs表，同时订阅者记录在activemq_acks表中，消费后不删除消息记录。

**activemq_acks**

{% asset_img  activemq_acks.jpg  %}

**activemq_msgs**

{% asset_img  activemq_msgs.jpg %}



### ActiveMQ Journel

每次消息的创建和消费，都需要去读写库。

ActiveMQ journal 使用高速缓存写入技术，大大提高了性能。

**当消费者的消费速度能够及时跟得上生产者消息的生产速度时，journal文件能够大大减少需要写入到DB的消息。**



**这句话是什么意思呢？举个栗子**

生产者生产了1000条消息，这1000条消息存入 journal文件中，如果消费者消费速度很快，消费了9000条，那么这个时候就只要同步剩下的1000条到数据库中，如果消费者消费速度慢，journal文件就会批量处理的方式写入到DB中。



也就是说，消息消费快的时候， 大部分逻辑都在journal的高速内存中处理了，有点像redis哈。

#### 如何配置 Journal

注意 dataSource的名称

```xml
  <persistenceFactory>
       <journalPersistenceAdapterFactory 
       journalLogFiles="5"
       journalLogFileSize="32768"
       useQuickJournal = "true"
       dataDirectory="activemq-data" 
       dataSource="#mysql-ds"/>
  </persistenceFactory> 
```

**修改 activemq.xml配置文件**

以前是

{% asset_img  persistence_jdbc.jpg %}

现在改成

{%  asset_img  persistence_journal.jpg %}

**测试：**

当我们用生产者产生队列消息，刷新几秒都不会存入到mysql数据库，因为这个时候队列消息是存在journal高速缓存中。而消费者可以消费到产生的消息。当我们几分钟都不消费，才会写入到mysql数据库中。



## ActiveMQ多节点集群：

### **如何保证MQ高可用？**

集群就完事了

{%  asset_img  zookeeper.jpg %}

activemq5.9版本以后，基于zookeeper和leveldb搭建activemq集群 集群仅提供主备方式的高可用集群功能，避免单点故障



zookeeper+replicated-leveldb-store的主从集群

### zookeeper与activemq原理

使用ZooKeeper实现的Master-Slave实现方式，是对ActiveMQ进行高可用的一种有效的解决方案，高可用的原理：使用ZooKeeper（集群）注册所有的ActiveMQ Broker。只有其中的一个Broker可以对外提供服务（也就是Master节点），其他的Broker处于待机状态，被视为Slave。如果Master因故障而不能提供服务，则利用ZooKeeper的内部选举机制会从Slave中选举出一个Broker充当Master节点，继续对外提供服务。

由于zookeeper没学，暂时这里不写了

### 配置步骤：

**1、配置的话基本就是将activemq复制3份，然后配置相关端口**

{%  asset_img  zk_port.jpg %}



**2、 修改管理控制台端口：**

配置各个节点的端口，在jetty.xml 配置文件中 

{%  asset_img  zk_jetty.jpg %}



**3、修改节点服务器的hostname，统一名称**

{%  asset_img  zk_hostname.jpg %} 



**4、在3个节点上的activemq.xml 配置持久化设置 ： zookeeper集群**

注意这个bind端口的设置，3台机器对应不同的bind端口

```xml
<persistenceAdapter>
         <replicatedLevelDB
           directory="${activemq.data}/leveldb"
           replicas="3"
           bind="tcp://0.0.0.0:63632"
           zkAddress="localhost:2191,localhost:2192,localhost:2193"
           hostname="duanyuefeng-server"
           sync="local_disk"
           zkPath="/activemq/leveldb-stores"
           />
    </persistenceAdapter>
```



**5.按照顺序启动3个ActiveMQ节点， 到这步前题是zk集群已经成功启动运行**

**测试可靠性，kill掉当前master的机器，看zookeeper是否从其他的slave中选择一台作为master**

{%  asset_img  zk_master.jpg %} 



## ActiveMQ 高级特性：

### 1、引入消息队列之后该如何保证其高可用性

zookeeper和leveldb搭建activemq集群



### 2、异步投递Async Sends

ActiveMQ支持同步、异步两种发送的模式将消息发送到broker，模式的选择对发送延迟有巨大的影响。produce能达到怎样的产出率

（产出率=发送数据总量/时间）主要受发送延时的影响，使用异步发送可以显著的提高发送的性能。



**ActiveMQ默认使用异步发送的模式**

**同步发送的情景：**

- 明确指定使用同步发送的方式
- 未使用事务的前提下发送持久化的信息



如果你没有使用事务且发送的是持久化信息，每一次发送都是同步发送的且会堵塞producer 直到broker返回一个确认，表示消息已经被安全的持久化到磁盘。

确认机制提供了消息安全的保障，但同时会阻塞客户端带来很大的延时。



很多高性能的应用，**允许在失败的情况下有少量的数据丢失**。如果你的应用满足这个特点，你可以使用异步发送来提高生产率，即使发送的是持久化的消息。



**异步发送**

它可以最大化producer端的发送效率，我们通常在发送消息量比较密集的情况下使用异步发送，可以很大的提升 producer性能。

不过这也带来了额外的问题

就是需要消耗较多的Client端内存同时也会导致broker端性能消耗增加；

此外它不能有效的确保消息的发送成功。在useAsyncSend=true的情况下，客户端需要容忍消息丢失的可能。

设置方式

{%  asset_img  async_setting.jpg %} 



### 3、 那如何确定异步发送成功呢？

#### **异步发送丢失消息的场景是：**

生产者设置 UseAsyncSend=true,使用producer.send(msg)持续发送消息。

由于消息不阻塞，生产者会认为所有send消息均被成功发送至MQ

如果MQ突然宕机，此时生产者端内存中尚未被发送至MQ的消息 会丢失。

所以，正确的异步发送方法是需要接受回调的。



#### **同步发送和异步发送的区别：**

同步发送等send不阻塞了就表示一定发送成功了。

异步发送需要接受回执并由客户端再判断一次是否发送成功。



需要写一个消息回调函数进行判断

```java
message = session.createTextMessage("async msg---" + i);
message.setJMSMessageID(UUID.randomUUID().toString() + "--order ");
String msgID = message.getJMSMessageID();
activeMQMessageProducer.send(message, new AsyncCallback() {
    @Override
    public void onSuccess() {
        System.out.println(msgID + "has been send success!");
    }

    @Override
    public void onException(JMSException e) {
        System.out.println(msgID + "fail to end!");
    }
});
```



## 3、 延时投递和定时投递

 需开启 activemq.xml中 broker标签的一个属性 schedulerSupport

{% asset_img  delay_setting.jpg %}

示例：

{% asset_img  delay_example.jpg %}

### 四大属性

{% asset_img  delay_property.jpg %}

### 官网设置的代码

{% asset_img  delay_setting2.jpg %}

示例：设置消息延迟3秒发送，重复发送5次，每次隔4秒



```java
long delay = 3 * 1000;
long period = 4 * 1000;
int repeat = 5;


TextMessage message = null;
for (int i = 1; i <= 3; i++) {
    message = session.createTextMessage("delay msg---" + i);
    message.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY,delay);
    message.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_PERIOD,period);
    message.setIntProperty(ScheduledMessage.AMQ_SCHEDULED_REPEAT,repeat);

}
```


------
## ActiveMQ的消息重试机制：

### 哪些情况会导致消息重发？

1. Client用了 transactions 且 在session中调用了 rollback()
2. Client用了transactions 且在调用commit() 之前关闭或者没有commit
3. Client在CLIENT_ACKNOWLEDGE的传递模式下，在session中调用了recover()



### 消息时间重发间隔与重发次数？

间隔： 1

次数： 6




### 有毒消息 Posion ACK 

{% asset_img  posion_ack.jpg %}

一个消息被redelivedred 超过默认的最大重发次数（6次），消费端会给MQ发送一个 “poison ack”表示这个消息有毒，告诉broker不要再发了，这个时候broker会把这个消息放到DLQ（死信队列）

{% asset_img  posion_property.jpg %}

测试： 消费者开启事务，session不commit，消费7次后（第一次失败+6次重发）

标记为死信队列

{% asset_img  activemq_DLQ.jpg %}

{% asset_img  mysql_DLQ.jpg %}

可以自己设置相关属性

```java
ActiveMQConnectionFactory activeMQConnectionFactory = new ActiveMQConnectionFactory(ACTIVEMQ_URL);

RedeliveryPolicy redeliveryPolicy = new RedeliveryPolicy();
// 设置最大重发次数为3次
redeliveryPolicy.setMaximumRedeliveries(2);
activeMQConnectionFactory.setRedeliveryPolicy(redeliveryPolicy);
Connection connection = activeMQConnectionFactory.createConnection();
connection.start();

```

------


## 死信队列：

{% asset_img  DLQ_example.jpg %}

{% asset_img  module_DLQ.jpg %}





## 如何保证消息不被重复消费呢？谈谈幂等性问题。

给消息一个唯一主键，如果数据库已有该主键，证明重复消费了。

或者是用redis 记录<ID,message>

redis,set(id,message)

set(ID: xx-324234-4234234-1:1:!:!,UUID.randomUUID());

做数据重复检测