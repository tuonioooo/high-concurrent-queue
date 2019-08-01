# 消息确认机制\(AMQP事务\)

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

当我们使用txSelect提交开始事务之后，我们就可以发布消息给Broke代理服务器，如果txCommit提交成功了，则消息一定到达了Broker了，如果在txCommit执行之前Broker出现异常崩溃或者由于其他原因抛出异常，这个时候我们便可以捕获异常通过txRollback方法进行回滚事务了。

所以RabbitMQ事务中的主要代码为：

```
channel.txSelect();
channel.basicPublish(exchange, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes());
channel.txCommit();
```

先进行事务提交，然后开始发送消息，最后提交事务。

还是在原来的demo代码基础下，在sender和receiver包下分别新建TransactionSender1.java和TransactionReceiver1.java。分别如下所示：

```
TransactionSender1.java
 
package net.anumbrella.rabbitmq.sender;
 
import java.io.IOException;
import java.util.concurrent.TimeoutException;
 
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
 
public class TransactionSender1 {
 
    private final static String QUEUE_NAME = "transition";
 
    public static void main(String[] args) throws IOException, TimeoutException {
        /**
         * 创建连接连接到MabbitMQ
         */
        ConnectionFactory factory = new ConnectionFactory();
 
        // 设置MabbitMQ所在主机ip或者主机名
        factory.setUsername("guest");
        factory.setPassword("guest");
        factory.setHost("127.0.0.1");
        factory.setVirtualHost("/");
        factory.setPort(5672);
 
        // 创建一个连接
        Connection connection = factory.newConnection();
 
        // 创建一个频道
        Channel channel = connection.createChannel();
 
        // 指定一个队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 发送的消息
        String message = "This is a transaction message！";
 
        try {
            // 开启事务
            channel.txSelect();
            // 往队列中发出一条消息，使用rabbitmq默认交换机
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            // 提交事务
            channel.txCommit();
        } catch (Exception e) {
            e.printStackTrace();
            // 事务回滚
            channel.txRollback();
        }
 
        System.out.println(" TransactionSender1 Sent '" + message + "'");
        // 关闭频道和连接
        channel.close();
        connection.close();
    }
 
}

```



