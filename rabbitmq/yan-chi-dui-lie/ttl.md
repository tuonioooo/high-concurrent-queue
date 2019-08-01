# TTL

TTL是Time To Live的缩写，也就是生存时间。RabbitMQ支持消息的过期时间，在消息发送时可以进行指定。  
RabbitMQ支持队列的过期时间，从消息入队列开始计算，只要超过了队列的超时时间配置，那么消息会自动的清除。

这与 Redis 中的过期时间概念类似。我们应该合理使用 TTL 技术，可以有效的处理过期垃圾消息，从而降低服务器的负载，最大化的发挥服务器的性能。

> RabbitMQ allows you to set TTL \(time to live\) for both messages and queues. This can be done using optional queue arguments or policies \(the latter option is recommended\). Message TTL can be enforced for a single queue, a group of queues or applied for individual messages.
>
> RabbitMQ允许您为消息和队列设置TTL（生存时间）。 这可以使用可选的队列参数或策略来完成（建议使用后一个选项）。 可以对单个队列，一组队列强制执行消息TTL，也可以为单个消息应用消息TTL。
>
> ​ ——摘自 RabbitMQ 官方文档

### 1.消息的 TTL {#消息的-ttl}

我们在生产端发送消息的时候可以在 properties 中指定 `expiration`属性来对消息过期时间进行设置，单位为毫秒\(ms\)。

```
     /**
         * deliverMode 设置为 2 的时候代表持久化消息
         * expiration 意思是设置消息的有效期，超过10秒没有被消费者接收后会被自动删除
         * headers 自定义的一些属性
         * */
        //5. 发送
        Map<String, Object> headers = new HashMap<String, Object>();
        headers.put("myhead1", "111");
        headers.put("myhead2", "222");
 
        AMQP.BasicProperties properties = new AMQP.BasicProperties().builder()
                .deliveryMode(2)
                .contentEncoding("UTF-8")
                .expiration("100000")
                .headers(headers)
                .build();
        String msg = "test message";
        channel.basicPublish("", queueName, properties, msg.getBytes());

```

我们也可以后台管理页面中进入 Exchange 发送消息指定`expiration`

![](/assets/1543774-20190522121234090-1048197586.png)

### 2.队列的 TTL {#队列的-ttl}

我们也可以在后台管理界面中新增一个 queue，创建时可以设置 ttl，对于队列中超过该时间的消息将会被移除。

![](/assets/1543774-20190522121311523-815458531.png)





