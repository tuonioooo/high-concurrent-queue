# 安装配置

## 参考文档：

RabbitMQ实战  高效部署分布式消息队列.pdf

链接：[https://pan.baidu.com/s/1ZRj6Q9qaquE2e9Ww8Q-nkw](https://pan.baidu.com/s/1ZRj6Q9qaquE2e9Ww8Q-nkw) 密码：71ug

RabbitMQ官网地址：    [http://www.rabbitmq.com](http://www.rabbitmq.com)

第三方用户示例地址：

[https://www.jb51.net/article/59823.htm](https://www.jb51.net/article/59823.htm)

[https://www.cnblogs.com/huacw/p/5968227.html](https://www.cnblogs.com/huacw/p/5968227.html)



## 常用命令总结：

> \#启动/停止 start/stop 
>
> $sudo /sbin/service rabbitmq-server start 
>
> Starting rabbitmq-server: SUCCESS 
>
> rabbitmq-server.
>
> $sudo /sbin/service rabbitmq-server stop 
>
> Stopping rabbitmq-server: rabbitmq-server.
>
> \#状态查看 sudo rabbitmqctl status

  


##  常见问题总结：

* 使用命令  service rabbitmq-server start 一直无法启动

> Startup\_err 中记录以下错误信息
>
> /usr/lib/rabbitmq/bin/rabbitmq-server: line 50: erl: command not found
>
> 是因为环境变量不同，导致无法找到相应命令，按照指引将erlang的erl软连接到/usr/bin目录下
>
> \[root@iZ250x18mnzZ rabbitmq\]\# ln -s /usr/local/erlang/bin/erl /usr/bin/erl重新执行成功

* 异常处理：java.util.concurrent.TimeoutException

> 解决方式：
>
> 该测试的broker使用了一台，没有主备，所以在/etc/hosts下没有配置IP和域名的对应关系。因此导致producer调用api登陆broker有时候比较耗时。
>
> 参考broker上IP和hostname配置说明：
>
> [http://www.rabbitmq.com/clustering.html](http://www.rabbitmq.com/clustering.html)
>
> [http://docs.celeryproject.org/en/latest/getting-started/brokers/rabbitmq.html](http://docs.celeryproject.org/en/latest/getting-started/brokers/rabbitmq.html)
>
> 配置远程访问：
>
> [https://blog.csdn.net/qq\_22075041/article/details/78855708](https://blog.csdn.net/qq_22075041/article/details/78855708)



## 设置账号密码

> \#
>
> 添加用户\#./rabbitmqctl add\_user 账号 密码
>
> ./rabbitmqctl add\_user admin admin
>
> \#分配用户标签\(admin为要赋予administrator权限的刚创建的那个账号的名字\)
>
> ./rabbitmqctl set\_user\_tags admin administrator
>
> \#设置权限&lt;即开启远程访问&gt;\(如果需要远程连接,例如java项目中需要调用mq,则一定要配置,否则无法连接到mq,admin为要赋予远程访问权限的刚创建的那个账号的名字,必须运行着rabbitmq此命令才能执行\)
>
> ./rabbitmqctl set\_permissions -p "/" admin ".\*" ".\*" ".\*"

## 



