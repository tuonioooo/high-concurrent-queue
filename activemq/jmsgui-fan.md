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

