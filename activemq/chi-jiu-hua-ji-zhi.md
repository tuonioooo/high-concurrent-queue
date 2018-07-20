# 持久化机制

### 背景

场景问题：服务器断电重启，未被消费的消息是否会在重启之后被继续消费？ 两种选择：非持久性模式/持久性模式

**非持久性模式：**

`服务器断电(关闭)之后，使用非持久性模式时，没有被消费的消息不会继续消费全部丢失；` 程序会报一个连接关闭异常停止运行，继续启动服务器运行程序，不会接收任何消息。

**持久性模式：**

`服务器断电(关闭)后，使用持久性模式时，没有被消费的消息会继续消费；` 程序也会报连接关闭异常，但再次启动服务器和程序后，接收方还能继续原来的消息再次接收。

### ActiveMQ的几种存储模式选择

* AMQ消息存储 \(默认的消息存储\)

* KahaDB 消息存储\(提供容量的提升和恢复能力\) 基于文件支持事务的消息存储器，是一个可靠，高性能，可扩展的消息存储器

* JDBC 消息存储\(消息基于JDBC存储\)

* 使用Mysql或Oracle数据库存储消息

* Memory 消息存储\(基于内容的消息存储\)  ActiveMQ支持将消息保存到内存中，将Broker的“prsistent” 属性设置为“false”。

### 持久化操作

* #### 持久化文件

ActiveMQ默认就支持这种方式，只要在发消息时设置消息为持久化就可以了。

打开安装目录下的配置文件：

D:\ActiveMQ\apache-activemq\conf\activemq.xml在越80行会发现默认的配置项：

&lt;persistenceAdapter&gt;

&lt;kahaDB directory="${activemq.data}/kahadb"/&gt;

&lt;/persistenceAdapter&gt;

注意这里使用的是kahaDB，是一个基于文件支持事务的消息存储器，是一个可靠，高性能，可扩展的消息存储器。

```
 他的设计初衷就是使用简单并尽可能的快。KahaDB的索引使用一个transaction log，并且所有的destination只使用一个index，有人测试表明：如果用于生产环境，支持1万个active connection，每个connection有一个独立的queue。该表现已经足矣应付大部分的需求。
```

然后再发送消息的时候改变第二个参数为：

MsgDeliveryMode.Persistent

Message保存方式有2种  
PERSISTENT：保存到磁盘，consumer消费之后，message被删除。  
NON\_PERSISTENT：保存到内存，消费之后message被清除。  
注意：堆积的消息太多可能导致内存溢出。

然后打开生产者端发送一个消息：

[![](https://images0.cnblogs.com/blog/16765/201411/272212018567693.png "wps30F4.tmp")](https://images0.cnblogs.com/blog/16765/201411/272212014189293.png)

不启动消费者端，同时在管理界面查看：

[![](https://images0.cnblogs.com/blog/16765/201411/272212026214592.png "wps3105.tmp")](https://images0.cnblogs.com/blog/16765/201411/272212022316879.png)

发现有一个消息正在等待，这时如果没有持久化，ActiveMQ宕机后重启这个消息就是丢失，而我们现在修改为文件持久化，重启ActiveMQ后消费者仍然能够收到这个消息。

[![](https://images0.cnblogs.com/blog/16765/201411/272212033876192.png "wps3106.tmp")](https://images0.cnblogs.com/blog/16765/201411/272212029963778.png)

* #### 持久化为数据库

我们从支持Mysql为例，将：

* commons-dbcp-1.4.jar
* commons-pool-1.6.jar
* mysql-connector-java-5.1.30.jar

D:\ActiveMQ\apache-activemq\lib目录下。

打开并修改配置文件：

```
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd">

    <!-- Allows us to use system properties as variables in this configuration file -->
    <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <value>file:${activemq.conf}/credentials.properties</value>
        </property>
    </bean>

   <!-- Allows accessing the server log -->
    <bean id="logQuery" class="org.fusesource.insight.log.log4j.Log4jLogQuery"
          lazy-init="false" scope="singleton"
          init-method="start" destroy-method="stop">
    </bean>

    <!--
        The <broker> element is used to configure the ActiveMQ broker.
    -->
    <broker xmlns="http://activemq.apache.org/schema/core" brokerName="localhost" dataDirectory="${activemq.data}">

        <destinationPolicy>
            <policyMap>
              <policyEntries>
                <policyEntry topic=">" >
                    <!-- The constantPendingMessageLimitStrategy is used to prevent
                         slow topic consumers to block producers and affect other consumers
                         by limiting the number of messages that are retained
                         For more information, see:

                         http://activemq.apache.org/slow-consumer-handling.html

                    -->
                  <pendingMessageLimitStrategy>
                    <constantPendingMessageLimitStrategy limit="1000"/>
                  </pendingMessageLimitStrategy>
                </policyEntry>
              </policyEntries>
            </policyMap>
        </destinationPolicy>


        <!--
            The managementContext is used to configure how ActiveMQ is exposed in
            JMX. By default, ActiveMQ uses the MBean server that is started by
            the JVM. For more information, see:

            http://activemq.apache.org/jmx.html
        -->
        <managementContext>
            <managementContext createConnector="false"/>
        </managementContext>

        <!--
            Configure message persistence for the broker. The default persistence
            mechanism is the KahaDB store (identified by the kahaDB tag).
            For more information, see:

            http://activemq.apache.org/persistence.html
            <kahaDB directory="${activemq.data}/kahadb"/>
        -->
        <persistenceAdapter>
            <jdbcPersistenceAdapter dataDirectory="${activemq.base}/data" dataSource="#derby-ds"/>
        </persistenceAdapter>


          <!--
            The systemUsage controls the maximum amount of space the broker will
            use before disabling caching and/or slowing down producers. For more information, see:
            http://activemq.apache.org/producer-flow-control.html
          -->
          <systemUsage>
            <systemUsage>
                <memoryUsage>
                    <memoryUsage percentOfJvmHeap="70" />
                </memoryUsage>
                <storeUsage>
                    <storeUsage limit="100 gb"/>
                </storeUsage>
                <tempUsage>
                    <tempUsage limit="50 gb"/>
                </tempUsage>
            </systemUsage>
        </systemUsage>

        <!--
            The transport connectors expose ActiveMQ over a given protocol to
            clients and other brokers. For more information, see:

            http://activemq.apache.org/configuring-transports.html
        -->
        <transportConnectors>
            <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
            <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="amqp" uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="mqtt" uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="ws" uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
        </transportConnectors>

        <!-- destroy the spring context on shutdown to stop jetty -->
        <shutdownHooks>
            <bean xmlns="http://www.springframework.org/schema/beans" class="org.apache.activemq.hooks.SpringContextHook" />
        </shutdownHooks>

    </broker>
   <!-- 加入的配置  --> 
  <bean id="derby-ds" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost/activemq?relaxAutoCommit=true"/>
    <property name="username" value="root"/>
    <property name="password" value=""/>
    <property name="maxActive" value="200"/>
    <property name="poolPreparedStatements" value="true"/>
  </bean>
    <!--
        Enable web consoles, REST and Ajax APIs and demos
        The web consoles requires by default login, you can disable this in the jetty.xml file

        Take a look at ${ACTIVEMQ_HOME}/conf/jetty.xml for more details
    -->
    <import resource="jetty.xml"/>

</beans>
<!-- END SNIPPET: example -->
```

启动ActiveMQ，分析数据表结构发现多了3张表：

**activemq\_acks、activemq\_lock、activemq\_msgs**

> **activemq\_acks：**用于存储订阅关系。如果是持久化Topic，订阅者和服务器的订阅关系在这个表保存。
>
> CONTAINER：消息的目的地  
> SUB\_DEST：如果是使用static集群，这个字段会有集群其他系统的信息  
> CLIENT\_ID：每个订阅者客户端ID  
> SUB\_NAME：订阅者名称  
> SELECTOR：选择器，可以选择只消费满足条件的消息。条件可以用自定义属性实现，可支持多属性and和or操作  
> LAST\_ACKED\_ID：记录消费过的消息的id。  
> PRIORITY：优先级  
> XID：  
> **activemq\_lock**：用于记录哪一个Broker是Master Broker。这张表只有在集群环境中才会用到，在集群中，只能有一个Broker来接收消息，那么这个Broker就是主Broker，其他的作为从Broker，用来备份等待。
>
> ID：主键
>
> TIME：时间
>
> BROKER\_NAME：Broker名称
>
> **activemq\_msgs**：用于存储消息，Topic和Queue消息都会保存在这张表中
>
> ID：自增主键  
> CONTAINER：容器名称  
> MSGID\_PROD：消息发送者客户端的主键  
> MSGID\_SEQ：发送消息的顺序，msgid\_prod+msg\_seq可以组成jms的messageid  
> EXPIRATION：消息的过期时间，存储的是从1970-01-01到现在的毫秒数  
> MSG：消息本体的java序列化对象的二进制数据  
> PRIORITY：优先级，从0-9，数值越大优先级越高

然后分别启动生产者和消费者，在MySQL中进行消息数据查看。

#### 持久化为数据库



