# direct、topic、fanout的使用以及区别

在rabbitmq中有许多交换机，不同的交换机适用于不同的场景。如下：

![](/assets/20190605150355477.png)

这么多交换机中，最常用的交换机有三种：direct、topic、fanout。我分别叫他们：“直接连接交换机”，“主题路由匹配交换机”，“无路由交换机”。以下是详细的介绍：

Direct 交换机

这个交换机就是一个直接连接交换机，什么叫做直接连接交换机呢？

所谓“直接连接交换机”就是：Producer\(生产者\)投递的消息被DirectExchange \(交换机\)转发到通过routingkey绑定到具体的某个Queue\(队列\)，把消息放入队列,然后Consumer从Queue中订阅消息。

##### 消费端：

```
package com.wy.testrabbitmq.testdirect;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.QueueingConsumer;

/**
* @author wangyan@163.com
* @version 1.0
* @date 2019-06-05 10:56
*/
public class Consumer {
   public static void main(String[] args) throws Exception {
       //创建连接工厂
       ConnectionFactory connectionFactory = new ConnectionFactory();
       connectionFactory.setHost("127.0.0.1");
       connectionFactory.setPort(5672);
       connectionFactory.setVirtualHost("/");
       //创建连接
       Connection connection = connectionFactory.newConnection();
       // 创建通道
       Channel channel = connection.createChannel();
       //交换机名
       String exchangeName = "testDirectExchange";
       //队列
       String queueName = "test002";
       //routingkey
       String routingkey = "testDirect";
       //交换机类型
       String exchangeType = "direct";
       /**
        * 声明一个交换机
        */
       channel.exchangeDeclare(exchangeName,exchangeType,true,false,false,null);
       /**
        * 声明一个队列
        * 第一个参数表示这个信道连接哪个队列
        * 第二个参数表示是否持久化，当这个参数设置为true，即使你的服务器关了从新开数据还是存在的
        * 第三个参数表示是否独占队列，也就是所只能自己去监听这个队列
        * 第四个参数表示队列脱离绑定时是否自动删除
        * 第五个参数表示扩展参数，可设置为null
        */
       channel.queueDeclare(queueName, true, false, false, null);

       //建立绑定关系(哪个队列，哪个交换机，绑定哪个routingkey)
       channel.queueBind(queueName, exchangeName, routingkey);

       //创建消费者，指定建立在那个连接上
       QueueingConsumer queueingConsumer = new QueueingConsumer(channel);
       //设置channel
       // 第二个参数 是否自动签收
       // 第三个参数表示消费对象
       channel.basicConsume(queueName, true, queueingConsumer);
       //获取消息
       //int i=0;
       while (true) {
           // 没有消息就阻塞
           QueueingConsumer.Delivery delivery = queueingConsumer.nextDelivery();
           String message = new String(delivery.getBody());
           System.out.println("消费端接收消息：" + message);
       }
   }
}

```

##### 生产端：

```
package com.wy.testrabbitmq.testdirect;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

/**Direct交换机
* @author wangyan@163.com
* @version 1.0
* @date 2019-06-05 10:16
*/
public class Producer {
   public static void main(String[] args) throws Exception {
       //创建连接工厂
       ConnectionFactory connectionFactory = new ConnectionFactory();
       connectionFactory.setHost("127.0.0.1");
       connectionFactory.setPort(5672);
       connectionFactory.setVirtualHost("/");
       //创建连接
       Connection connection = connectionFactory.newConnection();
       // 创建通道
       Channel channel=connection.createChannel();
      //声明交换机
       String exchangeName="testDirectExchange";
        // routingkey
       String routingkey="testDirect";
       // 发送消息
       String message="这是一条测试direct的数据";
       channel.basicPublish(exchangeName,routingkey,null,message.getBytes());
       //注意：关闭连接
       channel.close();
       connection.close();
   }
}

```

我启动生产者和消费者，可以看到服务已经被消费。如下图：

![](/assets/20190605155958644.png)

在管控台Exchange中可以看到多了一个交换机：

![](/assets/20190605160247319.png)

点击testDirectExchange中Bindings可以看到我们的Routingkey：testDirect和绑定的队列test002

![](/assets/20190605160633629.png)

点击test002可以快速进入到队列中，点击binding可以查看到队列绑定的交换机。

![](/assets/20190605160124859.png)

想一下，是不是：生产者发送消息到DirectExchange交换机，交换机根据routingkey转发消息到绑定的Queue，供消费者消费。

#### Topic 交换机

举个现实生活中的栗子：

假如你想在淘宝上买一双运动鞋，那么你是不是会在搜索框中搜“XXX运动鞋”，这个时候系统将会模糊匹配的所有符合要求的运动鞋，然后展示给你。

所谓“主题路由匹配交换机”也是这样一个道理，但是使用时也有一定的规则。

比如：

```
String routingkey = “testTopic.#”;
String routingkey = “testTopic.*”;
```

\*表示只匹配一个词

* \#表示匹配多个词



