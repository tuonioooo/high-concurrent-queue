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

