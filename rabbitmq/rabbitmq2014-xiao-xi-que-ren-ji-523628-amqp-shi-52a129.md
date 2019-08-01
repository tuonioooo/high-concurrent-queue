# RabbitMQ—消息确认机制\(AMQP事务\)

我们知道可以通过持久化（交换机、队列和消息持久化）来保障我们在服务器崩溃时，重启服务器消息数据不会丢失。但是我们无法确认当消息的发布者在将消息发送出去之后，消息到底有没有正确到达Broker代理服务器呢？如果不进行特殊配置的话，默认情况下发布操作是不会返回任何信息给生产者的，也就是默认情况下我们的生产者是不知道消息有没有正确到达Broker的。如果在消息到达Broker之前已经丢失的话，持久化操作也解决不了这个问题，因为消息根本就没到达代理服务器，这个是没有办法进行持久化的，那么当我们遇到这个问题又该如何去解决呢？

这里就是我们讲解到的RabbitMQ中的消息确认机制，通过消息确认机制我们可以确保我们的消息可靠送达到我们的用户手中，即使消息丢失掉，我们也可以通过进行重复分发确保用户可靠收到消息。

今天我们讲解的RabbitMQ消息确认机制（面试会问消息的可靠性），主要包括两个方面，因为RabbitMQ为我们提供了两种方式：

通过AMQP事务机制实现，这也是AMQP协议层面提供的解决方案；

  
通过将channel设置成confirm模式来实现；

  
一、AMQP事务  
1、使用java原生事务  
我们知道事务可以保证消息的传递，使得可靠消息最终一致性。接下来我们先来探究一下RabbitMQ的事务机制。

RabbitMQ中与事务有关的主要有三个方法：

```
txSelect()
txCommit()
txRollback()
```

txSelect主要用于将当前channel设置成transaction模式，txCommit用于提交事务，txRollback用于回滚事务。

当我们使用txSelect提交开始事务之后，我们就可以发布消息给Broke代理服务器，如果txCommit提交成功了，则消息一定到达了Broke了，如果在txCommit执行之前Broker出现异常崩溃或者由于其他原因抛出异常，这个时候我们便可以捕获异常通过txRollback方法进行回滚事务了。

所以RabbitMQ事务中的主要代码为：

```
channel.txSelect();
channel.basicPublish(exchange, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes());
channel.txCommit();
```



