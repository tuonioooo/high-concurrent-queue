# 消息中间件的基本概念

## 什么是消息中间件

**消息中间件**是指一种在[需要](http://wiki.mbalib.com/wiki/%E9%9C%80%E8%A6%81)进行网络通信的[系统](http://wiki.mbalib.com/wiki/%E7%B3%BB%E7%BB%9F)进行通道的建立，[数据](http://wiki.mbalib.com/wiki/%E6%95%B0%E6%8D%AE)或[文件](http://wiki.mbalib.com/wiki/%E6%96%87%E4%BB%B6)发送的[中间件](http://wiki.mbalib.com/wiki/%E4%B8%AD%E9%97%B4%E4%BB%B6)。消息中间件的一个重要作用是可以_**跨平台操作**_，为不同操作系统上的[应用软件](http://wiki.mbalib.com/wiki/%E5%BA%94%E7%94%A8%E8%BD%AF%E4%BB%B6)集成提供便利。

　　现在越来越多的分布式应用系统采用消息中间件方式来构建，人们通过使用消息中间件把应用扩展到不同的操作系统和不同的网络环境。基于消息的机制更适用于由事件驱动的应用，当一个事件发生时，消息中间件[通知](http://wiki.mbalib.com/wiki/%E9%80%9A%E7%9F%A5)服务方[应该](http://wiki.mbalib.com/wiki/%E5%BA%94%E8%AF%A5)进行如何操作。

　　使用消息中间件编程可以很好地扩展到不同的操作系统和硬件平台上。可以将消息中间件的核心安装在需要进行消息传递的系统上，并在它们之间建立逻辑通道，由消息中间件实现消息发送。消息中间件既可以支持同步方式通讯，又可以支持异步方式通讯，实际上它是一种点到点的[机制](http://wiki.mbalib.com/wiki/%E6%9C%BA%E5%88%B6)，因而可以很好地适用于面向对象的编程方式。

## 消息中间件的工作原理

消息中间件的工作原理是:应用之间以一系列消息的方式进行通信。在发送者和接收者的传送过程中，[消息](http://wiki.mbalib.com/wiki/%E6%B6%88%E6%81%AF)保存在队列中，避免在传送过程中消息丢失，并且为接收者查看消息提供了一个区域，应用把消息发送到与接收者相关的队列中去，如果[发送](http://wiki.mbalib.com/wiki/%E5%8F%91%E9%80%81)者想及时得到反馈，他们就要把接收返回消息的队列名称包含在所有他们发送的[消息](http://wiki.mbalib.com/wiki/%E6%B6%88%E6%81%AF)中。消息传递机制要[保证](http://wiki.mbalib.com/wiki/%E4%BF%9D%E8%AF%81)将发送者的消息传送到[目的地](http://wiki.mbalib.com/wiki/%E7%9B%AE%E7%9A%84%E5%9C%B0)。在消息传递中，应用组件之间不必建立直接的联系，也就是发送方将消息放人队列中，然后接收方自己从队列中提取消息。发送方在[发送](http://wiki.mbalib.com/wiki/%E5%8F%91%E9%80%81)消息时不必关心接收方是否处于接收状态。

　　使用消息中间件编程采用的是消息中间件的[API](http://wiki.mbalib.com/wiki/API)，可以很好地扩展到不同的操作系统和硬件平台上。消息中间件的核心安装在[需要](http://wiki.mbalib.com/wiki/%E9%9C%80%E8%A6%81)进行消息传递的[系统](http://wiki.mbalib.com/wiki/%E7%B3%BB%E7%BB%9F)上，在它们之间建立逻辑通道，由消息中间件实现消息发送。消息中间件可以既支持同步方式，又支持异步方式，实际上它是一种点到点的机制，因而可以很好地适用于面向对象的编程方式。

## 消息中间件的功能及优点

消息中间件的任务除了以其高[可靠性](http://wiki.mbalib.com/wiki/%E5%8F%AF%E9%9D%A0%E6%80%A7)、高安全性传递消息之外，还应包括如下[服务](http://wiki.mbalib.com/wiki/%E6%9C%8D%E5%8A%A1):完成不同系统之间的数据转换、加密/解密、支持消息驱动处理模式的触发机制、向多个应用广播数据、发布订阅\(publish subscribe\)、错误恢复、网络资源[定位](http://wiki.mbalib.com/wiki/%E5%AE%9A%E4%BD%8D)、消息和请求的优先排序以及广泛的错误查询机制等。其中发布订阅是一种消息传递常用的形式，在这种形式中，应用对其感兴趣的主题进行登记，一旦主题被一个应用“订阅”，那么这个应用就会接收到与该主题相关的消息。

　　面向消息的中间件为开发者提供了如下优点:在不可靠的[网络](http://wiki.mbalib.com/wiki/%E7%BD%91%E7%BB%9C)上实现可靠通信;实现来自于不同平台和网络协议的应用间的无缝连接;简化开发模型等。面向消息的[中间件](http://wiki.mbalib.com/wiki/%E4%B8%AD%E9%97%B4%E4%BB%B6)的开发者可以直接调用发送/接收的[应用程序接口](http://wiki.mbalib.com/wiki/%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E6%8E%A5%E5%8F%A3)\(API\)实现应用程序之间的互操作，避免了系统底层的工作，不必考虑复杂的网络通信问题。



