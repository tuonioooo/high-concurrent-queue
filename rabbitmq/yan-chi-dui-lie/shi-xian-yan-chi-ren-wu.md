# 实现延迟任务

场景一：物联网系统经常会遇到向终端下发命令，如果命令一段时间没有应答，就需要设置成超时。

场景二：订单下单之后30分钟后，如果用户没有付钱，则系统自动取消订单、库存回滚。

上述类似的需求是我们经常会遇见的问题。最常用的方法是定期轮训数据库，设置状态。在数据量小的时候并没有什么大的问题，但是数据量一大轮训数据库的方式就会变得特别耗资源。当面对千万级、上亿级数据量时，本身写入的IO就比较高，导致长时间查询或者根本就查不出来，更别说分库分表以后了。除此之外，还有优先级队列，基于优先级队列的JDK延迟队列，时间轮等方式。但如果系统的架构中本身就有RabbitMQ的话，那么选择RabbitMQ来实现类似的功能也是一种选择。

使用RabbitMQ来实现延迟任务必须先了解RabbitMQ的两个概念：消息的TTL和死信Exchange，通过这两者的组合来实现上述需求。

## 消息的TTL（Time To Live）

消息的TTL就是消息的存活时间。RabbitMQ可以对队列和消息分别设置TTL。对队列设置就是队列没有消费者连着的保留时间，也可以对每一个单独的消息做单独的设置。超过了这个时间，我们认为这个消息就死了，称之为死信。如果队列设置了，消息也设置了，那么会取小的。所以一个消息如果被路由到不同的队列中，这个消息死亡的时间有可能不一样（不同的队列设置）。这里单讲单个消息的TTL，因为它才是实现延迟任务的关键。

可以通过设置消息的expiration字段或者x-message-ttl属性来设置时间，两者是一样的效果。只是expiration字段是字符串参数，所以要写个int类型的字符串：

```
byte[] messageBodyBytes = "Hello, world!".getBytes();
AMQP.BasicProperties properties = new AMQP.BasicProperties();
properties.setExpiration("60000");
channel.basicPublish("my-exchange", "routing-key", properties, messageBodyBytes);

```

当上面的消息扔到队列中后，过了60秒，如果没有被消费，它就死了。不会被消费者消费到。这个消息后面的，没有“死掉”的消息对顶上来，被消费者消费。死信在队列中并不会被删除和释放，它会被统计到队列的消息数中去。单靠死信还不能实现延迟任务，还要靠Dead Letter Exchange。

## Dead Letter Exchanges

Exchage的概念在这里就不在赘述，可以从[这里](http://www.cnblogs.com/haoxinyue/archive/2012/09/28/2707041.html)进行了解。一个消息在满足如下条件下，会进死信路由，记住这里是路由而不是队列，一个路由可以对应很多队列。

1. 一个消息被Consumer拒收了，并且reject方法的参数里requeue是false。也就是说不会被再次放在队列里，被其他消费者使用。

2. 上面的消息的TTL到了，消息过期了。

3. 队列的长度限制满了。排在前面的消息会被丢弃或者扔到死信路由上。

Dead Letter Exchange其实就是一种普通的exchange，和创建其他exchange没有两样。只是在某一个设置Dead Letter Exchange的队列中有消息过期了，会自动触发消息的转发，发送到Dead Letter Exchange中去。

## 实现延迟队列

延迟任务通过消息的TTL和Dead Letter Exchange来实现。我们需要建立2个队列，一个用于发送消息，一个用于消息过期后的转发目标队列。

![](/assets/15700-20170324221432018-1693098922.png)

生产者输出消息到Queue1，并且这个消息是设置有有效时间的，比如60s。消息会在Queue1中等待60s，如果没有消费者收掉的话，它就是被转发到Queue2，Queue2有消费者，收到，处理延迟任务。

具体实现步骤如下：

第一步， 首先需要创建2个队列。Queue1和Queue2。Queue1是一个消息缓冲队列，在这个队列里面实现消息的过期转发。如下图，设置Dead letter exchange和Dead letter routing key。设置这两个属性就是当消息在这个队列中expire后，采用哪个路由发送。这个dlx的exchange需要事先创建好，就是一个普通的exchange。由于我们还需要向Queue1发送消息，那么还需要创建一个exchange，并且和Queue1绑定。例子中，exchange同样取名：queue1。

![](/assets/15700-20170324221433705-1183004086.png)

我们还需要建一个Queue2，这个队列用于消息在Queue1中过期后转发的目标队列。所以这个Queue2队列建好以后，需要绑定Queue1设置的死信路由：dlx。完成Queue2的绑定以后，环境就搭建完成了。

![](/assets/15700-20170324221435221-754012683.png)

第二步，实现消息的Producer。由于我们的目的是让进入Queue1的消息过期，然后自动转送到Queue2中，所以发送的时候，需要设置过期时间。

```
ConnectionFactory factory = new ConnectionFactory();
            factory.setUsername("bsp");
            factory.setPassword("123456");
            factory.setVirtualHost("/");
            factory.setHost("10.23.22.42");
            factory.setPort(5672);
 
            conn = factory.newConnection();
            channel = conn.createChannel();
 
            byte[] messageBodyBytes = "Hello, world!".getBytes();
 
            byte i = 10;
            while (i-- > 0) {                
                channel.basicPublish("queue1", "queue1", new AMQP.BasicProperties.Builder().expiration(String.valueOf(i * 1000)).build(),
                        new byte[] { i });
 
            }

```

上面的代码我模拟了1-10号消息，消息的内容里面是1-10。过期的时间是10-1秒。这里要注意，虽然10是第一个发送，但是它过期的时间最长。

第三步，实现消息的Consumer。Consumer就是延迟任务的具体实施者。由于具体的任务往往是一个比较耗时的任务，所以一般来说，任务一般在异步线程中执行。

```
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername("bsp");
factory.setPassword("123456");
factory.setVirtualHost("/");
factory.setHost("10.23.22.42");
factory.setPort(5672);
 
conn = factory.newConnection();
channel = conn.createChannel();
 
channel.basicConsume("queue2", true, "consumer", new DefaultConsumer(channel) {
@Override
public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,byte[] body) throws IOException {
 
long deliveryTag = envelope.getDeliveryTag();
                    
//do some work async
System.out.println(body[0]);
}
});

```

运行后如上面的程序，过了10s以后，消费者开始收到数据，但是它是一次性收到如下结果：

10、9 、8 、7 、6、5 、4 、3 、2 、1

Consumer第一个收到的还是10。虽然10是第一个放进队列，但是它的过期时间最长。所以由此可见，即使一个消息比在同一队列中的其他消息提前过期，提前过期的也不会优先进入死信队列，它们还是按照入库的顺序让消费者消费。如果第一进去的消息过期时间是1小时，那么死信队列的消费者也许等1小时才能收到第一个消息。参考官方文档发现“Only when expired messages reach the head of a queue will they actually be discarded \(or dead-lettered\).”只有当过期的消息到了队列的顶端（队首），才会被真正的丢弃或者进入死信队列。

所以在考虑使用RabbitMQ来实现延迟任务队列的时候，需要确保业务上每个任务的延迟时间是一致的。如果遇到不同的任务类型需要不同的延时的话，需要为每一种不同延迟时间的消息建立单独的消息队列。

总结：

延迟队列存储的对象肯定是对应的延时消息，所谓”延时消息”是指当消息被发送以后，并不想让消费者立即拿到消息，而是等待指定时间后，消费者才拿到这个消息进行消费。

  


        上面的Queue1队列是不设置消费者的，过了有效期后消息会路由放入Queue2队列，此时Queue2的消费者拿到消息进行处理。针对上面的场景二：“订单下单之后30分钟后，如果用户没有付钱，则系统自动取消订单、库存回滚。”问题，可以在Queue2的消费者拿到订单信息后，先判断订单是否已经支付，若是已经支付，直接将消息消费掉就可以了，否则执行订单状态修改取消、库存回滚等一系列操作，然后再将消息消费掉。



