# 消息确认机制\(Confirm模式\)

在上一篇文章中我们讲解了RabbitMQ中的AMQP事务来保证消息发送到Broker端，同时我们可以在事务之间发送多条消息（即在channel.txSelect\(\)和channel.txCommit\(\)之间发送多条消息，通过使用事务来保证它们准确到达Broker），如果忘记了事务的使用，可以复习一下上一篇文章——[消息确认机制\(AMQP事务\)](/rabbitmq/rabbitmq2014-xiao-xi-que-ren-ji-523628-amqp-shi-52a129.md)

但是使用事务虽然可以保证消息的准确达到，但是它极大地牺牲了性能，因此我们为了性能上的要求，可以采用另一种高效的解决方案——通过使用Confirm模式来保证消息的准确性。

这里的Confirm模式可以分为两个方面来讲解，一是消息的生产者\(Producer\)的Confirm模式，另一个是消息的消费者\(Consumer\)的Confirm模式。

### **一.生产者\(Producer\)的Confirm模式**

通过生产者的确认模式我们是要保证消息准确达到Broker端，而与AMQP事务不同的是Confirm是针对一条消息的，而事务是可以针对多条消息的。

发送原理图大致如下：

为了使用Confirm模式，client会发送confirm.select方法帧。通过是否设置了no-wait属性，来决定Broker端是否会以confirm.select-ok来进行应答。一旦在channel上使用confirm.select方法，channel就将处于Confirm模式。处于 transactional模式的channel不能再被设置成Confirm模式，反之亦然。

这里与前面的一些文章介绍的一致，发布确认和事务两者不可同时引入，channel一旦设置为Confirm模式就不能为事务模式，为事务模式就不能为Confirm模式。

在生产者将信道设置成Confirm模式，一旦信道进入Confirm模式，所有在该信道上面发布的消息都会被指派一个唯一的ID\(以confirm.select为基础从1开始计数\)，一旦消息被投递到所有匹配的队列之后，Broker就会发送一个确认给生产者（包含消息的唯一ID）,这就使得生产者知道消息已经正确到达目的队列了，如果消息和队列是可持久化的，那么确认消息会将消息写入磁盘之后发出，Broker回传给生产者的确认消息中deliver-tag域包含了确认消息的序列号，此外Broker也可以设置basic.ack的multiple域，表示到这个序列号之前的所有消息都已经得到了处理。

Confirm模式最大的好处在于它是异步的，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用便可以通过回调方法来处理该确认消息，如果RabbitMQ因为自身内部错误导致消息丢失，就会发送一条basic.nack来代替basic.ack的消息，在这个情形下，basic.nack中各域值的含义与basic.ack中相应各域含义是相同的，同时requeue域的值应该被忽略。通过nack一条或多条消息， Broker表明自身无法对相应消息完成处理，并拒绝为这些消息的处理负责。在这种情况下，client可以选择将消息re-publish。

在channel 被设置成Confirm模式之后，所有被publish的后续消息都将被Confirm（即 ack）或者被nack一次。但是没有对消息被Confirm的快慢做任何保证，并且同一条消息不会既被Confirm又被nack。

开启confirm模式的方法  
生产者通过调用channel的confirmSelect方法将channel设置为Confirm模式，如果没有设置no-wait标志的话，Broker会返回confirm.select-ok表示同意发送者将当前channel信道设置为Confirm模式\(从目前RabbitMQ最新版本3.6来看，如果调用了channel.confirmSelect方法，默认情况下是直接将no-wait设置成false的，也就是默认情况下broker是必须回传confirm.select-ok的\)。

编程模式  
对于固定消息体大小和线程数，如果消息持久化，生产者Confirm\(或者采用事务机制\)，消费者ack那么对性能有很大的影响.

消息持久化的优化没有太好方法，用更好的物理存储（SAS, SSD, RAID卡）总会带来改善。生产者confirm这一环节的优化则主要在于客户端程序的优化之上。归纳起来，客户端实现生产者confirm有三种编程方式：

* **普通Confirm模式**：每发送一条消息后，调用waitForConfirms\(\)方法，等待服务器端Confirm。实际上是一种串行Confirm了，每publish一条消息之后就等待服务端Confirm，如果服务端返回false或者超时时间内未返回，客户端进行消息重传； 
* **批量Confirm模式**：每发送一批消息之后，调用waitForConfirms\(\)方法，等待服务端Confirm，这种批量确认的模式极大的提高了Confirm效率，但是如果一旦出现Confirm返回false或者超时的情况，客户端需要将这一批次的消息全部重发，这会带来明显的重复消息，如果这种情况频繁发生的话，效率也会不升反降；
* **异步Confirm模式**：提供一个回调方法，服务端Confirm了一条或者多条消息后Client端会回调这个方法。 

**1、普通Confirm模式**

ConfirmSender1.java：

```
package net.anumbrella.rabbitmq.sender;


import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.MessageProperties;
import org.apache.commons.lang.StringUtils;
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

/**
 * 这是java原生类支持RabbitMQ，直接运行该类
 */
public class ConfirmSender1 {

    private final static String QUEUE_NAME = "confirm";

    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        /**
         * 创建连接连接到RabbitMQ
         */
        ConnectionFactory factory = new ConnectionFactory();

        // 设置RabbitMQ所在主机ip或者主机名
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
        String message = "This is a confirm message！";

        channel.confirmSelect();
        final long start = System.currentTimeMillis();
        //发送持久化消息
        for (int i = 0; i < 5; i++) {
            //第一个参数是exchangeName(默认情况下代理服务器端是存在一个""名字的exchange的,
            //因此如果不创建exchange的话我们可以直接将该参数设置成"",如果创建了exchange的话
            //我们需要将该参数设置成创建的exchange的名字),第二个参数是路由键
            channel.basicPublish("", QUEUE_NAME, MessageProperties.PERSISTENT_BASIC, (" Confirm模式， 第" + (i + 1) + "条消息").getBytes());
            if (channel.waitForConfirms()) {
                System.out.println("发送成功");
            }else{
                // 进行消息重发
            }
        }
        System.out.println("执行waitForConfirms耗费时间: " + (System.currentTimeMillis() - start) + "ms");
        // 关闭频道和连接
        channel.close();
        connection.close();
    }
}
```

我们在代码中发送了5条消息到Broker端，每条消息发送后都会等待确认。

ConfirmReceiver1.java：

```
package net.anumbrella.rabbitmq.receiver;


import com.rabbitmq.client.*;

import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.TimeoutException;

/**
 * 这是java原生类支持RabbitMQ，直接运行该类
 */
public class ConfirmReceiver1 {

    private final static String QUEUE_NAME = "confirm";

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
        System.out.println("ConfirmReceiver1 waiting for messages. To exit press CTRL+C");

        // 创建队列消费者
        final Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                                       byte[] body) throws IOException {
                SimpleDateFormat time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss:SSSS");

                String message = new String(body, "UTF-8");

                System.out.println(" ConfirmReceiver1  : " + message);
                System.out.println(" ConfirmReceiver1 Done! at " + time.format(new Date()));
            }
        };
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```

我们开启WireShak，监听RabbitMQ消息的发送。然后我们直接运行ConfirmSender1.java类，可以不用运行ConfirmReceiver.java，因为我们主要是测试消息到达Broker端，这主要是涉及到Producer和RabbitMQ的服务端。

在控制台打印出了信息：

发送成功  
发送成功  
发送成功  
发送成功  
发送成功  
执行waitForConfirms耗费时间: 181ms

在RabbitMQ管理界面confirm队列里，我们可以查看到我们发送的5条消息数据。

在WireShark中也可以发现开启了Confirm模式，以及我们发送的5条消息。

接着我们启动ConfirmReceiver.java，可以收到我们发送的具体消息：

ConfirmReceiver1 waiting for messages. To exit press CTRL+C  
 ConfirmReceiver1  :  Confirm模式， 第1条消息  
 ConfirmReceiver1 Done! at 2018-08-04 14:58:27:0014  
 ConfirmReceiver1  :  Confirm模式， 第2条消息  
 ConfirmReceiver1 Done! at 2018-08-04 14:58:27:0016  
 ConfirmReceiver1  :  Confirm模式， 第3条消息  
 ConfirmReceiver1 Done! at 2018-08-04 14:58:27:0016  
 ConfirmReceiver1  :  Confirm模式， 第4条消息  
 ConfirmReceiver1 Done! at 2018-08-04 14:58:27:0017  
 ConfirmReceiver1  :  Confirm模式， 第5条消息  
 ConfirmReceiver1 Done! at 2018-08-04 14:58:27:0017

**2、批量Confirm模式**

```
channel.confirmSelect();
for(int i=0;i<5;i++){
     channel.basicPublish("", QUEUE_NAME, MessageProperties.PERSISTENT_BASIC, (" Confirm模式， 第" + (i + 1) + "条消息").getBytes());
}
if(channel.waitForConfirms()){
    System.out.println("发送成功");
}else{
    // 进行消息重发
}
```

这里主要更改代码为发送批量消息后再进行等待服务器确认，还可以调用channel.waitForConfirmsOrDie\(\)方法，该方法会等到最后一条消息得到确认或者得到nack才会结束，也就是说在waitForConfirmsOrDie处会造成当前程序的阻塞。更改代码为批量Confirm模式，运行我们查看控制台：

发送成功  
执行waitForConfirms耗费时间: 59ms

在WireShark查看信息如下：

可以发现这里处理的就是在批量发送信息完毕后，再进行ACK确认。同时我们发现这里只有三个Basic.Ack，这是因为Broker对信息进行了批量处理。

我们可以发现multiple的值为true，这与前面我们讲解的一致，true确认所有将比第一个参数指定的 delivery-tag 小的消息都得到确认。

我们也可以发现执行时间比第一种模式缩短了很多，效率极大提高了。

如果我们要对每条消息进行监听处理，可以通过在channel中添加监听器来实现，

```
 channel.addConfirmListener(new ConfirmListener() {
        @Override  
        public void handleNack(long deliveryTag, boolean multiple\) throws IOException {  
            System.out.println\("nack: deliveryTag = " + deliveryTag + " multiple: " + multiple\);  
        }

        @Override  
        public void handleAck(long deliveryTag, boolean multiple\) throws IOException {  
            System.out.println("ack: deliveryTag = " + deliveryTag + " multiple: " + multiple\);  
        }  
    }\);
```

当收到Broker发送过来的ack消息时就会调用handleAck方法，收到nack时就会调用handleNack方法。

我们可以在控制台看到信息，这次调用了两次Basic.Ack方法。

ack: deliveryTag = 4 multiple: true  
ack: deliveryTag = 5 multiple: false  
发送成功  
执行waitForConfirms耗费时间: 50ms

**3、异步Confirm模式**  
这里使用的异步Confirm模式，也要用到上面提到的监听，但是这里需要我们自己去维护实现一个waitForConfirms\(\)方法或waitForConfirmsOrDie\(\)，而waitForConfirms\(\)是同步的，因此我们需要自己去实现维护delivery-tag。

我们可以在jar中查看到源码，其实waitForConfirmsOrDie\(\)最终调用的也是waitForConfirms\(\)方法，在waitForConfirms\(\)方法内部维护了一个同步块代码，而unconfirmedSet就是存储delivery-tag标识的。

我们要实现自己异步调用，主要就是为了维护delivery-tag，主要实现代码如下：

```
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
for (int i = 0; i < 10; i++) {
    String message = String.format("时间 => %s", new Date().getTime());
    channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
}
//异步监听确认和未确认的消息
channel.addConfirmListener(new ConfirmListener() {
    @Override
    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
        System.out.println("未确认消息，标识：" + deliveryTag);
    }
    @Override
    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
        System.out.println(String.format("已确认消息，标识：%d，多个消息：%b", deliveryTag, multiple));
    }
});
```

### **二、消费者\(Consumer\)的Confirm模式**

1、手动确认和自动确认  
为了保证消息从队列可靠地到达消费者，RabbitMQ提供消息确认机制\(message acknowledgment\)。消费者在声明队列时，可以指定noAck参数，当noAck=false时，RabbitMQ会等待消费者显式发回ack信号后才从内存\(和磁盘，如果是持久化消息的话\)中移去消息。否则，RabbitMQ会在队列中消息被消费后立即删除它。

采用消息确认机制后，只要令noAck=false，消费者就有足够的时间处理消息\(任务\)，不用担心处理消息过程中消费者进程挂掉后消息丢失的问题，因为RabbitMQ会一直持有消息直到消费者显式调用basicAck为止。

在Consumer中Confirm模式中分为手动确认和自动确认。

手动确认主要并使用以下方法：

basic.ack: 用于肯定确认，multiple参数用于多个消息确认。  
basic.recover：是路由不成功的消息可以使用recovery重新发送到队列中。  
basic.reject：是接收端告诉服务器这个消息我拒绝接收,不处理,可以设置是否放回到队列中还是丢掉，而且只能一次拒绝一个消息,官网中有明确说明不能批量拒绝消息，为解决批量拒绝消息才有了basicNack。  
basic.nack：可以一次拒绝N条消息，客户端可以设置basicNack方法的multiple参数为true，服务器会拒绝指定了delivery\_tag的所有未确认的消息\(tag是一个64位的long值，最大值是9223372036854775807\)。

肯定的确认只是指导RabbitMQ将一个消息记录为已投递。basic.reject的否定确认具有相同的效果。 两者的差别在于：肯定的确认假设一个消息已经成功处理，而对立面则表示投递没有被处理，但仍然应该被删除。

同样的Consumer中的Confirm模式也具有同时确认多个投递，通过将确认方法的 multiple “字段设置为true完成的，实现的意义与Producer的一致。

在自动确认模式下，消息在发送后立即被认为是发送成功。 这种模式可以提高吞吐量（只要消费者能够跟上），不过会降低投递和消费者处理的安全性。 这种模式通常被称为“发后即忘”。 与手动确认模式不同，如果消费者的TCP连接或信道在成功投递之前关闭，该消息则会丢失。

使用自动确认模式时需要考虑的另一件事是消费者过载。 手动确认模式通常与有限的信道预取一起使用，限制信道上未完成（“进行中”）传送的数量。 然而，对于自动确认，根据定义没有这样的限制。 因此，消费者可能会被交付速度所压倒，可能积压在内存中，堆积如山，或者被操作系统终止。 某些客户端库将应用TCP反压（直到未处理的交付积压下降超过一定的限制时才停止从套接字读取）。 因此，只建议当消费者可以有效且稳定地处理投递时才使用自动投递方式。

主要实现代码：

// 手动确认消息  
channel.basicAck\(envelope.getDeliveryTag\(\), false\);

// 关闭自动确认  
boolean autoAck = false;  
channel.basicConsume\(QUEUE\_NAME, autoAck, consumer\);





