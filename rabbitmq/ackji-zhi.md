# ack机制

## 前言

Consumer可能需要一段时间才能处理完收到的数据。如果在这个过程中，Consumer出错了，异常退出了，而数据还没有处理完成，这段数据就丢失了。如果我们采用no-ack的方式进行确认，也就是说，每次Consumer接到数据后，而不管是否处理完成，RabbitMQ Server会立即把这个Message标记为完成，然后从queue中删除了。

为了保证数据不被丢失，RabbitMQ支持消息确认机制，即ack。为了保证数据能被正确处理而不仅仅是被Consumer收到，我们就不能采用no-ack或者auto-ack，我们需要手动ack\(manual-ack\)。在数据处理完成后手动发送ack，这个时候Server才将Message删除。

## 如何设置

* ## action in java

```
# Channel.java，将autoAck设置为false即可。
String basicConsume(String queue, boolean autoAck, Consumer callback);
```

* ##### action in spring {#action-in-spring}

```
<rabbit:listener-container connection-factory="connectionFactory" acknowledge="manual">
    <rabbit:listener ref="myConsumer" queues="myQueue"/>
</rabbit:listener-container>
```

## 忘记ack

只能说后果很严重，因为没有ack的话，RabbitMQ Server会重新分发，并且RabbitMQ Server不会再发送数据给它（直至Server收到ack才会再次发送消息，参看测试用例），因为Server认为这个Consumer处理能力有限。这样就导致消息在RabbitMQ Server上堆积，最终造成内存泄露。

## 如何排查

可以使用以下命令查看没有被ACK的消息，如果有大量消息，那八九不离十就是忘记ack了。

```
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

## ACK其他作用

ACK机制可以起到限流的作用，比如在消费者处理后，sleep一段时间，然后再ACK，这可以帮助消费者负载均衡。

当然，除了ACK，有两个比较重要的参数也在控制着consumer的load-balance，即prefetch和concurrency

##### prefetch & concurrency {#prefetch-amp-concurrency}

1. prefetch: 每次从一次性从broker里面取的待消费的消息的个数。prefetch可以改变RabbitMQ的循环分发机制，比如设置其值为1，当消费者在处理消息完成，还没有ACK的时候，RabbitMQ的Server不会再发消息给该消费者。
2. concurrency: 这个表示每个listener创建多少个消费者（会创建多少个线程来消费）

> 1. 首先，basic.qos是针对channel进行设置的，也就是说只有在channel建立之后才能发送basic.qos信令。
> 2. 其实basic.qos里还有另外两个参数可进行设置（global和prefetch\_size），但RabbitMQ没有相应的实现。
> 3. RabbitMQ如何挑选消费者？当RabbitMQ要将队列中的一条消息投递给消费者时，会遍历该队列上的消费者列表，选一个合适的消费者，然后将消息投递出去。其中挑选消费者的一个依据就是看消费者对应的channel上未ack的消息数是否达到设置的prefetch\_count个数，如果未ack的消息数达到了prefetch\_count的个数，则不符合要求。

## ACK测试

* ##### 没有ACK {#case1：没有ACK}

```
Channel channel = connection.createChannel();
// 消费者
DefaultConsumer consumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println("1号消费者 ===> " + message);
        //getChannel().basicAck(envelope.getDeliveryTag(), false);
    }
};
channel.basicConsume(QUEUE_NAME, false, null, consumer);
```

> 结论：没有ACK，消费者任然会正常消费，消费者断开重连后，RabbitMQ Server会重新将消息发送给消费者，消费者重新执行任务，这可能导致**\#重复消费\#**

* ##### 两个消费者，一个正常ACK，一个不ACK {#case2：两个消费者，一个正常ACK，一个不ACK}

```
Channel channel = connection.createChannel();
// 消费者
DefaultConsumer consumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println("1号消费者 ===> " + message);
        // Thread.sleep(2000); // 延时ACK
        getChannel().basicAck(envelope.getDeliveryTag(), false);
    }
};
channel.basicConsume(QUEUE_NAME, false, null, consumer);
// 消费者
DefaultConsumer anotherConsumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println("2号消费者 ===> " + message);
        //getChannel().basicAck(envelope.getDeliveryTag(), false);
    }
};
channel.basicConsume(QUEUE_NAME, false, null, anotherConsumer);
```

> 结论：测试发现，两个消费者交替消费（**循环分发机制**）。
>
> 1. 重启2号消费者，这个时候RabbitMQ会将其消费过的消息重新发给1号消费者消费。
> 2. 即使没有ACK，RabbitMQ也不是立即不发送消息到没有ack的consumer，其实这很好理解，既然ack可以延时，那Server完全有理由相信consumer回复只是比较慢而已，它不是不回。所以继续会发第二条。
> 3. 本来以为是2号消费者重启过程中，1号消费者消费过快，等2号消费者起起来时，消息已经被1号消费者消费完，其实不是，就算给1号消费者加上延迟ACK，2号消费者也不会再接收消息。应该是RabbitMQ内部的某种记忆功能，将2号消费者没有ACK的消息，直接归给1号消费者消费。
> 4. 再次执行生产者，此时2号消费者任然可以重新接收消息。
> 5. 设置1号消费者prefetch=1\(`channel.basicQos(1)`\)，2号消费者不做任何设置，然后两个消费者都订阅同一队列，开启acknowledge机制。RabbitMQ向1号消费者投递了一条消息后，消费者未对该消息进行ack，RabbitMQ不会再向该消费者投递消息，剩下的消息均投递给了2号消费者。这和第二个试验结果不同。

### 补充 {#补充}

> 我们知道，有了ACK机制，当消费者挂掉后，消息可以不丢失，但是如果RabbitMQ Server挂掉了呢？这就需要持久化机制。如果没有设置相应的持久化，会在Server退出后丢掉Exchange和Queue。



