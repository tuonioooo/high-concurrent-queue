# 死信队列

死信队列：没有被及时消费的消息存放的队列

消息没有被及时消费的原因：

* **消息被拒绝（basic.reject/ basic.nack）并且不再重新投递 requeue=false**

* **TTL\(time-to-live\) 消息超时未消费**

* **达到最大队列长度**

### 实现死信队列步骤 {#实现死信队列步骤}

首先需要设置死信队列的 exchange 和 queue，然后进行绑定:

```
Exchange: dlx.exchange
Queue: dlx.queue
RoutingKey: # 代表接收所有路由 key
```

然后我们进行正常声明交换机、队列、绑定，只不过我们需要在普通队列加上一个参数即可: arguments.put\("x-dead-letter-exchange",' dlx.exchange' \)

这样消息在过期、requeue失败、 队列在达到最大长度时，消息就可以直接路由到死信队列!

```
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
public class DlxProducer {
    public static void main(String[] args) throws Exception {
                //设置连接以及创建 channel 湖绿
        String exchangeName = "test_dlx_exchange";
        String routingKey = "item.update";
      
        String msg = "this is dlx msg";
 
        //我们设置消息过期时间，10秒后再消费 让消息进入死信队列
        AMQP.BasicProperties properties = new AMQP.BasicProperties().builder()
                .deliveryMode(2)
                .expiration("10000")
                .build();
 
        channel.basicPublish(exchangeName, routingKey, true, properties, msg.getBytes());
        System.out.println("Send message : " + msg);
 
        channel.close();
        connection.close();
    }
}

```

```
import com.rabbitmq.client.*;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
 
public class DlxConsumer {
    public static void main(String[] args) throws Exception {
                //创建连接、创建channel忽略 内容可以在上面代码中获取
        String exchangeName = "test_dlx_exchange";
        String queueName = "test_dlx_queue";
        String routingKey = "item.#";
 
        //必须设置参数到 arguments 中
        Map<String, Object> arguments = new HashMap<String, Object>();
        arguments.put("x-dead-letter-exchange", "dlx.exchange");
 
        channel.exchangeDeclare(exchangeName, "topic", true, false, null);
        //将 arguments 放入队列的声明中
        channel.queueDeclare(queueName, true, false, false, arguments);
 
        //一般不用代码绑定，在管理界面手动绑定
        channel.queueBind(queueName, exchangeName, routingKey);
 
 
        //声明死信队列
        channel.exchangeDeclare("dlx.exchange", "topic", true, false, null);
        channel.queueDeclare("dlx.queue", true, false, false, null);
        //路由键为 # 代表可以路由到所有消息
        channel.queueBind("dlx.queue", "dlx.exchange", "#");
 
        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body)
                    throws IOException {
 
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + message + "'");
 
            }
        };
 
        //6. 设置 Channel 消费者绑定队列
        channel.basicConsume(queueName, true, consumer);
    }
}

```

### 总结 {#总结}

DLX也是一个正常的 Exchange，和一般的 Exchange 没有区别，它能在任何的队列上被指定，实际上就是设置某个队列的属性。当这个队列中有死信时，RabbitMQ 就会自动的将这个消息重新发布到设置的 Exchange 上去，进而被路由到另一个队列。可以监听这个队列中消息做相应的处理。

