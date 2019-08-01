# 消费端限流、TTL、死信队列

## 消费端限流 {#消费端限流}

### 1. 为什么要对消费端限流 {#为什么要对消费端限流}

假设一个场景，首先，我们 Rabbitmq 服务器积压了有上万条未处理的消息，我们随便打开一个消费者客户端，会出现这样情况: 巨量的消息瞬间全部推送过来，但是我们单个客户端无法同时处理这么多数据!

当数据量特别大的时候，我们对生产端限流肯定是不科学的，因为有时候并发量就是特别大，有时候并发量又特别少，我们无法约束生产端，这是用户的行为。所以我们应该对消费端限流，用于保持消费端的稳定，当消息数量激增的时候很有可能造成资源耗尽，以及影响服务的性能，导致系统的卡顿甚至直接崩溃。

### 2.限流的 api 讲解 {#限流的-api-讲解}

RabbitMQ 提供了一种 qos （服务质量保证）功能，即在非自动确认消息的前提下，如果一定数目的消息（通过基于 consume 或者 channel 设置 Qos 的值）未被确认前，不进行消费新的消息。

```
/**
* Request specific "quality of service" settings.
* These settings impose limits on the amount of data the server
* will deliver to consumers before requiring acknowledgements.
* Thus they provide a means of consumer-initiated flow control.
* @param prefetchSize maximum amount of content (measured in
* octets) that the server will deliver, 0 if unlimited
* @param prefetchCount maximum number of messages that the server
* will deliver, 0 if unlimited
* @param global true if the settings should be applied to the
* entire channel rather than each consumer
* @throws java.io.IOException if an error is encountered
*/
void basicQos(int prefetchSize, int prefetchCount, boolean global) throws IOException;

```

* prefetchSize：0，单条消息大小限制，0代表不限制

* prefetchCount：一次性消费的消息数量。会告诉 RabbitMQ 不要同时给一个消费者推送多于 N 个消息，即一旦有 N 个消息还没有 ack，则该 consumer 将 block 掉，直到有消息 ack。

* global：true、false 是否将上面设置应用于 channel，简单点说，就是上面限制是 channel 级别的还是 consumer 级别。当我们设置为 false 的时候生效，设置为 true 的时候没有了限流功能，因为 channel 级别尚未实现。
* 注意：prefetchSize 和 global 这两项，rabbitmq 没有实现，暂且不研究。特别注意一点，prefetchCount 在 no\_ask=false 的情况下才生效，即在自动应答的情况下这两个值是不生效的。



