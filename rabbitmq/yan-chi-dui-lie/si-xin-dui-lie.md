# 死信队列

死信队列：没有被及时消费的消息存放的队列

消息没有被及时消费的原因：

* **消息被拒绝（basic.reject/ basic.nack）并且不再重新投递 requeue=false**

* **TTL\(time-to-live\) 消息超时未消费**

* **达到最大队列长度**





