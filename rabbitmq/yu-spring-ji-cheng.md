## springboot 2.0.3 自定义配置rabbitmq

**参考文档：**  
  
   rabbitmq官网教程：http://www.rabbitmq.com/getstarted.html  
   springboot官网教程：https://docs.spring.io/spring-amqp/docs/2.0.4.RELEASE/reference/html/
   
**WEB登录界面**

    http://192.168.111.103:15672/
    
**POM依赖**  

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

```

**RabbitConfig配置**

```
package com.master.rabbit;

import org.springframework.amqp.core.AmqpAdmin;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitAdmin;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author tuonioooo
 * @create 2016/9/25.
 * @blog https://blog.csdn.net/tuoni123
 */
@Configuration
public class RabbitConfig {

    @Bean
    public Queue myQueue() {
        return new Queue("rabbit-queue");
    }

    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory();
        cachingConnectionFactory.setHost("192.168.111.103");
        cachingConnectionFactory.setUsername("springboot");
        cachingConnectionFactory.setPassword("123456");
        cachingConnectionFactory.setPort(5672);
        return cachingConnectionFactory;
    }

    @Bean
    public AmqpAdmin amqpAdmin() {
        return new RabbitAdmin(connectionFactory());
    }

    @Bean
    public RabbitTemplate rabbitTemplate() {
        return new RabbitTemplate(connectionFactory());
    }
}
```

**接受者**
```
package com.master.rabbit;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

/**
 * @author tuonioooo
 * @create 2016/9/25.
 * @blog https://blog.csdn.net/tuoni123
 */
@Component
public class Receiver {

    @RabbitListener(queues = "rabbit-queue")
    @RabbitHandler
    public void process(String message) {
        System.out.println("Receiver : " + message);
    }

}
```

>@RabbitListener 提供队列声明 官网介绍：https://docs.spring.io/spring-amqp/docs/2.0.4.RELEASE/reference/html/_reference.html#async-annotation-driven
 @RabbitHandler  不同类型的消息使用不同的方法来处理。
 
**发送者**

```
@Component
public class Sender {

	@Autowired
	private RabbitTemplate rabbitTemplate;

	public void send() {
		String context = "hello " + new Date();
		System.out.println("Sender : " + context);
		this.rabbitTemplate.convertAndSend("rabbit-queue", context);
	}

}
```

**测试类**

```
package com.master;

import com.master.rabbit.RabbitConfig;
import com.master.rabbit.Sender;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootConfigRabbitmqApplicationTests {

	@Autowired
	private Sender sender;

	private final static String QUEUE_NAME = "rabbit-queue";

	@Test
	public void testRabbitMQ() throws Exception {
		sender.send();
	}


	@Test
	public void testUseApplicationContext(){

		ApplicationContext context =
				new AnnotationConfigApplicationContext(RabbitConfig.class);
		AmqpTemplate template = context.getBean(AmqpTemplate.class);
		template.convertAndSend("rabbit-queue", "foo");
		String foo = (String) template.receiveAndConvert("rabbit-queue");
	}

	@Test
	public void testcon() throws Exception{
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
	}
}
```



