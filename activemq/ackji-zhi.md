# ack机制

## 概述

ACK \(Acknowledgement）即是确认字符，在数据通信中，接收站发给发送站的一种传输类[控制字符](https://baike.baidu.com/item/%E6%8E%A7%E5%88%B6%E5%AD%97%E7%AC%A6/6913704)。表示发来的数据已确认接收无误。在[TCP/IP协议](https://baike.baidu.com/item/TCP%2FIP%E5%8D%8F%E8%AE%AE)中，如果接收方成功的接收到数据，那么会回复一个ACK数据。通常ACK信号有自己固定的格式,长度大小,由接收方回复给发送方。

**详细释义**

其格式取决于采取的网络协议。当发送方接收到ACK信号时，就可以发送下一个数据。如果发送方没有收到信号，那么发送方可能会重发当前的[数据包](https://baike.baidu.com/item/%E6%95%B0%E6%8D%AE%E5%8C%85)，也可能停止传送数据。具体情况取决于所采用的[网络协议](https://baike.baidu.com/item/%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE)。

1、TCP[报文](https://baike.baidu.com/item/%E6%8A%A5%E6%96%87)格式中的控制位由6个标志比特构成，其中一个就是ACK，ACK为1表示确认号有效，为0表示报文中不包含确认信息，忽略确认号字段。

2、ACK也可用于AT24cxx这一系列的EEPROM中。

3、在USB传输中，ACK事务包用来向主机/设备报告包正确的传输。



