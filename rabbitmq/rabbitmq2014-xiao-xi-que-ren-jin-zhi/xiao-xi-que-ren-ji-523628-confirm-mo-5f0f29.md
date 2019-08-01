# 消息确认机制\(Confirm模式\)

在上一篇文章中我们讲解了RabbitMQ中的AMQP事务来保证消息发送到Broker端，同时我们可以在事务之间发送多条消息（即在channel.txSelect\(\)和channel.txCommit\(\)之间发送多条消息，通过使用事务来保证它们准确到达Broker），如果忘记了事务的使用，可以复习一下上一篇文章——[消息确认机制\(AMQP事务\)](/rabbitmq/rabbitmq2014-xiao-xi-que-ren-ji-523628-amqp-shi-52a129.md)

但是使用事务虽然可以保证消息的准确达到，但是它极大地牺牲了性能，因此我们为了性能上的要求，可以采用另一种高效的解决方案——通过使用Confirm模式来保证消息的准确性。

这里的Confirm模式可以分为两个方面来讲解，一是消息的生产者\(Producer\)的Confirm模式，另一个是消息的消费者\(Consumer\)的Confirm模式。

  
在生产者将信道设置成Confirm模式，一旦信道进入Confirm模式，所有在该信道上面发布的消息都会被指派一个唯一的ID\(以confirm.select为基础从1开始计数\)，一旦消息被投递到所有匹配的队列之后，Broker就会发送一个确认给生产者（包含消息的唯一ID）,这就使得生产者知道消息已经正确到达目的队列了，如果消息和队列是可持久化的，那么确认消息会将消息写入磁盘之后发出，Broker回传给生产者的确认消息中deliver-tag域包含了确认消息的序列号，此外Broker也可以设置basic.ack的multiple域，表示到这个序列号之前的所有消息都已经得到了处理。

Confirm模式最大的好处在于它是异步的，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用便可以通过回调方法来处理该确认消息，如果RabbitMQ因为自身内部错误导致消息丢失，就会发送一条basic.nack来代替basic.ack的消息，在这个情形下，basic.nack中各域值的含义与basic.ack中相应各域含义是相同的，同时requeue域的值应该被忽略。通过nack一条或多条消息， Broker表明自身无法对相应消息完成处理，并拒绝为这些消息的处理负责。在这种情况下，client可以选择将消息re-publish。

在channel 被设置成Confirm模式之后，所有被publish的后续消息都将被Confirm（即 ack）或者被nack一次。但是没有对消息被Confirm的快慢做任何保证，并且同一条消息不会既被Confirm又被nack。

开启confirm模式的方法  
生产者通过调用channel的confirmSelect方法将channel设置为Confirm模式，如果没有设置no-wait标志的话，Broker会返回confirm.select-ok表示同意发送者将当前channel信道设置为Confirm模式\(从目前RabbitMQ最新版本3.6来看，如果调用了channel.confirmSelect方法，默认情况下是直接将no-wait设置成false的，也就是默认情况下broker是必须回传confirm.select-ok的\)。

编程模式  
对于固定消息体大小和线程数，如果消息持久化，生产者Confirm\(或者采用事务机制\)，消费者ack那么对性能有很大的影响.

消息持久化的优化没有太好方法，用更好的物理存储（SAS, SSD, RAID卡）总会带来改善。生产者confirm这一环节的优化则主要在于客户端程序的优化之上。归纳起来，客户端实现生产者confirm有三种编程方式：

普通Confirm模式：每发送一条消息后，调用waitForConfirms\(\)方法，等待服务器端Confirm。实际上是一种串行Confirm了，每publish一条消息之后就等待服务端Confirm，如果服务端返回false或者超时时间内未返回，客户端进行消息重传；  
批量Confirm模式：批量Confirm模式，每发送一批消息之后，调用waitForConfirms\(\)方法，等待服务端Confirm，这种批量确认的模式极大的提高了Confirm效率，但是如果一旦出现Confirm返回false或者超时的情况，客户端需要将这一批次的消息全部重发，这会带来明显的重复消息，如果这种情况频繁发生的话，效率也会不升反降；  
异步Confirm模式：提供一个回调方法，服务端Confirm了一条或者多条消息后Client端会回调这个方法。

