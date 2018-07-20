# 原生Java客户端使用

## RabbitMQ-Java客户端API指南-上 {#rabbitmq-java客户端api指南-上}

客户端API严格按照AMQP 0-9-1协议规范进行建模，并提供了易于使用的附加抽象。  
RabbitMQ Java客户端使用com.rabbitmq.client作为其顶层包。关键的类和接口是：

* Channel
* Connection
* ConnectionFactory
* Consumer

协议操作可通过Channel接口获得。Connection用于打开通道，注册连接生命周期事件处理程序，并关闭不再需要的连接。 连接是通过ConnectionFactory实例化的，这就是你如何配置各种连接设置，如虚拟主机或用户名。

## Connections和Channels {#connections和channels}

```
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
```

## 连接到Broker {#连接到broker}

以下代码使用给定的参数（主机名，端口号等）连接到AMQP代理：

```
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(userName);
factory.setPassword(password);
factory.setVirtualHost(virtualHost);
factory.setHost(hostName);
factory.setPort(portNumber);
Connection conn = factory.newConnection();
```

所有这些参数都对本地运行的RabbitMQ服务器具有合理的默认值。

或者，可以使用URI：

```
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://userName:password@hostName:portNumber/virtualHost");
Connection conn = factory.newConnection();
```

所有这些参数都对本地运行的RabbitMQ服务器有合理的默认值。

然后接口可以用于打开一个通道：

```
Channel channel = conn.createChannel();
```

现在可以使用该通道发送和接收消息，如后面的部分所述。

要断开连接，只需关闭通道和连接：

```
channel.close();
conn.close();
```

## 使用Exchanges和Queues {#使用exchanges和queues}

客户端应用程序与AMQP的高级构建块交换和排队。这些必须被“声明”才可以使用。声明任何一种类型的对象只是确保其中一个名称存在，如果有必要的话创建它。

继续前面的例子，下面的代码声明一个Exchange和一个Queue，然后将它们绑定在一起。

```
channel.exchangeDeclare(exchangeName, "direct", true);
String queueName = channel.queueDeclare().getQueue();
channel.queueBind(queueName, exchangeName, routingKey);
```

这将主动声明以下对象，这两个对象都可以使用附加参数进行定制。这里他们都没有任何特别的论点。

* 一个耐用，非自动删除“直接”类型的交换
* 一个具有生成名称的非持久，独占，自动删除队列

上面的函数调用然后用给定的路由密钥将队列绑定到交换机上。

请注意，这将是一个典型的方式来声明一个队列，当只有一个客户端想要使用它：它不需要一个众所周知的名称，没有其他客户端可以使用它（独占），将自动清理（自动删除）。如果有几个客户想共享一个知名名字的队列，这个代码将是合适的：

```
channel.exchangeDeclare(exchangeName, "direct", true);
channel.queueDeclare(queueName, true, false, false, null);
channel.queueBind(queueName, exchangeName, routingKey);
```

这将主动宣布：

* 一个耐用，非自动删除“直接”类型的交换
* 一个具有众所周知名称的持久的，非排他性的非自动删除队列

请注意，所有这些Channel API方法都被重载。这些便捷的exchangeDeclare，queueDeclare和queueBind 使用合理的默认值。还有更多的参数更多的形式，让你根据需要重写这些默认值，在需要的地方给予完全控制。

这个“短形式，长形式”模式在整个客户端API使用。

示例代码：

```
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("192.168.111.103");
factory.setUsername("springboot");
factory.setPassword("123456");
factory.setPort(5672);
Connection connection = factory.newConnection();
Channel channel = connection.createChannel();
channel.queueDeclare(QUEUE_NAME, false, false, false, null);
String message = "Hello World!";
channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
channel.close();
connection.close();
```





