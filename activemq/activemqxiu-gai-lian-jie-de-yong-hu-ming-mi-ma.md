# ActiveMQ修改连接的用户名密码

安装目录下conf/activemq.xml

添加如下内容：

&lt;plugins&gt;  
&lt;simpleAuthenticationPlugin&gt;  
&lt;users&gt;  
&lt;authenticationUser username="zhangsan" password="123" groups="users,admins"/&gt;  
&lt;/users&gt;  
&lt;/simpleAuthenticationPlugin&gt;  
&lt;/plugins&gt;

![](https://images2018.cnblogs.com/blog/1058092/201803/1058092-20180328204044647-1845672871.png)

