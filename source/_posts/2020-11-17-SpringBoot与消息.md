---
title: 07_SpringBoot与消息
date: 2020-11-17 18:12:48
categories:
- SpringBoot
tags:
- SpringBoot
- 消息
---

<center><font size=4 color="red">07_SpringBoot与消息</font></center>

<!--more-->

# SpringBoot与消息

消息队列作用：异步处理、应用解耦、流量削锋

## JMS&AMQP

**JMS：**Java消息服务，它是基于Java代理的规范，而ActiveMQ是由JMS实现的，这就意味着ActiveMQ只能使用Java代码来操作。ActiveMQ在我之前分布式笔记中有过介绍。

**AMQP：**高级消息队列，兼容JMS，RabbitMQ是由AMQP实现的，具有跨语言性，支持其它的语言来操作，此次介绍Springboot与消息，用的消息队列就是？RabbitMQ

|              | JMS              | AMQP               |
| ------------ | ---------------- | ------------------ |
| 定义         | Java api         | 网络线级协议       |
| 跨语言       | 否               | 是                 |
| 跨平台       | 否               | 是                 |
| Model        | 提供两种消息模式 | 提供五种消息模式   |
| 支持消息类型 | XXXMessage       | byte[]             |
| 综合评价     | 适用于Java体系   | 适用于各种语言体系 |

JMS的model提供的两种消息模式：

* Peer-2-Peer：点对点模式
* Pub/Sub：发布订阅模式

AMQP提供的五种消息模式：

* direct exchange
* fanout exchange
* topic exchange
* headers exchange(不常用)
* system exchange(不常用)

## Spring的支持

spring对JMS、AMQP都提供了支持

其中需要ConnectionFactory的实现来连接消息代理，如果是Springboot，直接在主配置文件中配置连接的主机、用户名、密码即可

|                      | Spring对JMS的支持    | Spring对AMQP的支持      |
| -------------------- | -------------------- | ----------------------- |
| 提供方式             | spring-jms           | spring-amqp             |
| 发送接收消息         | JmsTemplate          | RabbitTemplate          |
| 监听消息             | @JmsListener         | @RabbitListener         |
| 注解支持             | @EnableJms           | @EnableRabbit           |
| Springboot自动配置类 | JmsAutoConfiguration | RabbitAutoConfiguration |

## RabbitMQ简介

#### 核心概念

![](Rabbit组件.webp)

**Message：**消息，由消息头和消息体组成，消息体不透明，消息头由一系列可选的属性组成，这些属性主要包括routing-key（路由键）、priority（相对于其它消息的优先权）、delivery-mode（指出该消息可能需要持久化的属性）等。

**Publisher：**消息的生产者，也是一个向交换器发送消息的客户端应用程序

**Exchange：**交换器，用来接收生产者发送的消息，并将这些消息路由给服务器中的队列。Exchange有4种类型：

direct（默认） 、fanout 、topic 和headers ，不同类型的Exchange转发消息的策略有所区别。

**Queue：**消息队列，用来保存消息直到被消费者消费。它是消息的容器，也是消息的终点，一个消息可投入到一个或者多个队列。

**Binding：**绑定，用于消息队列和交换器之间的关联。一个绑定就是基于路由键将交换器与消息队列连接起来的路由规则。Exchange和Queue的绑定是多对多的关系，即一个Exchange可以绑定多个消息队列，一个消息队列也可以被多个Exchange绑定。

**Routing-key：**路由键，就是Exchange与Queue的绑定规则

**Connection：**网络连接。就是配置Rabbit服务器与程序进行的TCP连接。

**Channel：**信道。无论是发送消息到Rabbit服务器还是从Rabbit服务器上消费消息，都是通过信道完成的。完成过程如果都用TCP连接太耗费资源，所有我们只需要开辟一个TCP连接，然后在该连接上开辟多个信道进行通讯，它是建立在真是TCP连接上的虚拟连接。

**Consumer：**消息的消费者，表示一个从一个消息队列中取得消息的客户端应用程序。

**Virtual Host：**虚拟主机。标识一批交换机、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个vhost本质上就是一个mini版的RabbitMQ服务器，拥有自己的队列、交换器、绑定和权限机制。vhost是AMQP概念的基础，必须在链接时指定，RabbitMQ默认的vhost是 /。

**Broker：**标识消息队列服务器实体.

就核心概念和上面的图进行讲解整个流程：

1. Publisher发布消息到Broker，也就是Rabbit所在的服务器
2. 而Broker里可以定义多个Rabbit的Virtual Host，默认我们使用的vhost是 /，不过我们不在配置文件里指定Virtual Host，默认消息是发到路径为`/`的虚拟主机中
3. Virtual Host收到的消息会流入到服务器的Exchange，发送来的消息是需要指定具体发送到哪一个Exchange
4. Rabbit服务器上的Exchange和Queue是提前创建起来的，并且绑定规则也是提前绑定好的
5. 发送的消息会根据指定的具体Exchange进入到服务器上的Exchange中，然后根据携带的路由键进入该Exchange绑定好的Queue中
6. 最后等消费者通过信道消费掉Queue里的消息

#### Exchange类型

我们常用的RabbitMQ中的Exchange类型只要有3中：

* direct exchange
* fanout exchange
* topic exchange

首先明确一点：Exchange要在服务上创建，在创建时需要指定Exchange的类型，并且起一个Exchange的名称


![](direct.jpg)


**direct exchange**

默认的Exchange类型，也叫点对点模式，只能通过指定的具体路由键绑定到指定的具体Queue中

如上图所示：

我们先在Rabbit服务器上创建好1个Exchange交换器，3个queue队列，交换器名称是exchangedemo，并指定该交换器是direct类型的（如果不指定，默认就是direct类型），3个队列名称分别是queue1，queue2，queue3，且交换器与队列绑定的路由键Routing-key分别是dog、dog1、dog2，此时我们发布消息时，我们指定消息的Exchange名称为exchangedemo，路由键为dog（路由键必须是服务器上已经存在的路由键，否则找不到），然后我们发布的消息就可以流入到队列queue1中了。

**fanout exchange**

仍然以上图为例：

创建exchangedemo时，我们创建的交换器类型是fanout类型。发布消息时，我们指定消息的Exchange名称为exchangedemo，此时的路由键无论我们指定名称是什么，消息最终都会被绑定了exchangedemo的所有queue接收。也叫广播模式，当然也是处理最快的一个模式。

**topic exchange**

这种是通过模式匹配分配路由键属性的。具体操作下图分析

![](topic.jpg)

exchange.1通过key.#和#.key分别绑定queue1和queue2

exchange.2通过key1.*和key.#分别绑定queue1和queue3

符号#和\*都是通配符。#表示匹配0个、1个或多个单词，\*表示只匹配1个单词，每一个.后面就算一个单词，例如key.a表示key后面有1个单词；key.a.b表示key后面有2个单词

假如我们指定消息的Exchange名称为exchange1，此时的路由键无论我们指定名称是key.1，结果就会把数据存入到queue1队列中

假如我们指定消息的Exchange名称为exchange1，此时的路由键无论我们指定名称是key.key，结果就会把数据存入到queue1和queue2队列中

假如我们指定消息的Exchange名称为exchange2，此时的路由键无论我们指定名称是key1.a.b，因\*只表示一个单词，所有都匹配不上，队列中不会存数据

假如我们指定消息的Exchange名称为exchange2，此时的路由键无论我们指定名称是key1.a，queue1中会有数据

## Docker-Compose安装RabbitMQ

docker-compose.yml文件：

```yaml
version: '3'
services:
  rabbitmq:
    image: rabbitmq:3.8.3-management
    container_name: rabbitmq
    restart: always
    hostname: myRabbitmq
    ports:
      - 15672:15672
      - 5672:5672
    volumes:
      - ./data:/var/lib/rabbitmq
    environment:
      # 用来设置超级管理员的账号和密码，如果不设置，默认都是 guest
      - RABBITMQ_DEFAULT_USER=root
      - RABBITMQ_DEFAULT_PASS=123456
```

访问：ip:15672，用户名和密码是docker-compose文件中设置的root、123456

## Springboot引入RabbitMQ

pom依赖引入：

```xml
<!--rabbitmq-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

application.yml配置

```yaml
spring: 
  rabbitmq:
    host: 192.168.31.180
    port: 5672
    username: root
    password: 123456
    # 虚拟host，如果不设置或者设置为空默认就是"/"
    virtual-host:
```

测试发布消息

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootdemoApplicationTests {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    public void rabbitSend() {
        /**
         * exchange:交换器名称
         * routingKey：路由键
         * message：消息，可以自己定义，包括消息头和消息体
         * 一般我们不需要定义消息头，所以这个send方法我们一般不用
         */
        //rabbitTemplate.send(exchange,routingKey,message);

        //定义user对象消息进行发送
        User user=new User();
        user.setUserId(1);
        user.setUserName("zhangsan");
        user.setUserAge(17);
        user.setDate(new Date());
        rabbitTemplate.convertAndSend("demo","dog",user);
    }

}
```

执行程序后，在Rabbit的管理页面可以看到接收到的消息

![](rabbit-manager.jpg)

但是由图中：content_type:application/x-java-serialized-object可知序列化时是按照java的方式进行序列化的，我们可以将序列化方式转化为Json的序列化方式

增加配置文件：

com.hui.config.MyRabbitConfig

```java
package com.hui.config;


import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyRabbitConfig {

    /**
     * 将rabbit发布消息序列化到rabbit服务器的方式为json方式
     * @param connectionFactory
     * @return
     */
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
        return rabbitTemplate;
    }
}
```

重新运行上面发布消息的程序：

![](rabbit-json.jpg)

可以看到content_type:application/json

测试消费消息：

```java
@Test
public void rabbitReceive() {
    //根据队列名称消费消息
    User user = (User) rabbitTemplate.receiveAndConvert("queue1");
    System.out.println(user.toString());
}
```

#### 消息监听

我们在做订单系统和库存系统时，我们可以使用消息队列进行解耦，让RabbitMQ监听库存系统，我们让库存系统消息时刻监听，如果订单系统有消息发布到消息队列中，就进行消费，减少库存。这时我们就用到了消息监听，消息监听要满足两个注解条件：

* 主配置类加注解：@EnableRabbit
* 需要监听的方法上加注解：@RabbitListener  


当然@RabbitListener也 可以标注在类上面，需配合 @RabbitHandler 注解一起使用

@RabbitListener 标注在类上面表示当有收到消息的时候，就交给 @RabbitHandler 的方法处理，具体使用哪个方法处理，根据 MessageConverter 转换后的参数类型

如果使用了消息监听，需要增加消息转换器的配置，修改MyRabbitConfig配置如下：

com.hui.config.MyRabbitConfig

```java
package com.hui.config;


import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyRabbitConfig {

    /**
     * 将rabbit发布消息序列化到rabbit服务器的方式为json方式
     * @param connectionFactory
     * @return
     */
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
        return rabbitTemplate;
    }


    /**
     * 增加消息转换器，如果没有这个转换规则，被序列化为json的数据在消息监听获取的反序列数据会报错
     * @param objectMapper
     * @return
     */
    @Bean
    public MessageConverter messageConverter(ObjectMapper objectMapper){
        return new Jackson2JsonMessageConverter(objectMapper);
    }
}
```

设置监听：

主函数加上开启监听注解：

```java
@SpringBootApplication
@EnableRabbit
public class SpringbootdemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootdemoApplication.class, args);
    }
    
}
```

监听消费不是配置类，只要加上注解@RabbitListener即可。例如：

com.hui.service.RabbitListenerService

```java
@Service
public class RabbitListenerService {

    /**
     * 监听获取到的消息会直接由方法的参数进行接收
     * @param user
     * @return
     */
    @RabbitListener(queues = "queue1")
    public void getUser(User user){
        System.out.println(user.toString());
    }
}
```

这样就能起到监听的作用，只要队列queue1中有发布的消息，加上@RabbitListener注解的方法就能消费掉这个消息

#### AmqpAdmin

AmqpAdmin是RabbitMQ系统功能处理组件，该组件的作用是我们可以在程序中创建Exchange、Queue已经将Exchange和Queue进行绑定

**Exchange的创建**

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootdemoApplicationTests {

    @Autowired
    private AmqpAdmin amqpAdmin;

    /**
     * 创建交换器
     */
    @Test
    public void createExchange(){
        //构造DirectExchange类型交换器对象
        Exchange directExchange=new DirectExchange("directDemo");
        //构造创建FanoutExchange类型交换器对象
        Exchange fanoutExchange=new FanoutExchange("fanoutDemo");
        //构造TopicExchange类型交换器对象
        Exchange topicExchange=new TopicExchange("topicDemo");
        //创建DirectExchange类型交换器
        amqpAdmin.declareExchange(directExchange);
        System.out.println("创建Exchange完成。。。");
    }

}
```

**Queue的创建**

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootdemoApplicationTests {

    @Autowired
    private AmqpAdmin amqpAdmin;

    /**
     * 创建队列
     */
    @Test
    public void createQueue(){
        //构造队列对象
        Queue queue=new Queue("queueDemo");
        amqpAdmin.declareQueue(queue);
        System.out.println("创建Queue完成。。。");
    }

}
```

**绑定**

将Exchange与Queue进行绑定

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootdemoApplicationTests {

    @Autowired
    private AmqpAdmin amqpAdmin;

    /**
     * 创建绑定
     */
    @Test
    public void createBinding(){
        /**
         * 第一个参数destination：目的地，我们是交换器绑定队列，所以目的地是队列的名称。
         * 交换器也能绑定交换器，这时的目的地就是交换器的名称，下面的类型就是交换器EXCHANGE
         * 第二个参数destinationType：目的地类型，因为我们的目的地是队列，所以类型是QUEUE
         * 第三个参数exchange：交换器的名称
         * 第四个参数routingKey：绑定的路由键
         * 第五个参数arguments：Map类型的参数，暂时不知道有啥用，如果没有，就传null
         */
        Binding binding=new Binding("queueDemo", Binding.DestinationType.QUEUE,"directDemo","keyDemo",null);
        amqpAdmin.declareBinding(binding);
        System.out.println("创建Binding完成。。。");
    }

}
```

这样就能将交换器directDemo以路由键keyDemo的规则和队列queueDemo进行绑定了。


