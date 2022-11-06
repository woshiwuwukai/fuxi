# 0. 学习目标

- 能够说出什么是消息中间件
- 能够安装RabbitMQ
- 能够编写RabbitMQ的入门程序
- 能够说出RabbitMQ的5种模式特征
- 能够使用Spring整合RabbitMQ



# 1. 消息中间件概述

## 1.1. 什么是消息中间件

MQ全称为Message Queue，消息队列是应用程序和应用程序之间的通信方法。是在消息的传输过程中保存消息的容器。多用于分布式系统之间进行通信。

![image-20210312171534774](assets/image-20210312171534774.png) 

 

### 1.1.1为什么使用MQ

在项目中，可将一些无需即时返回且耗时的操作提取出来，进行**异步处理**，而这种异步处理的方式大大的节省了服务器的请求响应时间，从而**提高**了**系统**的**吞吐量**。



### 1.1.2消息队列（MQ）应用场景

1、任务**异步**处理

将不需要同步处理的并且耗时长的操作由消息队列通知消息接收方进行异步处理。提高了应用程序的响应时间。

![image-20210312172304257](assets/image-20210312172304257.png) 

同步方式的问题：当一个用户提交订单到成功需要300ms+300ms+300ms+20ms = 920ms，这是不能容忍的。也就是说库存、支付、物流、最后保存数据库全部成功，订单的提交才算完成

异步方式：

![image-20210312172632817](assets/image-20210312172632817.png) 





用户点击完下单按钮后，只需等待25ms就能得到下单响应 (20 + 5 = 25ms)。也就是说，订单消息提交到MQ，MQ回馈一个消息成功,然后再把订单提交到数据库20ms，就完成了。至于MQ通知库存、支付、物流系统所花费的时间和订单系统成功没有关系了。后
提升用户体验和系统吞吐量（单位时间内处理请求的数目）。



2、应用程序**解耦合**

MQ相当于一个中介，生产方通过MQ与消费方交互，它将应用程序进行解耦合。

![image-20210312171738311](assets/image-20210312171738311.png) 

系统的耦合性越高，容错性就越低，可维护性就越低

![image-20210312172014512](assets/image-20210312172014512.png) 

使用 MQ 使得应用间解耦，提升容错性和可维护性。库存和支付和物流直接去MQ取到订单的信息即可，即使库存系统报错，没关系，等到库存修复后再次从MQ中去取就可以了





3、**削峰填谷**

如订单系统，在下单的时候就会往数据库写数据。但是数据库只能支撑每秒1000左右的并发写入，并发量再高就容易宕机。低峰期的时候并发也就100多个，但是在高峰期时候，并发量会突然激增到5000以上，这个时候数据库肯定卡死了。

![image-20210312173150805](assets/image-20210312173150805.png)  



![image-20210312173339481](assets/image-20210312173339481.png) 



![](assets/01.jpg) 

消息被MQ保存起来了，5000条数据对于MQ，简直是小意思，然后系统就可以按照自己的消费能力来消费，比如每秒1000个数据，这样慢慢写入数据库，这样就不会卡死数据库了。

![](assets/02.jpg) 

但是使用了MQ之后，限制消费消息的速度为1000，但是这样一来，高峰期产生的数据势必会被积压在MQ中，高峰就被“削”掉了。但是因为消息积压，在高峰期过后的一段时间内，消费消息的速度还是会维持在1000QPS，直到消费完积压的消息,这就叫做“填谷”

![](assets/03.jpg)



### 1.1.3MQ的劣势

1、系统可用性降低
系统引入的外部依赖越多，系统稳定性越差。一旦 MQ 宕机，就会对业务造成影响。如何保证MQ的高可用？

2、系统复杂度提高
MQ 的加入大大增加了系统的复杂度，以前系统间是同步的远程调用，现在是通过 MQ 进行异步调用。如何保证消息没有被重复消费？怎么处理消息丢失情况？那么保证消息传递的顺序性？

3、一致性问题
A 系统处理完业务，通过 MQ 给B、C、D三个系统发消息数据，如果 B 系统、C 系统处理成功，D 系统处理失败。如何保证消息数据处理的一致性？

既然 MQ 有优势也有劣势，那么使用 MQ 需要满足什么条件呢？

~~~
生产者不需要从消费者处获得反馈。引入消息队列之前的直接调用，其接口的返回值应该为空，这才让明明下层的动作还没做，上层却当成动作做完了继续往后走，即所谓异步成为了可能。
容许短暂的不一致性。
确实是用了有效果。即解耦、提速、削峰这些方面的收益，超过加入MQ，管理MQ这些成本。

~~~



### 1.1.4常见的 MQ 产品

目前业界有很多的 MQ 产品，例如 RabbitMQ、RocketMQ、ActiveMQ、Kafka、ZeroMQ、MetaMq等，也有直接使用 Redis 充当消息队列的案例，而这些消息队列产品，各有侧重，在实际选型时，需要结合自身需求及 MQ 产品特征，综合考虑。

| ** **          | **RabbitMQ**                                                 | **ActiveMQ**                            | **RocketMQ**             | **Kafka**                                      |
| -------------- | ------------------------------------------------------------ | --------------------------------------- | ------------------------ | ---------------------------------------------- |
| 公司/社区      | Rabbit                                                       | Apache                                  | 阿里                     | Apache                                         |
| 开发语言       | Erlang                                                       | Java                                    | Java                     | Scala&Java                                     |
| 协议支持       | AMQP，XMPP，SMTP，STOMP                                      | OpenWire,STOMP，REST,XMPP,AMQP          | 自定义                   | 自定义协议，社区封装了http协议支持             |
| 客户端支持语言 | 官方支持Erlang，Java，Ruby等,社区产出多种API，几乎支持所有语言 | Java，C，C++，Python，PHP，Perl，.net等 | Java，C++（不成熟）      | 官方支持Java,社区产出多种API，如PHP，Python等  |
| 单机吞吐量     | 万级（其次）                                                 | 万级（最差）                            | 十万级（最好）           | 十万级（次之）                                 |
| 消息延迟       | 微妙级                                                       | 毫秒级                                  | 毫秒级                   | 毫秒以内                                       |
| 功能特性       | 并发能力强，性能极其好，延时低，社区活跃，管理界面丰富       | 老牌产品，成熟度高，文档较多            | MQ功能比较完备，扩展性佳 | 只支持主要的MQ功能，毕竟是为大数据领域准备的。 |



## 1.2. AMQP 和 JMS

MQ是消息通信的模型；实现MQ的大致有两种主流方式：AMQP、JMS。

### 1.2.1. AMQP

AMQP是一种协议，Advanced Message Queuing Protocol（高级消息队列协议）,是一个网络协议，是应用层协议的一个开放标准，为面向消息的中间件设计。基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，不同的开发语言等条件的限制。2006年，AMQP 规范发布。类比HTTP

![image-20210315082555386](assets/image-20210315082555386.png) 



### 1.2.2. JMS

JMS即Java消息服务（JavaMessage Service）应用程序接口，好比java提供一套jdbc的接口API，是一个Java平台中关于面向消息中间件（MOM）的API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。



### 1.2.3. AMQP 与 JMS 区别

- JMS是定义了统一的接口，来对消息操作进行统一；AMQP是通过规定协议来统一数据交互的格式
- JMS限定了必须使用Java语言；AMQP只是协议，不规定实现方式，因此是跨语言的。
- JMS规定了两种消息模式；而AMQP的消息模式更加丰富



## 1.3. RabbitMQ简介

RabbitMQ是由erlang语言开发，基于AMQP（Advanced Message Queue 高级消息队列协议）协议实现的消息队列，它是一种应用程序之间的通信方法，消息队列在分布式系统开发中应用非常广泛。2007年，Rabbit 技术公司基于 AMQP 标准开发的 RabbitMQ 1.0 发布。 (Erlang 语言由 Ericson 设计，专门为开发高并发和分布式系统的一种语言，在电信领域使用广泛)

RabbitMQ官方地址：http://www.rabbitmq.com/

RabbitMQ 基础架构如下图：

![image-20210315082502548](assets/image-20210315082502548.png) 

Broker：接收和分发消息的应用，就是一个中介，RabbitMQ Server就是 Message Broker
Virtual host：出于多租户和安全因素设计的，把 AMQP 的基本组件划分到一个虚拟的分组中，类似于网络中的 namespace 概念。当多个不同的用户使用同一个 RabbitMQ server 提供的服务时，可以划分出多个vhost，每个用户在自己的 vhost 创建 exchange／queue 等
Connection：publisher／consumer 和 broker 之间的 TCP 连接
Channel：如果每一次访问 RabbitMQ 都建立一个 Connection，在消息量大的时候建立 TCP Connection的开销将是巨大的，效率也较低。Channel 是在 connection 内部建立的逻辑连接，如果应用程序支持多线程，通常每个thread创建单独的 channel 进行通讯，AMQP method 包含了channel id 帮助客户端和message broker 识别 channel，所以 channel 之间是完全隔离的。Channel 作为轻量级的 Connection 极大减少了操作系统建立 TCP connection 的开销

Exchange：message 到达 broker 的第一站，根据分发规则，匹配查询表中的 routing key，分发消息到queue 中去。常用的类型有：direct (point-to-point), topic (publish-subscribe) and fanout (multicast)
Queue：消息最终被送到这里等待 consumer 取走
Binding：exchange 和 queue 之间的虚拟连接，binding 中可以包含 routing key。Binding 信息被保存到 exchange 中的查询表中，用于 message 的分发依据



RabbitMQ提供了6种模式：简单模式，work模式，Publish/Subscribe发布与订阅模式，Routing路由模式，Topics主题模式，RPC远程调用模式（远程调用，不太算MQ；暂不作介绍）；

官网对应模式介绍：https://www.rabbitmq.com/getstarted.html        点击手册按钮 RabbitMQ Tutorials



![1555988678324](assets/1555988678324.png)



# 2. 安装及配置RabbitMQ

RabbitMQ 官方地址：http://www.rabbitmq.com/  



# 3. RabbitMQ入门

## 3.1. 搭建示例工程

### 3.1.1 添加依赖

往woniu-rabbitmq的pom.xml文件中添加如下依赖：

```xml
 <dependencies>
        <!--rabbitmq java 客户端-->
        <dependency>
            <groupId>com.rabbitmq</groupId>
            <artifactId>amqp-client</artifactId>
            <version>5.6.0</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
```



## 3.2. 编写生产者

```
package com.woniu.producer;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Producer_HelloWord {

    public static void main(String[] args) throws IOException, TimeoutException {
        //1、创建连接工程
        ConnectionFactory factory = new ConnectionFactory();
        //2、设置参数
        factory.setHost("192.168.128.129");
        factory.setVirtualHost("/woniu");//虚拟机 默认值/
        factory.setPort(5672); //端口  默认值 5672
        factory.setUsername("woniu");//用户名 默认 guest
        factory.setPassword("woniu");//密码 默认值 guest
        //3. 创建连接 Connection
        Connection connection = factory.newConnection();
        //4. 创建Channel
        Channel channel = connection.createChannel();
        //5. 创建队列Queue
        /*
        queueDeclare(String queue,boolean durable,boolean exclusive,boolean autoDelete,Map<String,Object> arguments)
        参数：
            1、queue：队列名称
            2、durable:是否持久化，当mq重启之后，是否数据还在
            3、exclusive：
                * 否是独占，只能有一个消费者监听这个队列
                * 当connection关闭时，是否删除队列
            4、auto掉Delete：是否自动删除，当没有Consumer时，自动删除
            5、arguments：其它参数
         */

        //如果没有一个名字叫hello_world的队列，则会创建该队列，如果有就不会创建
        channel.queueDeclare("hello_world",true,false,false,null);
        /*
        basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body)
        参数：
            1、exchange：交换机名称，简单模式下交换机会使用默认的，默认是 “”
            2、routingKey：路由名称，如果交换机采用默认的 “ ” 那么routingKey要和队列名字保持一样
            3、props：配置信息
            4、body：发送消息数据
         */
        String body = "你好，小兔子";
        //6、发送消息
        channel.basicPublish("","hello_world",null,body.getBytes());
        //7.释放资源
        channel.close();
        connection.close();
    }
}
```

在执行上述的消息发送之后；可以登录rabbitMQ的管理控制台，可以发现队列和其消息：

![image-20210315143719950](assets/image-20210315143719950.png) 



![1556006647177](assets/1556006647177.png)



## 3.3. 编写消费者

1、导入jar，和生产者的jar一样，略

~~~
package com.woniu.comsumer;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.nio.charset.Charset;
import java.util.concurrent.TimeoutException;

public class Consumer_Helloword {
    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("woniu");
        factory.setPassword("woniu");
        factory.setPort(5672);
        factory.setHost("192.168.128.129");

        //虚拟主机名
        factory.setVirtualHost("/woniu");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 声明（创建）队列
        /**
         * 参数1：队列名称
         * 参数2：是否定义持久化队列
         * 参数3：是否独占本次连接
         * 参数4：是否在不使用的时候自动删除队列
         * 参数5：队列其它参数
         */
        channel.queueDeclare("hello_world",true,false,false,null);
        DefaultConsumer consumer = new DefaultConsumer(channel){
            //回调方法，当收到消息后，会自动执行该方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                /*
                参数
                    1、consumerTag 消息者标签，在channel.basicConsumer时候可以指定
                    2、envelope 消息包的内容，可从中获取消息id，消息routingkey，交换机，消息和重传标志（收到消息失败后是否需要重新发送）
                    3、properties 属性信息
                    4、body：消息
                 */
                System.out.println("consumerTag:"+consumerTag);
                //路由key
                System.out.println("路由key为：" + envelope.getRoutingKey());
                //交换机
                System.out.println("交换机为：" + envelope.getExchange());
                //消息id
                System.out.println("消息id为：" + envelope.getDeliveryTag());
                //收到的消息
                System.out.println("接收到的消息为：" + new String(body, "utf-8"));
            }
        };
        /* basicConsume(String queue, boolean autoAck, Consumer callback)
        参数：
            1、queue：队列名称
            2、是否自动确认，设置为true为表示消息接收到自动向mq回复接收到了，mq接收到回复会删除消息，设置为false则需要手动确认
        */
        channel.basicConsume("hello_world",true,consumer);
        //不关闭资源，应该一直监听消息
        //channel.close();
        //connection.close();
    }
}

~~~







## 3.4. 小结

上述的入门案例中中其实使用的是如下的简单模式：

![1555991074575](assets/1555991074575.png)

在上图的模型中，有以下概念：

- P：生产者，也就是要发送消息的程序
- C：消费者：消息的接受者，会一直等待消息到来。
- queue：消息队列，图中红色部分。类似一个邮箱，可以缓存消息；生产者向其中投递消息，消费者从其中取出消息。



# 4. AMQP

## 4.1. 相关概念介绍

AMQP  一个提供统一消息服务的应用层标准高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。

AMQP是一个二进制协议，拥有一些现代化特点：多信道、协商式，异步，安全，扩平台，中立，高效。

RabbitMQ是AMQP协议的Erlang的实现。

| 概念           | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| 连接Connection | 一个网络连接，比如TCP/IP套接字连接。                         |
| 会话Session    | 端点之间的命名对话。在一个会话上下文中，保证“恰好传递一次”。 |
| 信道Channel    | 多路复用连接中的一条独立的双向数据流通道。为会话提供物理传输介质。 |
| 客户端Client   | AMQP连接或者会话的发起者。AMQP是非对称的，客户端生产和消费消息，服务器存储和路由这些消息。 |
| 服务节点Broker | 消息中间件的服务节点；一般情况下可以将一个RabbitMQ Broker看作一台RabbitMQ 服务器。 |
| 端点           | AMQP对话的任意一方。一个AMQP连接包括两个端点（一个是客户端，一个是服务器）。 |
| 消费者Consumer | 一个从消息队列里请求消息的客户端程序。                       |
| 生产者Producer | 一个向交换机发布消息的客户端应用程序。                       |

## 4.2. RabbitMQ运转流程

在入门案例中：

- 生产者发送消息
  1. 生产者创建连接（Connection），开启一个信道（Channel），连接到RabbitMQ Broker；
  2. 声明队列并设置属性；如是否排它，是否持久化，是否自动删除；
  3. 将路由键（空字符串）与队列绑定起来；
  4. 发送消息至RabbitMQ Broker；
  5. 关闭信道；
  6. 关闭连接；
- 消费者接收消息
  1. 消费者创建连接（Connection），开启一个信道（Channel），连接到RabbitMQ Broker
  2. 向Broker 请求消费相应队列中的消息，设置相应的回调函数；
  3. 等待Broker回应闭关投递响应队列中的消息，消费者接收消息；
  4. 确认（ack，自动确认）接收到的消息；
  5. RabbitMQ从队列中删除相应已经被确认的消息；
  6. 关闭信道；
  7. 关闭连接；

![1565105223969](assets/1565105223969.png)



## 4.3. 生产者流转过程说明

1. 客户端与代理服务器Broker建立连接。会调用newConnection() 方法,这个方法会进一步封装Protocol Header 0-9-1 的报文头发送给Broker ，以此通知Broker 本次交互采用的是AMQPO-9-1 协议，紧接着Broker 返回Connection.Start 来建立连接，在连接的过程中涉及Connection.Start/.Start-OK 、Connection.Tune/.Tune-Ok ，Connection.Open/ .Open-Ok 这6 个命令的交互。
2. 客户端调用connection.createChannel方法。此方法开启信道，其包装的channel.open命令发送给Broker,等待channel.basicPublish方法，对应的AMQP命令为Basic.Publish,这个命令包含了content Header 和content Body()。content Header 包含了消息体的属性，例如:投递模式，优先级等，content Body 包含了消息体本身。
3. 客户端发送完消息需要关闭资源时，涉及到Channel.Close和Channl.Close-Ok 与Connetion.Close和Connection.Close-Ok的命令交互。

![生产者流转过程图](assets/生产者流转过程图.bmp)

## 4.4. 消费者流转过程说明

1. 消费者客户端与代理服务器Broker建立连接。会调用newConnection() 方法,这个方法会进一步封装Protocol Header 0-9-1 的报文头发送给Broker ，以此通知Broker 本次交互采用的是AMQPO-9-1 协议，紧接着Broker 返回Connection.Start 来建立连接，在连接的过程中涉及Connection.Start/.Start-OK 、Connection.Tune/.Tune-Ok ，Connection.Open/ .Open-Ok 这6 个命令的交互。
2. 消费者客户端调用connection.createChannel方法。和生产者客户端一样，协议涉及Channel . Open/Open-Ok命令。
3. 在真正消费之前，消费者客户端需要向Broker 发送Basic.Consume 命令(即调用channel.basicConsume 方法〉将Channel 置为接收模式，之后Broker 回执Basic . Consume - Ok 以告诉消费者客户端准备好消费消息。
4. Broker 向消费者客户端推送(Push) 消息，即Basic.Deliver 命令，这个命令和Basic.Publish 命令一样会携带Content Header 和Content Body。
5. 消费者接收到消息并正确消费之后，向Broker 发送确认，即Basic.Ack 命令。
6. 客户端发送完消息需要关闭资源时，涉及到Channel.Close和Channl.Close-Ok 与Connetion.Close和Connection.Close-Ok的命令交互。

![消费者流转过程图](assets/消费者流转过程图.bmp)



# 5. RabbitMQ工作模式

## 4.1. Work queues工作队列模式

### 4.1.1. 模式说明

![1556009144848](assets/1556009144848.png) 

`Work Queues`与入门程序的`简单模式`相比，多了一个或一些消费端，多个消费端共同消费同一个队列中的消息。它们处于竞争者的关系，也就是说只有一个消费者消息消息

**应用场景**：对于 任务过重或任务较多情况使用工作队列可以提高任务处理的速度。如生产者生产一千条消息，那么c1和c2各消费500条

### 4.1.2. 代码

`Work Queues`（工作队列）与入门程序的`简单模式`的代码是几乎一样的；可以完全复制，并复制多一个消费者进行多个消费者同时消费消息的测试。

#### 1）生产者

```java
package com.woniu.producer;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Producer_WorkQueues {

    public static void main(String[] args) throws IOException, TimeoutException {
        //1、创建连接工程
        ConnectionFactory factory = new ConnectionFactory();
        //2、设置参数
        factory.setHost("192.168.128.129");
        factory.setVirtualHost("/woniu");//虚拟机 默认值/
        factory.setPort(5672); //端口  默认值 5672
        factory.setUsername("woniu");//用户名 默认 guest
        factory.setPassword("woniu");//密码 默认值 guest
        //3. 创建连接 Connection
        Connection connection = factory.newConnection();
        //4. 创建Channel
        Channel channel = connection.createChannel();
        //5. 创建队列Queue
        /*
        queueDeclare(String queue,boolean durable,boolean exclusive,boolean autoDelete,Map<String,Object> arguments)
        参数：
            1、queue：队列名称
            2、durable:是否持久化，当mq重启之后，是否数据还在
            3、exclusive：
                * 否是独占，只能有一个消费者监听这个队列
                * 当connection关闭时，是否删除队列
            4、autoDelete：是否自动删除，当没有Consumer时，自动删除
            5、arguments：其它参数
         */

        //如果没有一个名字叫hello_world的队列，则会创建该队列，如果有就不会创建
        channel.queueDeclare("work_queues",true,false,false,null);
        /*
        basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body)
        参数：
            1、exchange：交换机名称，简单模式下交换机会使用默认的，默认是 “”
            2、routingKey：路由名称，如果交换机采用默认的 “ ” 那么routingKey要和队列名字保持一样
            3、props：配置信息
            4、body：发送消息数据
         */
        for (int i = 1; i <=10 ; i++) {
            String body = i+"hello work queues~~~~~";
            //6、发送消息
            channel.basicPublish("","work_queues",null,body.getBytes());
        }
        //7.释放资源
        channel.close();
        connection.close();
    }
}

```

#### 2）消费者1

```java
package com.woniu.comsumer;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Consumer_WorkQueues1 {
    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("woniu");
        factory.setPassword("woniu");
        factory.setPort(5672);
        factory.setHost("192.168.128.129");

        //虚拟主机名
        factory.setVirtualHost("/woniu");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 声明（创建）队列
        /**
         * 参数1：队列名称
         * 参数2：是否定义持久化队列
         * 参数3：是否独占本次连接
         * 参数4：是否在不使用的时候自动删除队列
         * 参数5：队列其它参数
         */
        channel.queueDeclare("work_queues",true,false,false,null);
        DefaultConsumer consumer = new DefaultConsumer(channel){
            //回调方法，当收到消息后，会自动执行该方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                /*
                参数
                    1、consumerTag 消息者标签，在channel.basicConsumer时候可以指定
                    2、envelope 消息包的内容，可从中获取消息id，消息routingkey，交换机，消息和重传标志（收到消息失败后是否需要重新发送）
                    3、properties 属性信息
                    4、body：消息
                 */
                System.out.println("接收到的消息为：" + new String(body, "utf-8"));
            }
        };
        /* basicConsume(String queue, boolean autoAck, Consumer callback)
        参数：
            1、queue：队列名称
            2、是否自动确认，设置为true为表示消息接收到自动向mq回复接收到了，mq接收到回复会删除消息，设置为false则需要手动确认
        */
        channel.basicConsume("work_queues",true,consumer);
        //不关闭资源，应该一直监听消息
        //channel.close();
        //connection.close();
    }
}

```

#### 3）消费者2

```java
package com.woniu.comsumer;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Consumer_WorkQueues2 {
    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("woniu");
        factory.setPassword("woniu");
        factory.setPort(5672);
        factory.setHost("192.168.128.129");

        //虚拟主机名
        factory.setVirtualHost("/woniu");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 声明（创建）队列
        /**
         * 参数1：队列名称
         * 参数2：是否定义持久化队列
         * 参数3：是否独占本次连接
         * 参数4：是否在不使用的时候自动删除队列
         * 参数5：队列其它参数
         */
        channel.queueDeclare("work_queues",true,false,false,null);
        DefaultConsumer consumer = new DefaultConsumer(channel){
            //回调方法，当收到消息后，会自动执行该方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                /*
                参数
                    1、consumerTag 消息者标签，在channel.basicConsumer时候可以指定
                    2、envelope 消息包的内容，可从中获取消息id，消息routingkey，交换机，消息和重传标志（收到消息失败后是否需要重新发送）
                    3、properties 属性信息
                    4、body：消息
                 */
                System.out.println("接收到的消息为：" + new String(body, "utf-8"));
            }
        };
        /* basicConsume(String queue, boolean autoAck, Consumer callback)
        参数：
            1、queue：队列名称
            2、是否自动确认，设置为true为表示消息接收到自动向mq回复接收到了，mq接收到回复会删除消息，设置为false则需要手动确认
        */
        channel.basicConsume("work_queues",true,consumer);
        //不关闭资源，应该一直监听消息
        //channel.close();
        //connection.close();
    }
}

```



### 4.1.3. 测试

启动两个消费者，然后再启动生产者发送消息；到IDEA的两个消费者对应的控制台查看是否竞争性的接收到消息。

### 4.1.4. 小结

在一个队列中如果有多个消费者，那么消费者之间对于同一个消息的关系是**竞争**的关系。



## 4.2. 订阅模式类型

订阅模式示例图：

![1556014499573](assets/1556014499573.png)

前面2个案例中，只有3个角色：

- P：生产者，也就是要发送消息的程序
- C：消费者：消息的接受者，会一直等待消息到来。
- queue：消息队列，图中红色部分



而在订阅模型中，多了一个exchange角色，而且过程略有变化：

- P：生产者，也就是要发送消息的程序，但是不再发送到队列中，而是发给X（交换机）
- C：消费者，消息的接受者，会一直等待消息到来。
- Queue：消息队列，接收消息、缓存消息。
- Exchange：交换机，图中的X。一方面，接收生产者发送的消息。另一方面，知道如何处理消息，例如递交给某个特别队列、递交给所有队列、或是将消息丢弃。到底如何操作，取决于Exchange的类型。Exchange有常见以下3种类型：
  - Fanout：广播，将消息交给所有绑定到交换机的队列
  - Direct：定向，把消息交给符合指定routing key 的队列
  - Topic：通配符，把消息交给符合routing pattern（路由模式） 的队列

**Exchange（交换机）只负责转发消息，不具备存储消息的能力**，因此如果没有任何队列与Exchange绑定，或者没有符合路由规则的队列，那么消息会丢失！



## 4.3. Publish/Subscribe发布与订阅模式

### 4.3.1. 模式说明

![1556010329032](assets/1556010329032.png) 

发布订阅模式：
1、P：生产者，也就是要发送消息的程序，但是不再发送到队列中，而是发给X（交换机）
2、C：消费者，消息的接收者，会一直等待消息到来
3、Queue：消息队列，接收消息、缓存消息
4、Exchange：交换机（X）。一方面，接收生产者发送的消息。另一方面，知道如何处理消息，例如递交给某个特别队列、递交给所有队列、或是将消息丢弃。到底如何操作，取决于Exchange的类型。Exchange有常见以下3种类型：
Fanout：广播，将消息交给所有绑定到交换机的队列
Direct：定向，把消息交给符合指定routing key 的队列
Topic：通配符，把消息交给符合routing pattern（路由模式） 的队列
Exchange（交换机）只负责转发消息，不具备存储消息的能力，因此如果没有任何队列与 Exchange 绑定，或者没有符合路由规则的队列，那么消息会丢失！



### 4.3.2. 代码

#### 1）生产者

```java
 package com.woniu.producer;

import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Producer_PubSub {

    public static void main(String[] args) throws IOException, TimeoutException {
        //1、创建连接工程
        ConnectionFactory factory = new ConnectionFactory();
        //2、设置参数
        factory.setHost("192.168.128.129");
        factory.setVirtualHost("/woniu");//虚拟机 默认值/
        factory.setPort(5672); //端口  默认值 5672
        factory.setUsername("woniu");//用户名 默认 guest
        factory.setPassword("woniu");//密码 默认值 guest
        //3. 创建连接 Connection
        Connection connection = factory.newConnection();
        //4. 创建Channel
        Channel channel = connection.createChannel();
        //5、创建交换机
        /*
        exchangeDeclare(String exchange, BuiltinExchangeType type, boolean durable,
                            boolean autoDelete, boolean internal, Map<String, Object> arguments)
        参数说明
            1、exchange 交换机名称
            2、type 交换机的类型
                DIRECT("direct"), 定向
                FANOUT("fanout"), 扇形（广播） 发送消息到每一个与之绑定队列
                TOPIC("topic"), 通配符的方式
                HEADERS("headers"); 参数匹配  实际开发中使用不多
            3、durable 是否持久化
            4、autoDelete 是否自动删除
            5、internal  内部使用，一般为false
            6、arguments：参数
         */
        String exchange_name = "test_fanout";
        channel.exchangeDeclare(exchange_name, BuiltinExchangeType.FANOUT,false,false,null);
        //6、创建队列
        String queue1Name = "test_fanout_queue1";
        String queue2Name = "test_fanout_queue2";
        channel.queueDeclare(queue1Name,true,false,false,null);
        channel.queueDeclare(queue2Name,true,false,false,null);
        //7、交换机和队列的绑定
        /*
        queueBind(String queue, String exchange, String routingKey)
        参数：
            1、queue  队列名称
            2、exchange 交换机名称
            3、routingKey 路由的键，绑定规则，如果交换机的类型为fanout，routingKey设置为“”
       */
        channel.queueBind(queue1Name,exchange_name,"");
        channel.queueBind(queue2Name,exchange_name,"");
        //8、发送消息
        String message = "日志消息，虎大调用了findAll方法...日志级别：info...";
        /*
        basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body)
        参数说明
            1、exchange 交换机名称
            2、routingKey “”  （如果交换机类型是fanout，则routingKey为“”）
            3、props 基本属性配置
            4、body  要发送的数据
         */
        channel.basicPublish(exchange_name,"",null,message.getBytes());
        //9.释放资源
        channel.close();
        connection.close();
    }
}

```



#### 2）消费者1

```java
package com.woniu.comsumer;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Consumer_PubSub1 {
    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("woniu");
        factory.setPassword("woniu");
        factory.setPort(5672);
        factory.setHost("192.168.128.129");

        //虚拟主机名
        factory.setVirtualHost("/woniu");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        //不需要创建队列了，只需要绑定要监听的队列即可
        String queue1Name = "test_fanout_queue1";
        DefaultConsumer consumer = new DefaultConsumer(channel){
            //回调方法，当收到消息后，会自动执行该方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {

                System.out.println("接收到的消息为：" + new String(body, "utf-8"));
                System.out.println("将日志信息打印到控制台.....");
            }
        };

        channel.basicConsume(queue1Name,true,consumer);
        //不关闭资源，应该一直监听消息
        //channel.close();
        //connection.close();
    }
}

```

#### 3）消费者2

```java
package com.woniu.comsumer;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Consumer_PubSub2 {
    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("woniu");
        factory.setPassword("woniu");
        factory.setPort(5672);
        factory.setHost("192.168.128.129");

        //虚拟主机名
        factory.setVirtualHost("/woniu");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        //不需要创建队列了，只需要绑定要监听的队列即可
        String queue2Name = "test_fanout_queue2";
        DefaultConsumer consumer = new DefaultConsumer(channel){
            //回调方法，当收到消息后，会自动执行该方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {

                System.out.println("接收到的消息为：" + new String(body, "utf-8"));
                System.out.println("将日志信息保存到数据库.....");
            }
        };

        channel.basicConsume(queue2Name,true,consumer);
        //不关闭资源，应该一直监听消息
        //channel.close();
        //connection.close();
    }
}

```


### 4.3.3. 测试

启动所有消费者，然后使用生产者发送消息；在每个消费者对应的控制台可以查看到生产者发送的所有消息；到达**广播**的效果。

在执行完测试代码后，其实到RabbitMQ的管理后台找到`Exchanges`选项卡，点击 `fanout_exchange` 的交换机，可以查看到如下的绑定：

![1556015006220](assets/1556015006220.png)

### 4.3.4. 小结

交换机需要与队列进行绑定，绑定之后；队列1和队列2都能收到交换机的消息，消费者1监听队列1，消费者2监听队列2，一个消息可以被多个消费者都收到。至于收到消息怎么处理，那就是消费者自己的事情了。



**发布订阅模式与工作队列模式的区别**

1、工作队列模式不用定义交换机，而发布/订阅模式需要定义交换机。 

2、发布/订阅模式的生产方是面向交换机发送消息，工作队列模式的生产方是面向队列发送消息(底层使用默认交换机)。

3、发布/订阅模式需要设置队列和交换机的绑定，工作队列模式不需要设置，实际上工作队列模式会将队列绑 定到默认的交换机 。



## 4.4. Routing路由模式

### 4.4.1. 模式说明

路由模式特点：

- 队列与交换机的绑定，不能是任意绑定了，而是要指定一个`RoutingKey`（路由key）
- 消息的发送方在 向 Exchange发送消息时，也必须指定消息的 `RoutingKey`。
- Exchange不再把消息交给每一个绑定的队列，而是根据消息的`Routing Key`进行判断，只有队列的`Routingkey`与消息的 `Routing key`完全一致，才会接收到消息

![1556029284397](assets/1556029284397.png) 

图解：

- P：生产者，向Exchange发送消息，发送消息时，会指定一个routing key。
- X：Exchange（交换机），接收生产者的消息，然后把消息递交给 与routing key完全匹配的队列
- C1：消费者，其所在队列指定了需要routing key 为 error 的消息
- C2：消费者，其所在队列指定了需要routing key 为 info、error、warning 的消息



### 4.4.2. 代码

在编码上与 `Publish/Subscribe发布与订阅模式` 的区别是交换机的类型为：Direct，还有队列绑定交换机的时候需要指定routing key。

#### 1）生产者

```java
package com.woniu.producer;

import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Producer_Routing {

    public static void main(String[] args) throws IOException, TimeoutException {
        //1、创建连接工程
        ConnectionFactory factory = new ConnectionFactory();
        //2、设置参数
        factory.setHost("192.168.128.129");
        factory.setVirtualHost("/woniu");//虚拟机 默认值/
        factory.setPort(5672); //端口  默认值 5672
        factory.setUsername("woniu");//用户名 默认 guest
        factory.setPassword("woniu");//密码 默认值 guest
        //3. 创建连接 Connection
        Connection connection = factory.newConnection();
        //4. 创建Channel
        Channel channel = connection.createChannel();
        //5、创建交换机
        /*
        exchangeDeclare(String exchange, BuiltinExchangeType type, boolean durable,
                            boolean autoDelete, boolean internal, Map<String, Object> arguments)
        参数说明
            1、exchange 交换机名称
            2、type 交换机的类型
                DIRECT("direct"), 定向
                FANOUT("fanout"), 扇形（广播） 发送消息到每一个与之绑定队列
                TOPIC("topic"), 通配符的方式
                HEADERS("headers"); 参数匹配  实际开发中使用不多
            3、durable 是否持久化
            4、autoDelete 是否自动删除
            5、internal  内部使用，一般为false
            6、arguments：参数
         */
        String exchange_name = "test_direct";
        channel.exchangeDeclare(exchange_name, BuiltinExchangeType.DIRECT,false,false,null);
        //6、创建队列
        String queue1Name = "test_direct_queue1";
        String queue2Name = "test_direct_queue2";
        channel.queueDeclare(queue1Name,true,false,false,null);
        channel.queueDeclare(queue2Name,true,false,false,null);
        //7、交换机和队列的绑定
        /*
        queueBind(String queue, String exchange, String routingKey)
        参数：
            1、queue  队列名称
            2、exchange 交换机名称
            3、routingKey 路由的键，绑定规则，如果交换机的类型为fanout，routingKey设置为“”
       */
        //队列1绑定 error
        channel.queueBind(queue1Name,exchange_name,"error");

        //队列2绑定 info  error  warning
        channel.queueBind(queue2Name,exchange_name,"warning");
        channel.queueBind(queue2Name,exchange_name,"error");
        channel.queueBind(queue2Name,exchange_name,"info");
        //8、发送消息
        String message = "日志消息，虎大调用了findall方法...日志级别：error...";
        /*
        basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body)
        参数说明
            1、exchange 交换机名称
            2、routingKey “”  （如果交换机类型是fanout，则routingKey为“”）
            3、props 基本属性配置
            4、body  要发送的数据
         */
        channel.basicPublish(exchange_name,"error",null,message.getBytes());
        //9.释放资源
        channel.close();
        connection.close();
    }
}

```
#### 2）消费者1

```java
package com.woniu.comsumer;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Consumer_Routing1 {
    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("woniu");
        factory.setPassword("woniu");
        factory.setPort(5672);
        factory.setHost("192.168.128.129");

        //虚拟主机名
        factory.setVirtualHost("/woniu");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        //不需要创建队列了，只需要绑定要监听的队列即可
        String queue1Name = "test_direct_queue1";

        DefaultConsumer consumer = new DefaultConsumer(channel){
            //回调方法，当收到消息后，会自动执行该方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {

                System.out.println("接收到的消息为：" + new String(body, "utf-8"));
                System.out.println("将日志信息保存到数据库....");
            }
        };

        channel.basicConsume(queue1Name,true,consumer);
        //不关闭资源，应该一直监听消息
        //channel.close();
        //connection.close();
    }
}

```
#### 3）消费者2

```java
package com.woniu.comsumer;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Consumer_Routing2 {
    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("woniu");
        factory.setPassword("woniu");
        factory.setPort(5672);
        factory.setHost("192.168.128.129");

        //虚拟主机名
        factory.setVirtualHost("/woniu");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        //不需要创建队列了，只需要绑定要监听的队列即可
        String queue2Name = "test_direct_queue2";

        DefaultConsumer consumer = new DefaultConsumer(channel){
            //回调方法，当收到消息后，会自动执行该方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {

                System.out.println("接收到的消息为：" + new String(body, "utf-8"));
                System.out.println("将日志打印到控制台....");
            }
        };

        channel.basicConsume(queue2Name,true,consumer);
        //不关闭资源，应该一直监听消息
        //channel.close();
        //connection.close();
    }
}

```


### 4.4.3. 测试

启动所有消费者，然后使用生产者发送消息；在消费者对应的控制台可以查看到生产者发送对应routing key对应队列的消息；到达**按照需要接收**的效果。

在执行完测试代码后，其实到RabbitMQ的管理后台找到`Exchanges`选项卡，点击 `direct_exchange` 的交换机，可以查看到如下的绑定：

### 4.4.4. 小结

Routing模式要求队列在绑定交换机时要指定routing key，消息会转发到符合routing key的队列。



## 4.5. Topics通配符模式

### 4.5.1. 模式说明

`Topic`类型与`Direct`相比，都是可以根据`RoutingKey`把消息路由到不同的队列。只不过`Topic`类型`Exchange`可以让队列在绑定`Routing key` 的时候**使用通配符**！



`Routingkey` 一般都是有一个或多个单词组成，多个单词之间以”.”分割，例如： `item.insert`

 通配符规则：

`#`：匹配一个或多个词

`*`：匹配不多不少恰好1个词

举例：

`item.#`：能够匹配`item.insert.abc` 或者 `item.insert`

`item.*`：只能匹配`item.insert`

![1556031362048](assets/1556031362048.png) 



![1556031519931](assets/1556031519931.png) 

图解：

- 红色Queue：绑定的是`usa.#` ，因此凡是以 `usa.`开头的`routing key` 都会被匹配到
- 黄色Queue：绑定的是`#.news` ，因此凡是以 `.news`结尾的 `routing key` 都会被匹配



### 4.5.2. 代码

#### 1）生产者

使用topic类型的Exchange，发送消息的routing key有3种： `item.insert`、`item.update`、`item.delete`：

```java
package com.woniu.producer;

import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Producer_Topic {

    public static void main(String[] args) throws IOException, TimeoutException {
        //1、创建连接工程
        ConnectionFactory factory = new ConnectionFactory();
        //2、设置参数
        factory.setHost("192.168.128.129");
        factory.setVirtualHost("/woniu");//虚拟机 默认值/
        factory.setPort(5672); //端口  默认值 5672
        factory.setUsername("woniu");//用户名 默认 guest
        factory.setPassword("woniu");//密码 默认值 guest
        //3. 创建连接 Connection
        Connection connection = factory.newConnection();
        //4. 创建Channel
        Channel channel = connection.createChannel();
        //5、创建交换机
        /*
        exchangeDeclare(String exchange, BuiltinExchangeType type, boolean durable,
                            boolean autoDelete, boolean internal, Map<String, Object> arguments)
        参数说明
            1、exchange 交换机名称
            2、type 交换机的类型
                DIRECT("direct"), 定向
                FANOUT("fanout"), 扇形（广播） 发送消息到每一个与之绑定队列
                TOPIC("topic"), 通配符的方式
                HEADERS("headers"); 参数匹配  实际开发中使用不多
            3、durable 是否持久化
            4、autoDelete 是否自动删除
            5、internal  内部使用，一般为false
            6、arguments：参数
         */
        String exchange_name = "test_topic";
        channel.exchangeDeclare(exchange_name, BuiltinExchangeType.TOPIC,false,false,null);
        //6、创建队列
        String queue1Name = "test_topic_queue1";
        String queue2Name = "test_topic_queue2";
        channel.queueDeclare(queue1Name,true,false,false,null);
        channel.queueDeclare(queue2Name,true,false,false,null);
        //7、交换机和队列的绑定
        /*
        queueBind(String queue, String exchange, String routingKey)
        参数：
            1、queue  队列名称
            2、exchange 交换机名称
            3、routingKey 路由的键，绑定规则，如果交换机的类型为fanout，routingKey设置为“”
       */
        channel.queueBind(queue1Name,exchange_name,"#.error");
        channel.queueBind(queue1Name,exchange_name,"order.*");

        channel.queueBind(queue2Name,exchange_name,"*.*");

        //8、发送消息
        String message = "日志消息，虎大调用了findall方法...日志级别：error...";
        /*
        basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body)
        参数说明
            1、exchange 交换机名称
            2、routingKey “”  （如果交换机类型是fanout，则routingKey为“”）
            3、props 基本属性配置
            4、body  要发送的数据
         */
        channel.basicPublish(exchange_name,"goods.xxx",null,message.getBytes());
        //9.释放资源
        channel.close();
        connection.close();
    }
}

```
#### 2）消费者1

接收两种类型的消息：更新商品和删除商品

```java
package com.woniu.comsumer;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Consumer_Topic1 {
    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("woniu");
        factory.setPassword("woniu");
        factory.setPort(5672);
        factory.setHost("192.168.128.129");

        //虚拟主机名
        factory.setVirtualHost("/woniu");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        //不需要创建队列了，只需要绑定要监听的队列即可
        String queue1Name = "test_topic_queue1";

        DefaultConsumer consumer = new DefaultConsumer(channel){
            //回调方法，当收到消息后，会自动执行该方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {

                System.out.println("接收到的消息为：" + new String(body, "utf-8"));
                System.out.println("将日志信息保存到数据库....");
            }
        };

        channel.basicConsume(queue1Name,true,consumer);
        //不关闭资源，应该一直监听消息
        //channel.close();
        //connection.close();
    }
}

```
#### 3）消费者2

接收所有类型的消息：新增商品，更新商品和删除商品。

```java
package com.woniu.comsumer;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Consumer_Topic2 {
    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setUsername("woniu");
        factory.setPassword("woniu");
        factory.setPort(5672);
        factory.setHost("192.168.128.129");

        //虚拟主机名
        factory.setVirtualHost("/woniu");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        //不需要创建队列了，只需要绑定要监听的队列即可
        String queue2Name = "test_topic_queue2";

        DefaultConsumer consumer = new DefaultConsumer(channel){
            //回调方法，当收到消息后，会自动执行该方法
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {

                System.out.println("接收到的消息为：" + new String(body, "utf-8"));
                System.out.println("将日志信息打印到控制台....");
            }
        };

        channel.basicConsume(queue2Name,true,consumer);
        //不关闭资源，应该一直监听消息
        //channel.close();
        //connection.close();
    }
}

```

### 4.5.3. 测试

启动所有消费者，然后使用生产者发送消息；在消费者对应的控制台可以查看到生产者发送对应routing key对应队列的消息；到达**按照需要接收**的效果；并且这些routing key可以使用通配符。



### 4.5.4. 小结

Topic主题模式可以实现 `Publish/Subscribe发布与订阅模式` 和 ` Routing路由模式` 的功能；只是Topic在配置routing key 的时候可以使用通配符，显得更加灵活。

## 4.6. 模式总结

RabbitMQ工作模式：
**1、简单模式 HelloWorld**
一个生产者、一个消费者，不需要设置交换机（使用默认的交换机）

**2、工作队列模式 Work Queue**
一个生产者、多个消费者（竞争关系），不需要设置交换机（使用默认的交换机）

**3、发布订阅模式 Publish/subscribe**
需要设置类型为fanout的交换机，并且交换机和队列进行绑定，当发送消息到交换机后，交换机会将消息发送到绑定的队列

**4、路由模式 Routing**
需要设置类型为direct的交换机，交换机和队列进行绑定，并且指定routing key，当发送消息到交换机后，交换机会根据routing key将消息发送到对应的队列

**5、通配符模式 Topic**
需要设置类型为topic的交换机，交换机和队列进行绑定，并且指定通配符方式的routing key，当发送消息到交换机后，交换机会根据routing key将消息发送到对应的队列



# 5. Spring 整合RabbitMQ

## 5.1. 搭建生产者工程

### 5.1.1. 创建工程  

略

### 5.1.2. 添加依赖

修改pom.xml文件内容为如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.woniu</groupId>
    <artifactId>spring-rabbitmq-producer</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.1.7.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-rabbit</artifactId>
            <version>2.1.8.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>5.1.7.RELEASE</version>
        </dependency>
    </dependencies>

</project>
```



### 5.1.3. 配置整合

1. 创建`spring-rabbitmq-producer\src\main\resources\properties\rabbitmq.properties`连接参数等配置文件；

```properties
rabbitmq.host=192.168.128.129
rabbitmq.port=5672
rabbitmq.username=woniu
rabbitmq.password=woniu
rabbitmq.virtual-host=/woniu
```



2. 创建 `spring-rabbitmq-producer\src\main\resources\spring\spring-rabbitmq.xml` 整合配置文件；

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">
    <!--加载配置文件-->
    <context:property-placeholder location="classpath:properties/rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>
    <!--定义管理交换机、队列-->
    <rabbit:admin connection-factory="connectionFactory"/>

    <!--定义持久化队列，不存在则自动创建；不绑定到交换机则绑定到默认交换机
    默认交换机类型为direct，名字为：""，路由键为队列的名称
    -->
    <rabbit:queue id="spring_queue" name="spring_queue" auto-declare="true"/>

    <!-- ~~~~~~~~~~~~~~~~~~~~~~~~~~~~广播；所有队列都能收到消息~~~~~~~~~~~~~~~~~~~~~~~~~~~~ -->
    <!--定义广播交换机中的持久化队列，不存在则自动创建-->
    <rabbit:queue id="spring_fanout_queue_1" name="spring_fanout_queue_1" auto-declare="true"/>

    <!--定义广播交换机中的持久化队列，不存在则自动创建-->
    <rabbit:queue id="spring_fanout_queue_2" name="spring_fanout_queue_2" auto-declare="true"/>

    <!--定义广播类型交换机；并绑定上述两个队列-->
    <rabbit:fanout-exchange id="spring_fanout_exchange" name="spring_fanout_exchange" auto-declare="true">
        <rabbit:bindings>
            <rabbit:binding queue="spring_fanout_queue_1"/>
            <rabbit:binding queue="spring_fanout_queue_2"/>
        </rabbit:bindings>
    </rabbit:fanout-exchange>

    <!-- ~~~~~~~~~~~~~~~~~~~~~~~~~~~~通配符；*匹配一个单词，#匹配多个单词 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~ -->
    <!--定义广播交换机中的持久化队列，不存在则自动创建-->
    <rabbit:queue id="spring_topic_queue_star" name="spring_topic_queue_star" auto-declare="true"/>
    <!--定义广播交换机中的持久化队列，不存在则自动创建-->
    <rabbit:queue id="spring_topic_queue_well" name="spring_topic_queue_well" auto-declare="true"/>
    <!--定义广播交换机中的持久化队列，不存在则自动创建-->
    <rabbit:queue id="spring_topic_queue_well2" name="spring_topic_queue_well2" auto-declare="true"/>

    <rabbit:topic-exchange id="spring_topic_exchange" name="spring_topic_exchange" auto-declare="true">
        <rabbit:bindings>
            <rabbit:binding pattern="woniu.*" queue="spring_topic_queue_star"/>
            <rabbit:binding pattern="woniu.#" queue="spring_topic_queue_well"/>
            <rabbit:binding pattern="woniu.#" queue="spring_topic_queue_well2"/>
        </rabbit:bindings>
    </rabbit:topic-exchange>

    <!--定义rabbitTemplate对象操作可以在代码中方便发送消息-->
    <rabbit:template id="rabbitTemplate" connection-factory="connectionFactory"/>
</beans>
```



### 5.1.4. 发送消息

创建测试文件 `spring-rabbitmq-producer\src\test\java\com\woniu\rabbitmq\ProducerTest.java`

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring/spring-rabbitmq.xml")
public class ProducerTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 只发队列消息
     * 默认交换机类型为 direct
     * 交换机的名称为空，路由键为队列的名称
     */
    @Test
    public void queueTest(){
        //路由键与队列同名
        rabbitTemplate.convertAndSend("spring_queue", "只发队列spring_queue的消息。");
    }

    /**
     * 发送广播
     * 交换机类型为 fanout
     * 绑定到该交换机的所有队列都能够收到消息
     */
    @Test
    public void fanoutTest(){
        /**
         * 参数1：交换机名称
         * 参数2：路由键名（广播设置为空）
         * 参数3：发送的消息内容
         */
        rabbitTemplate.convertAndSend("spring_fanout_exchange", "", "发送到spring_fanout_exchange交换机的广播消息");
    }

    /**
     * 通配符
     * 交换机类型为 topic
     * 匹配路由键的通配符，*表示一个单词，#表示多个单词
     * 绑定到该交换机的匹配队列能够收到对应消息
     */
    @Test
    public void topicTest(){
        /**
         * 参数1：交换机名称
         * 参数2：路由键名
         * 参数3：发送的消息内容
         */
        rabbitTemplate.convertAndSend("spring_topic_exchange", "woniu.bj", "发送到spring_topic_exchange交换机woniu.bj的消息");
        rabbitTemplate.convertAndSend("spring_topic_exchange", "woniu.bj.1", "发送到spring_topic_exchange交换机woniu.bj.1的消息");
        rabbitTemplate.convertAndSend("spring_topic_exchange", "woniu.bj.2", "发送到spring_topic_exchange交换机woniu.bj.2的消息");
        rabbitTemplate.convertAndSend("spring_topic_exchange", "woniu.cn", "发送到spring_topic_exchange交换机woniu.cn的消息");
    }
}

```



## 5.2. 搭建消费者工程

### 5.2.1. 创建工程

略

### 5.2.2. 添加依赖
修改pom.xml文件内容为如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.woniu</groupId>
    <artifactId>spring-rabbitmq-consumer</artifactId>
    <version>1.0-SNAPSHOT</version>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.2.10.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-rabbit</artifactId>
            <version>2.1.8.RELEASE</version>
        </dependency>
        
          <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>5.2.10.RELEASE</version>
        </dependency>
    </dependencies>


</project>
```

### 5.2.3. 配置整合
1. 创建`spring-rabbitmq-consumer\src\main\resources\properties\rabbitmq.properties`连接参数等配置文件；

```properties
rabbitmq.host=192.168.128.129
rabbitmq.port=5672
rabbitmq.username=woniu
rabbitmq.password=woniu
rabbitmq.virtual-host=/woniu
```



2. 创建 `spring-rabbitmq-consumer\src\main\resources\spring\spring-rabbitmq.xml` 整合配置文件；

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/rabbit
       http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">
    <!--加载配置文件-->
    <context:property-placeholder location="classpath:properties/rabbitmq.properties"/>

    <!-- 定义rabbitmq connectionFactory -->
    <rabbit:connection-factory id="connectionFactory" host="${rabbitmq.host}"
                               port="${rabbitmq.port}"
                               username="${rabbitmq.username}"
                               password="${rabbitmq.password}"
                               virtual-host="${rabbitmq.virtual-host}"/>

    <bean id="springQueueListener" class="com.woniu.rabbitmq.listener.SpringQueueListener"/>
    <bean id="fanoutListener1" class="com.woniu.rabbitmq.listener.FanoutListener1"/>
    <bean id="fanoutListener2" class="com.woniu.rabbitmq.listener.FanoutListener2"/>
    <bean id="topicListenerStar" class="com.woniu.rabbitmq.listener.TopicListenerStar"/>
    <bean id="topicListenerWell" class="com.woniu.rabbitmq.listener.TopicListenerWell"/>
    <bean id="topicListenerWell2" class="com.woniu.rabbitmq.listener.TopicListenerWell2"/>

    <rabbit:listener-container connection-factory="connectionFactory" auto-declare="true">
        <rabbit:listener ref="springQueueListener" queue-names="spring_queue"/>
        <rabbit:listener ref="fanoutListener1" queue-names="spring_fanout_queue_1"/>
        <rabbit:listener ref="fanoutListener2" queue-names="spring_fanout_queue_2"/>
        <rabbit:listener ref="topicListenerStar" queue-names="spring_topic_queue_star"/>
        <rabbit:listener ref="topicListenerWell" queue-names="spring_topic_queue_well"/>
        <rabbit:listener ref="topicListenerWell2" queue-names="spring_topic_queue_well2"/>
    </rabbit:listener-container>
</beans>
```

### 5.2.4. 消息监听器

#### 1）队列监听器

创建 `spring-rabbitmq-consumer\src\main\java\com\woniu\rabbitmq\listener\SpringQueueListener.java`

```java
public class SpringQueueListener implements MessageListener {
    public void onMessage(Message message) {
        try {
            String msg = new String(message.getBody(), "utf-8");

            System.out.printf("接收路由名称为：%s，路由键为：%s，队列名为：%s的消息：%s \n",
                    message.getMessageProperties().getReceivedExchange(),
                    message.getMessageProperties().getReceivedRoutingKey(),
                    message.getMessageProperties().getConsumerQueue(),
                    msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

创建测试类

~~~
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring/spring-rabbitmq.xml")
public class ConsumerTest {

    @Test
    public void test1(){
        boolean flag = true;
        while (true){

        }
    }
}
~~~



#### 2）广播监听器1

创建 `spring-rabbitmq-consumer\src\main\java\com\\rabbitmq\listener\FanoutListener1.java`

```java
public class FanoutListener1 implements MessageListener {
    public void onMessage(Message message) {
        try {
            String msg = new String(message.getBody(), "utf-8");

            System.out.printf("广播监听器1：接收路由名称为：%s，路由键为：%s，队列名为：%s的消息：%s \n",
                    message.getMessageProperties().getReceivedExchange(),
                    message.getMessageProperties().getReceivedRoutingKey(),
                    message.getMessageProperties().getConsumerQueue(),
                    msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

#### 3）广播监听器2
创建 `spring-rabbitmq-consumer\src\main\java\com\\rabbitmq\listener\FanoutListener2.java`

```java
public class FanoutListener2 implements MessageListener {
    public void onMessage(Message message) {
        try {
            String msg = new String(message.getBody(), "utf-8");

            System.out.printf("广播监听器2：接收路由名称为：%s，路由键为：%s，队列名为：%s的消息：%s \n",
                    message.getMessageProperties().getReceivedExchange(),
                    message.getMessageProperties().getReceivedRoutingKey(),
                    message.getMessageProperties().getConsumerQueue(),
                    msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### 4）星号通配符监听器
创建 `spring-rabbitmq-consumer\src\main\java\com\\rabbitmq\listener\TopicListenerStar.java`

```java
public class TopicListenerStar implements MessageListener {
    public void onMessage(Message message) {
        try {
            String msg = new String(message.getBody(), "utf-8");

            System.out.printf("通配符*监听器：接收路由名称为：%s，路由键为：%s，队列名为：%s的消息：%s \n",
                    message.getMessageProperties().getReceivedExchange(),
                    message.getMessageProperties().getReceivedRoutingKey(),
                    message.getMessageProperties().getConsumerQueue(),
                    msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

#### 5）井号通配符监听器
创建 `spring-rabbitmq-consumer\src\main\java\com\\rabbitmq\listener\TopicListenerWell.java`

```java
public class TopicListenerWell implements MessageListener {
    public void onMessage(Message message) {
        try {
            String msg = new String(message.getBody(), "utf-8");

            System.out.printf("通配符#监听器：接收路由名称为：%s，路由键为：%s，队列名为：%s的消息：%s \n",
                    message.getMessageProperties().getReceivedExchange(),
                    message.getMessageProperties().getReceivedRoutingKey(),
                    message.getMessageProperties().getConsumerQueue(),
                    msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

#### 6）井号通配符监听器2
创建 `spring-rabbitmq-consumer\src\main\java\com\woniu\rabbitmq\listener\TopicListenerWell2.java`

```java
public class TopicListenerWell2 implements MessageListener {
    public void onMessage(Message message) {
        try {
            String msg = new String(message.getBody(), "utf-8");

            System.out.printf("通配符#监听器2：接收路由名称为：%s，路由键为：%s，队列名为：%s的消息：%s \n",
                    message.getMessageProperties().getReceivedExchange(),
                    message.getMessageProperties().getReceivedRoutingKey(),
                    message.getMessageProperties().getConsumerQueue(),
                    msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```



# 6. Spring Boot整合RabbitMQ



## 6.1. 简介

在Spring项目中，可以使用Spring-Rabbit去操作RabbitMQ
https://github.com/spring-projects/spring-amqp

尤其是在spring boot项目中只需要引入对应的amqp启动器依赖即可，方便的使用RabbitTemplate发送消息，使用注解接收消息。



*一般在开发过程中*：

**生产者工程：**

1. application.yml文件配置RabbitMQ相关信息；
2. 在生产者工程中编写配置类，用于创建交换机和队列，并进行绑定

3. 注入RabbitTemplate对象，通过RabbitTemplate对象发送消息到交换机



**消费者工程：**

1. application.yml文件配置RabbitMQ相关信息

2. 创建消息处理类，用于接收队列中的消息并进行处理



## 5.2. 搭建生产者工程

### 5.2.1. 创建工程

创建生产者工程springboot-rabbitmq-producer

![1556072078816](assets/1558321303612.png)

![1556072084653](assets/1558321417385.png)

### 5.2.2. 添加依赖

修改pom.xml文件内容为如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.4.RELEASE</version>
    </parent>
    <groupId>com.woniu</groupId>
    <artifactId>springboot-rabbitmq-producer</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
    </dependencies>
</project>
```



### 5.2.3. 启动类

```java
package com.woniu.rabbitmq;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ProducerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProducerApplication.class);
    }
}

```



### 5.2.4. 配置RabbitMQ

#### 1）配置文件

创建application.yml，内容如下：

```yml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    virtual-host: /woniu
    username: woniu
    password: woniu
```



#### 2）绑定交换机和队列

创建RabbitMQ队列与交换机绑定的配置类com..rabbitmq.config.RabbitMQConfig

```java
package com.woniu.rabbitmq.config;

import org.springframework.amqp.core.*;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {
    //交换机名称
    public static final String ITEM_TOPIC_EXCHANGE = "item_topic_exchange";
    //队列名称
    public static final String ITEM_QUEUE = "item_queue";

    //声明交换机
    @Bean("itemTopicExchange")
    public Exchange topicExchange(){
        return ExchangeBuilder.topicExchange(ITEM_TOPIC_EXCHANGE).durable(true).build();
    }

    //声明队列
    @Bean("itemQueue")
    public Queue itemQueue(){
        return QueueBuilder.durable(ITEM_QUEUE).build();
    }

    //绑定队列和交换机
    @Bean
    public Binding itemQueueExchange(@Qualifier("itemQueue") Queue queue,
                                     @Qualifier("itemTopicExchange") Exchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with("item.#").noargs();
    }

}
```

创建测试类，发送消息到消息队列

~~~
import com.woniu.rabbitmq.ProducerApplication;
import com.woniu.rabbitmq.config.RabbitMqConfig;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.test.context.junit4.SpringRunner;

@SpringBootTest(classes = ProducerApplication.class)
@RunWith(SpringRunner.class)
public class ProducerTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    public void testSend(){
        rabbitTemplate.convertAndSend(RabbitMqConfig.ITEM_TOPIC_EXCHANGE,
                "item.add","新增商品");

    }


}

~~~



## 5.3. 搭建消费者工程

### 5.3.1. 创建工程

创建消费者工程springboot-rabbitmq-consumer

### 5.3.2. 添加依赖

修改pom.xml文件内容为如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.4.RELEASE</version>
    </parent>
    <groupId>com.</groupId>
    <artifactId>springboot-rabbitmq-consumer</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
    </dependencies>

</project>
```



### 5.3.3. 启动类

```java
package com..rabbitmq;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class);
    }
}
```



### 5.3.4. 配置RabbitMQ

创建application.yml，内容如下：

```yml
spring:
  rabbitmq:
    host: 192.168.128.129
    port: 5672
    virtual-host: /woniu
    username: woniu
    password: woniu
```



### 5.3.5. 消息监听处理类

编写消息监听器com.woniu.rabbitmq.listener.MyListener

```java
package com..rabbitmq.listener;

import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class MyListener {

    /**
     * 监听某个队列的消息
     * @param message 接收到的消息
     */
    @RabbitListener(queues = "item_queue")
    public void myListener1(String message){
        System.out.println("消费者接收到的消息为：" + message);
    }
}

```



## 5.4. 测试

在生产者工程springboot-rabbitmq-producer中创建测试类，发送消息：

```java
package com..rabbitmq;

import com..rabbitmq.config.RabbitMQConfig;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@SpringBootTest(classes = ProducerApplication.class)
@RunWith(SpringRunner.class)
public class RabbitMQTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    public void test(){
        rabbitTemplate.convertAndSend(RabbitMQConfig.ITEM_TOPIC_EXCHANGE, "item.insert", "商品新增，routing key 为item.insert");
        rabbitTemplate.convertAndSend(RabbitMQConfig.ITEM_TOPIC_EXCHANGE, "item.update", "商品修改，routing key 为item.update");
        rabbitTemplate.convertAndSend(RabbitMQConfig.ITEM_TOPIC_EXCHANGE, "item.delete", "商品删除，routing key 为item.delete");
    }
}
```



先运行上述测试程序（交换机和队列才能先被声明和绑定），然后启动消费者；在消费者工程springboot-rabbitmq-consumer中控制台查看是否接收到对应消息。



另外；也可以在RabbitMQ的管理控制台中查看到交换机与队列的绑定：

![1556074827222](assets/1556074827222.png)