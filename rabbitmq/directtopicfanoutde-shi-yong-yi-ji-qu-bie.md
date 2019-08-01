# direct、topic、fanout的使用以及区别

在rabbitmq中有许多交换机，不同的交换机适用于不同的场景。如下：

![](/assets/20190605150355477.png)

这么多交换机中，最常用的交换机有三种：direct、topic、fanout。我分别叫他们：“直接连接交换机”，“主题路由匹配交换机”，“无路由交换机”。以下是详细的介绍：

Direct 交换机



这个交换机就是一个直接连接交换机，什么叫做直接连接交换机呢？



所谓“直接连接交换机”就是：Producer\(生产者\)投递的消息被DirectExchange \(交换机\)转发到通过routingkey绑定到具体的某个Queue\(队列\)，把消息放入队列,然后Consumer从Queue中订阅消息。







