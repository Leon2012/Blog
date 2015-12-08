title: kafka学习笔记
date: 2015/10/06 19:00:00
tags: [kafka,java]
categories: opensource
---

前段时间看kafka, 简单记录下:

### zookeeper
因为kafka的数据都是存在zk里的, 可以用zk客户端工具(例如ZooInspector)连接zk服务器查看, 可以比较直观的看到数据存储结构.

### 监控
可以使用KafkaOffsetMonitor这个工具, 启动以后访问类似的地址`curl http://localhost:8888/group/group1`可以看到返回的json报文, 可以看到消费的offset以及lag情况, 监控lag可以及时发现消费堵塞的问题.

### key默认分区
kafka的客户端生产消息可以通过`producer.send(new KeyedMessage<String, String>(topic, key, message))`这个方法, 但是key是可以为空的, 如果为空则一个producer会固定写到一个分区里, 会导致分布不均衡.  
所以如果没有key的话可以用随机数做为key, 随机数的范围也不能太小, 不然也会不均衡.

### 分区数
默认配置新建分区数是1, 最好修改下, 也可以在新建topic的时候指定分区数.  
一个分区只能被一个消费者消费, 所以分区数不能比消费者的个数还少, 不然多出来的消费者就浪费了.因此分区数最好是12、24这样, 方便以后扩容.

### 客户端
kafka本来只有scala的客户端, 0.8.2版本提供了java的客户端, 但是还 **只支持生产者** , 后续应该也会支持消费者.
