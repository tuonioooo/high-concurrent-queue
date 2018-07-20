

## SpringBoot集成activemq

### 参考文档：

[SpringBoot ActiveMQ Support 官网](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/htmlsingle/#boot-features-activemq)

[Activemq 官网](http://activemq.apache.org)


### 添加必要依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
```
> 提供连接或嵌入ActiveMQ实例的必要依赖项，以及与JMS集成的Spring基础结构

### application.properties配置

**ActiveMQ基础配置**

```
spring.activemq.broker-url=tcp://localhost:8161
spring.activemq.user = admin
spring.activemq.password = admin
spring.activemq.in-memory=true
```
**ActiveMQ配置JMS连接池**

```
spring.activemq.pool.enabled = true #需要添加activemq-pool依赖，false则不需要
spring.activemq.pool.max-connections = 50
```
>org.apache.activemq:activemq-pool并对其进行配置来池化JMS资源 PooledConnectionFactory
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-pool</artifactId>
    <version>5.15.4</version>
</dependency>

### 配置生产者和消费者

**生产者**

```
package com.master.activemq;
 
import javax.jms.Destination;
 
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jms.core.JmsMessagingTemplate;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.stereotype.Service;

 /**
 *  @Author tuonioooo
 *  @Date 2018-7-20 11:15
 *  @Info  
 *  @Blog https://blog.csdn.net/tuoni123
 */
@Service("producer")
public class Producer {
	@Autowired // 也可以注入JmsTemplate，JmsMessagingTemplate对JmsTemplate进行了封装
	private JmsMessagingTemplate jmsTemplate;

	@Autowired
	private JmsTemplate jmsTemplate2;
	// 发送消息，destination是发送到的队列，message是待发送的消息
	public void sendMessage(Destination destination, final String message){
		jmsTemplate.convertAndSend(destination, message);
	}
}

```

**消费者**

```
package com.master.activemq;

import org.springframework.jms.annotation.JmsListener;
import org.springframework.stereotype.Component;

/**
*  @Author tuonioooo
*  @Date 2018-7-20 11:15
*  @Info
*  @Blog https://blog.csdn.net/tuoni123
*/
@Component
public class Consumer {

    // 使用JmsListener配置消费者监听的队列，其中text是接收到的消息
    @JmsListener(destination = "mytest.queue")
    public void receiveQueue(String text) {
        System.out.println("Consumer收到的报文为:" + text);
    }
}

```

### 测试用例

```
package com.master;

import com.master.activemq.Producer;
import org.apache.activemq.command.ActiveMQQueue;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import javax.jms.Destination;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootActivemqApplicationTests {

	@Autowired
	private Producer producer;

	@Test
	public void contextLoads() throws InterruptedException {
		Destination destination = new ActiveMQQueue("mytest.queue");

		for(int i=0; i<100; i++){
			producer.sendMessage(destination, "myname is chhliu!!!");
		}
	}

}

```

代码示例： [https://github.com/tuonioooo?tab=repositories](https://github.com/tuonioooo?tab=repositories)

