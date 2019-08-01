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

**注意：发布确认和事务。\(两者不可同时使用\)在channel为事务时，不可引入确认模式；同样channel为确认模式下，不可使用事务。  
所以在使用事务时，在application.properties中，需要将确认模式更改为false。spring.rabbitmq.publisher-confirms=false**

* ### **同步**

通过设置RabbitTemplate的channelTransacted为true，来设置事务环境，使得可以使用RabbitMQ事务。如下：

template.setChannelTransacted\(true\);

在demo代码里面，主要是在config包下的RabbitConfig.java里的rabbitTemplateNew方法里面配置，如下：

```
@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public RabbitTemplate rabbitTemplateNew() {
        RabbitTemplate template = new RabbitTemplate(connectionFactory());
        template.setChannelTransacted(true);
        return template;
}
```

接着在在sender和receiver包，分别建立TransactionSender2.java和TransactionReceiver2.java。分别如下所示：

TransactionSender2.java

```
package net.anumbrella.rabbitmq.sender;

import java.text.SimpleDateFormat;
import java.util.Date;

import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
public class TransactionSender2 {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    @Transactional(rollbackFor = Exception.class)
    public void send(String msg) {
        SimpleDateFormat time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        String sendMsg = msg + time.format(new Date()) + " This is a transaction message！ ";
        /**
         * 这里可以执行数据库操作
         * 
         **/
        System.out.println("TransactionSender2 : " + sendMsg);
        this.rabbitTemplate.convertAndSend("transition", sendMsg);
    }

}
```

在上面代码中，我们通过调用者提供外部事务@Transactional\(rollbackFor = Exception.class\)，来现实事务方法。一旦方法中抛出异常，比如执行数据库操作时，就会被捕获到，同时事务将进行回滚，并且向外发送的消息将不会发送出去。

TransactionReceiver2.java

```
package net.anumbrella.rabbitmq.receiver;

import java.io.IOException;

import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import com.rabbitmq.client.Channel;

@Component
public class TransactionReceiver2 {

    @RabbitListener(queues = "transition")
    public void process(Message message, Channel channel) throws IOException {
        System.out.println("TransactionReceiver2  : " + new String(message.getBody()));
    }
}
```

添加完消息的发送者和接收者后，还需要在controller包下的RabbitTest.java中添加模拟消息发送的Restful接口方法，添加如下代码：

```
@Autowired
    private TransactionSender2 transactionSender;

    /**
     * 事务消息发送测试
     */
    @GetMapping("/transition")
    public void transition() {
        transactionSender.send("Transition:  ");
    }
```

启动wireshark，选择好网络，输入amqp过滤我们需要的信息。  
然后启动Spring Boot项目，访问接口[http://localhost:8080/rabbit/transition。](http://localhost:8080/rabbit/transition。)

在控制台我们可以得到消息已经发送和收到，

TransactionSender2 : Transition:  2018-06-18 23:00:16 This is a transaction message！  
TransactionReceiver2  : Transition:  2018-06-18 23:00:16 This is a transaction message！

查看wireshark如下：

![](/assets/20181227173121879.png)可以看到这里与前面我们讲解的原生事务是一致的，而当发送消息出现异常时，就会响应执行事务回滚。

* ### **异步**

刚才我们讲解的是同步的情况，现在我们讲解一下异步的形式。在异步当中，主要使用MessageListener 接口，它是 Spring AMQP 异步消息投递的监听器接口。而MessageListener的实现类SimpleMessageListenerContainer则是作为了整个异步消息投递的核心类存在。

接下来我们开始介绍使用异步的方法，同样表示需要的外部事务，用户需要在容器配置的时候指定PlatformTransactionManager的实现。代码如下：

```
    @Bean
    public SimpleMessageListenerContainer messageListenerContainer() {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory());
        container.setTransactionManager(rabbitTransactionManager());
        container.setChannelTransacted(true);
        // 开启手动确认
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        container.setQueues(transitionQueue());
        container.setMessageListener(new TransitionConsumer());
        return container;
    }
```

这段代码我们是添加在config下的RabbitConfig.java下，通过配置事务管理器，将channelTransacted属性被设置为true。

在容器中配置事务时，如果提供了transactionManager，channelTransaction必须为true，使得如果监听器处理失败，并且抛出异常，那么事务将进行回滚，那么消息将返回给消息代理；如果为false，外部的事务仍然可以提供给监听容器，造成的影响是在回滚的业务操作中也会提交消息传输的操作。

通过使用RabbitTransactionManager，这个事务管理器是PlatformTransactionManager接口的实现，它只能在一个Rabbit ConnectionFactory中使用。

注意：这种策略不能够提供XA事务，例如在消息和数据库之间共享事务。

除了上面的代码外，还有RabbitTransactionManager和TransitionConsumer需要添加，代码如下：

```
 /**
     * 声明transition2队列
     * 
     * @return
     */
    @Bean
    public Queue transitionQueue() {
        return new Queue("transition2");
    }

    /**
     * 事务管理
     * 
     * @return
     */
    @Bean
    public RabbitTransactionManager rabbitTransactionManager() {
        return new RabbitTransactionManager(connectionFactory());
    }

    /**
     * 自定义消费者
     */
    public class TransitionConsumer implements ChannelAwareMessageListener {

        @Override
        public void onMessage(Message message, Channel channel) throws Exception {
            byte[] body = message.getBody();
            System.out.println("TransitionConsumer: " + new String(body));
            // 确认消息成功消费
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
            // 除以0，模拟异常，进行事务回滚
            // int t = 1 / 0;
        }
    }
```

因为我们在container中设置队列为“transition2”，所以我们在TransactionSender2中更改发送的队列为“transition2”，如下：

this.rabbitTemplate.convertAndSend\("transition2", sendMsg\);  
接着我们启动wireshark，选择好网络，输入amqp过滤我们需要的信息。  
然后启动Spring Boot项目，访问接口[http://localhost:8080/rabbit/transition。](http://localhost:8080/rabbit/transition。)

我们可以在wireshark中看到有事务的提交，如下：

![](/assets/20181227173140307.png)

然后我们在TransitionConsumer中把除以0的模拟异常情况打开，然后再执行上面的操作，可得：

![](/assets/20181227173203538.png)可以看到先进行了事务提交，后面事务又回滚了。意味着消息没有接收成功，我们在RabbitMQ管理界面也可以查看到消息，如果将consumer关掉，则unacked的msg则会又回到了ready状态。（注意：这里我们模拟的是消费者接收事务，前面是消息生成者发送到Broker的事务）

