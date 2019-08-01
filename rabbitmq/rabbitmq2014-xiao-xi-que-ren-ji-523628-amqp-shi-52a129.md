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

TransactionSender1.java

```
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

在上面中我们使用try-catch来捕获异常，如果发送失败，就会进行事务回滚。

```
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
```

TransactionReceiver1.java

```
package net.anumbrella.rabbitmq.receiver;

import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.TimeoutException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class TransactionReceiver1 {

    private final static String QUEUE_NAME = "transition";

    public static void main(String[] argv) throws IOException, InterruptedException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();

        factory.setUsername("guest");
        factory.setPassword("guest");
        factory.setHost("127.0.0.1");
        factory.setVirtualHost("/");
        factory.setPort(5672);
        // 打开连接和创建频道，与发送端一样

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 声明队列，主要为了防止消息接收者先运行此程序，队列还不存在时创建队列。
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        System.out.println("Receiver1 waiting for messages. To exit press CTRL+C");

        // 创建队列消费者
        final Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
                SimpleDateFormat time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss:SSSS");

                String message = new String(body, "UTF-8");

                System.out.println(" TransactionReceiver1  : " + message);
                System.out.println(" TransactionReceiver1 Done! at " + time.format(new Date()));
            }
        };
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }

}
```

消息的接收者跟原来是一样的，因为事务主要是保证消息要发送到Broker当中。  
接着我们使用wireshark来监听网络，这里也可以使用Fiddler。由于笔者使用的是MAC系统，没有Fiddler版本。如果读者要使用Fiddler，同时使用windows可以看看这篇文章，后面有对Fiddler的介绍，JMeter搭配Fiddler的简单使用（一）。

启动wireshark，选择好网络，输入amqp过滤我们需要的信息。  
然后我们分别启动TransactionReceiver1.java 和 TransactionSender1.java。

![](/assets/20181227173020657.png)从上面我们可以清晰的看见消息的分发过程，与我们前面分析的一致。主要执行了四个步骤：

Client发送Tx.Select  
Broker发送Tx.Select-Ok\(在它之后，发送消息\)  
Client发送Tx.Commit  
Broker发送Tx.Commit-Ok  
接下来我们通过抛出异常来模拟发送消息错误，进行事务回滚。更改发送信息代码为：

```
 try {
            // 开启事务
            channel.txSelect();
            // 往队列中发出一条消息，使用rabbitmq默认交换机
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            // 除以0，模拟异常，使用rabbitmq默认交换机
            int t = 1/0;
            // 提交事务
            channel.txCommit();
        } catch (Exception e) {
            e.printStackTrace();
            // 事务回滚
            channel.txRollback();
        }
```

这里我们通过除以0来模拟抛出异常，接着按同样的顺序运行代码。

![](/assets/20181227173033348.png)

可以看见事务进行了回滚，同时我们在接收端也没有收到消息。

通过上面我们可以知道事务确实能够解决消息的发送者和Broker之间消息的确认，只有当消息成功被服务端Broker接收，并且接受时，事务才能提交成功，不然我们便可以在捕获异常进行事务回滚操作同时进行消息重发。

在上面的情况中，我们使用java原生代码来模拟事务进行发送，而在实际开发中，我们可能需要结合框架来完成。

2、结合Spring Boot来使用事务  
我们一般在Spring Boot使用RabbitMQ，主要是通过封装的RabbitTemplate模板来实现消息的发送，这里主要也是分为两种情况，使用RabbitTemplate同步发送，或者异步发送。

注意：发布确认和事务。\(两者不可同时使用\)在channel为事务时，不可引入确认模式；同样channel为确认模式下，不可使用事务。  
所以在使用事务时，在application.properties中，需要将确认模式更改为false。









