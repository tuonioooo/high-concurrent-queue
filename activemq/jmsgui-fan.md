# JMS规范

### 参考文档：

_JMS规范教程.pdf_

链接: [https://pan.baidu.com/s/1GZfmss-T6jCtyES-c66cJg](https://pan.baidu.com/s/1GZfmss-T6jCtyES-c66cJg) 密码: m8vf

## 概述

JMS即[Java消息服务](https://baike.baidu.com/item/Java消息服务)（Java Message Service）应用程序接口，是一个[Java平台](https://baike.baidu.com/item/Java平台)中关于面向[消息中间件](https://baike.baidu.com/item/消息中间件/5899771)（MOM）的[API](https://baike.baidu.com/item/API/10154)，用于在两个应用程序之间，或[分布式系统](https://baike.baidu.com/item/分布式系统/4905336)中发送消息，进行异步通信。Java消息服务是一个与具体平台无关的API，绝大多数MOM提供商都对JMS提供支持。JMS是一种与厂商无关的 API，用来访问消息收发系统消息，它类似于[JDBC](https://baike.baidu.com/item/JDBC)\(Java Database Connectivity\)。这里，JDBC 是可以用来访问许多不同关系数据库的 API，而 JMS 则提供同样与厂商无关的访问方法，以访问消息收发服务。许多厂商都支持 JMS，包括 IBM 的 MQSeries、BEA的 Weblogic JMS service和 Progress 的 SonicMQ。 JMS 使您能够通过消息收发服务（有时称为消息中介程序或路由器）从一个 JMS 客户机向另一个 JMS客户机发送消息。消息是 JMS 中的一种类型对象，由两部分组成：报头和消息主体。报头由路由信息以及有关该消息的元数据组成。消息主体则携带着应用程序的数据或有效负载。根据有效负载的类型来划分，可以将消息分为几种类型，它们分别携带：简单文本\(TextMessage\)、可序列化的对象 \(ObjectMessage\)、属性集合 \(MapMessage\)、字节流 \(BytesMessage\)、原始值流 \(StreamMessage\)，还有无有效负载的消息 \(Message\)。

## 专业技术规范

JMS（Java Messaging Service）是Java平台上有关面向消息中间件\(MOM\)的技术规范，它便于消息系统中的Java应用程序进行消息交换,并且通过提供标准的产生、发送、接收消息的接口简化企业应用的开发，翻译为Java消息服务。

## 历史

Java消息服务是一个在 Java标准化组织（JCP）内开发的标准（代号JSR 914）。2001年6月25日，Java消息服务发布JMS 1.0.2b，2002年3月18日Java消息服务发布 1.1，统一了消息域。

## 体系架构

JMS由以下元素组成。

* JMS提供者

连接面向消息中间件的，JMS接口的一个实现。提供者可以是Java平台的JMS实现，也可以是非Java平台的面向消息中间件的适配器。

* JMS客户

生产或消费基于消息的Java的应用程序或对象。

* JMS生产者

创建并发送消息的JMS客户。

* JMS消费者

接收消息的JMS客户。

* JMS消息

包括可以在JMS客户之间传递的数据的对象

* JMS队列

一个容纳那些被发送的等待阅读的消息的区域。与队列名字所暗示的意思不同，消息的接受顺序并不一定要与消息的发送顺序相同。一旦一个消息被阅读，该消息将被从队列中移走。

* JMS主题

一种支持发送消息给多个订阅者的机制。

## 对象模型

JMS对象模型包含如下几个要素：

1）连接工厂。连接工厂（ConnectionFactory）是由管理员创建，并绑定到[JNDI](https://baike.baidu.com/item/JNDI)树中。客户端使用JNDI查找连接工厂，然后利用连接工厂创建一个JMS连接。

2）JMS连接。JMS连接（Connection）表示JMS客户端和服务器端之间的一个活动的连接，是由客户端通过调用连接工厂的方法建立的。

3）JMS会话。JMS会话（Session）表示JMS客户与JMS服务器之间的会话状态。JMS会话建立在JMS连接上，表示客户与服务器之间的一个会话线程。

4）JMS目的。JMS目的（Destination），又称为[消息队列](https://baike.baidu.com/item/消息队列)，是实际的消息源。

5）JMS生产者和消费者。生产者（Message Producer）和消费者（Message Consumer）对象由Session对象创建，用于发送和接收消息。

6）JMS消息通常有两种类型：

① 点对点（Point-to-Point）。在点对点的消息系统中，消息分发给一个单独的使用者。点对点消息往往与队列（javax.jms.Queue）相关联。

② 发布/订阅（Publish/Subscribe）。发布/订阅消息系统支持一个事件驱动模型，消息生产者和消费者都参与消息的传递。生产者发布事件，而使用者订阅感兴趣的事件，并使用事件。该类型消息一般与特定的主题（javax.jms.Topic）关联。

[![](https://gss1.bdstatic.com/9vo3dSag_xI4khGkpoWK1HF6hhy/baike/s%3D220/sign=57e3b3b80c33874498c5287e610dd937/adaf2edda3cc7cd92cd7d9313901213fb90e9164.jpg)](https://baike.baidu.com/pic/JMS/2836691/0/adaf2edda3cc7cd92cd7d9313901213fb90e9164?fr=lemma&ct=single)

## 模型

Java消息服务应用程序结构支持两种模型：

点对点或队列模型

发布者/订阅者模型

在点对点或队列模型下，一个生产者向一个特定的队列发布消息，一个消费者从该队列中读取消息。这里，生产者知道消费者的队列，并直接将消息发送到消费者的队列。这种模式被概括为：

只有一个消费者将获得消息生产者不需要在接收者消费该消息期间处于运行状态，接收者也同样不需要在消息发送时处于运行状态。

每一个成功处理的消息都由接收者签收发布者/订阅者模型支持向一个特定的消息主题发布消息。0或多个订阅者可能对接收来自特定消息主题的消息感兴趣。在这种模型下，发布者和订阅者彼此不知道对方。这种模式好比是匿名公告板。这种模式被概括为：

多个消费者可以获得消息在发布者和订阅者之间存在时间[依赖性](https://baike.baidu.com/item/依赖性)。发布者需要建立一个订阅（subscription），以便客户能够订阅。订阅者必须保持持续的活动状态以接收消息，除非订阅者建立了持久的订阅。在那种情况下，在订阅者未连接时发布的消息将在订阅者重新连接时重新发布。

使用[Java语言](https://baike.baidu.com/item/Java语言)，JMS提供了将应用与提供数据的传输层相分离的方式。同一组Java类可以通过JNDI中关于提供者的信息，连接不同的JMS提供者。这一组类首先使用一个连接工厂以连接到队列或主题，然后发送或发布消息。在接收端，客户接收或订阅这些消息。

## 传递方式

JMS有两种传递消息的方式。标记为NON\_PERSISTENT的消息最多投递一次，而标记为PERSISTENT的消息将使用暂存后再转送的机理投递。如果一个JMS服务离线，那么持久性消息不会丢失但是得等到这个服务恢复联机时才会被传递。所以默认的消息传递方式是非持久性的。即使使用非持久性消息可能降低内务和需要的存储器，并且这种传递方式只有当你不需要接收所有的消息时才使用。

虽然JMS规范并不需要JMS供应商实现消息的优先级路线，但是它需要递送加快的消息优先于普通级别的消息。JMS定义了从0到9的优先级路线级别，0是最低的优先级而9则是最高的。更特殊的是0到4是正常优先级的变化幅度，而5到9是加快的优先级的变化幅度。举例来说： topicPublisher.publish \(message, DeliveryMode.PERSISTENT, 8, 10000\); //Pub-Sub 或 queueSender.send\(message,DeliveryMode.PERSISTENT, 8, 10000\);//P2P 　这个代码片断，有两种消息模型，映射递送方式是持久的，优先级为加快型，生存周期是10000 \(以毫秒[度量](https://baike.baidu.com/item/%E5%BA%A6%E9%87%8F)\)。如果生存周期设置为零，这则消息将永远不会过期。当消息需要时间限制否则将使其无效时，设置生存周期是有用的。

JMS定义了五种不同的消息正文格式，以及调用的消息类型，允许你发送并接收以一些不同形式的数据，提供现有消息格式的一些级别的兼容性。

· StreamMessage -- Java原始值的数据流

· MapMessage--一套名称-值对

· TextMessage--一个字符串对象

· ObjectMessage--一个序列化的 Java对象

· BytesMessage--一个未解释字节的数据流







  






