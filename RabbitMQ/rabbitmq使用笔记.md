# Rabbitmq使用笔记

###### gist:

publish -> exchange(fanout, direct, topic) -> queue -> consumer

四种类型交换机：fanout 广播，direct 直连，topic主题，header 头交换机

fanout ，direct 绑定多个或单个 queueLabel

binding 过程通过 routeKey 来唯一识别



tutorials:

## 1.Rabbitmq 简介

### 1.1 什么是消息中间件

消息队列中间件，或叫消息中间件，消息队列均可

有两种传递模式：1. 点对点（P2P）  2. 发布/订阅（Publisher/Subscriber）



### 1.2 消息中间件的使用

1.解耦

2.冗余（存储）

3.扩展性

4.流量削峰

5.可恢复性

6.顺序保证

7.缓冲

8.异步通信



### 1.3 Rabbitmq 的起源

### 1.4 Rabbitmq 的安装及简单使用

```java
// Helloworld Demo
// 消费者 client 端
package com.ly.rabbitmq.demo;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.MessageProperties;
import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitProducer {
  	private static final String EXCHANGE_NAME = "exchange_demo";
    private static final String ROUTING_KEY = "routingkey_demo";
    private static final String QUEUE_NAME = "queue_demo";
    private static final String IP_ADDRESS = "192.168.188.254";
    private static final int PORT = 5672;	// RabbitMQ 服务器默认端口号
    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost(IP_ADDRESS);
        factory.setPort(PORT);
        factory.setUsername("root");
        factory.setPassword("root123");
        Connection connection = factory.newConnection();	// 创建连接
        Channel channel = connection.createChannel();	// 创建信道
        channel.exchangeDeclare(EXCHANGE_NAME, "direct", true, false, null);	// 创建一个直连，持久化，非自动删除的交换器
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);	// 创建一个持久化的，非排他的，非自动删除的队列
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, ROTING_KEY);	//	通过 routingkey 绑定交换器与队列
        String message = "Hello World!";
        channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes);
        
        channel.close();
        connection.close();
    }
}

// 消费者客户端 ConsumerClient
package com.ly.rabbitmq.demo;

import com.rabbitmq.client.*;
import java.io.IOException;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

public class RabbitConsumer {
    private static final String QUEUE_NAME = "queue_demo";
    private static final String IP_ADDRESS = "192.168.188.253";
    private static final int PORT = 5672;
    
    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        Address[] addresses = new Address[] {
            new Address(IP_ADDRESS, PORT)
        };
        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("root");
        factory.setPassword("123456");
        // 连接方式与上面不同，注意对比
        Connection connection = factory.newConnection(addresses);	// 创建连接
        final Channel channel = connection.createChannel();		// 创建信道
        channel.basicQos(64);	// 设置客户端最多接收未被 ACK 的消息个数
        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("recv message: " + new String(body));
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch(InterruptedException e) {
                    e.printStackTrace();
                }
                channel.basicAck(envelope.getDeliveryTag(), false);
            }
        };
       	channel.basicConsume(QUEUE_NAME, consumer);
        // 等待回调函数执行完毕以后，关闭资源
        TimeUnit.SECONDS.sleep(5);
        channel.close();
        connection.close();
    }
}

```



## 2. Rabbitmq 入门

### 2.1 相关概念介绍

Exchange 分四种类型：fanout, direct, topic, header(头交换器用的很少)

###### 小结：RabbitMQ 运行流程

生产者端：

1.生产者连接到 RabbitMQ broker，建立一个连接 (Connection)，开启一个信道（Channel）

2.生产者声明交换器，设置交换器类型（四种之一），是否持久化，是否自动删除等。

3.生产者声明一个队列并设置是否排他，是否持久化，是否自动删除等。

4.生产者通过 routingKey 创建交换器和队列的绑定关系。

5.生产者发送消息到 RabbitMQ broker, 消息中要包含 routingKey, exchange_name 等必要信息。

6.相应的交换器根据接收到的 routingKey 查找相匹配的队列。

7.如果找到匹配的队列，则将从生产者发过来的消息存入相应的队列

8.如果没找到，则根据生产者配置的属性，选择丢弃还是回退给生产者

9.关闭 channel，再关闭 connection.

消费者端：

1.消费者连接到 RabbitMQ broker，建立一个连接（Connection），开启一个信道（Channel）.

2.消费者向 RabbitMQ  broker 请求消费相应队列中的消息，可能会设置相应的回调函数，以及做一些准备工作。

3.等待 RabbiteMQ broker 回应并投递相应队列中的消息，消费者接收消息。

4.消费者确认（ack）接收到的消息。

5.RabbitMQ 从队列中删除相应的已经被确认的消息。

6.关闭 channel，在关闭 connection.

### 2.2 AMQP 协议介绍

###### 标准 AMQP 协议包含三层：

1.Module Layer: 模块层。位于协议的最上层，主要定义了一些可供客户端调用的命令，客户端可以利用这些命令实现自己的业务逻辑。例如客户端使用 Queue.Declare 命令声明一个队列或者使用 Basic.Consume 订阅消费一个队列中的消息。

2.Session Layer: 会话层。主要负责将客户端的命令发送给服务器，再将服务器端的应答返回给客户端。主要为客户端与服务器之间的通信提供可靠的同步机制，以及错误处理机制。

3.Transport Layer: 传输层。主要传输二进制数据流，提供帧的处理，信道复用，错误检测和数据表示等。

###### AMQP 生产者流转过程：

// todo

###### AMQP 消费者流转过程：

// todo

###### AMQP 命令相关：

// todo

## 3. 客户端开发向导

### 3.1 连接 RabbitMQ

### 3.2 使用交换器和队列

### 3.3 发送消息

### 3.4 消费消息

### 3.5 消费端的确认与拒绝

### 3.6 关闭连接

## 4. RabbitMQ 进阶

mandatory 和 immediate 是 channel.basicPublish 方法中的两个参数，他们都有当消息传递过程中不可达目的地时将消息返回给生产者的功能。RabbitMQ 提供的备份交换器（Alternate Exchange）可以将未能被交换器路由的消息（没有绑定队列或者没有匹配的绑定）存储起来，而不用返回给客户端。



### 4.1 消息何去何从

#### 4.1.1 mandatory 参数

#### 4.1.2 immediate 参数

#### 4.1.3 备份交换器

### 4.2 过期时间(TTL)

#### 4.2.1 设置消息的 TTL

#### 4.2.2 设置队列的 TTL

### 4.3 死信队列

### 4.4 延迟队列

### 4.5 优先级队列

### 4.6 RPC 实现

### 4.7 持久化

### 4.8 生产者确认

#### 4.8.1 事务机制

#### 4.8.2 发送方确认机制

### 4.9 消费端要点介绍

#### 4.9.1 消息分发

#### 4.9.2 消息顺序性

#### 4.9.3 弃用 QueueingConsumer

### 4.10 消息传输保障

## 5. RabbitMQ 管理

### 5.1 多用户与权限

### 5.2 用户管理

### 5.3 Web 端管理

### 5.4 应用与集群管理

### 5.5 服务端状态

### 5.6 HTTPAPI 接口管理

## 6. RabbitMQ 配置

### 6.1 环境变量

### 6.2 配置文件

### 6.3 参数策略

## 7. RabbitMQ 运维

### 7.1 集群搭建

### 7.2 查看服务日志

### 7.3 单节点故障恢复

### 7.4 集群迁移

### 7.5 集群监控

## 8. 跨越集群的界限

### 8.1 Federation

### 8.2 Shovel

## 9. RabbitMQ 高阶

### 9.1 存储机制

### 9.2 内存及磁盘告警

### 9.3 流控

### 9.4 镜像队列

## 10. 网络分区

### 10.1 做网络分区意义

### 10.2 判定

### 10.3 模拟

### 10.4 网络分区的影响

### 10.5 手动处理网络分区

### 10.6 自动处理网络分区

### 10.7 案例：多分区情形

## 11. RabbitMQ 扩展

### 11.1 消息追踪

### 11.2 负载均衡

