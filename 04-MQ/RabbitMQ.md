# 一、初识 MQ

## 1、同步和异步通讯

微服务间通讯有同步和异步两种方式：

同步通讯：就像打电话，需要实时响应。

异步通讯：就像发邮件，不需要马上回复。

![image1](assets/image1.png)

两种方式各有优劣，打电话可以立即得到响应，但是你却不能跟多个人同时通话。发送邮件可以同时与多个人收发邮件，但是往往响应会有延迟。

### 1.1、同步通讯

我们之前学习的 Feign 调用就属于同步方式，虽然调用可以实时得到结果，但存在下面的问题：

![image2](assets/image2.png)

![image3](assets/image3.png)

总结：

> 同步调用的优点：
>
> - 时效性较强，可以立即得到结果
>
> 同步调用的问题：
>
> - 耦合度高
> - 性能和吞吐能力下降
> - 有额外的资源消耗
> - 有级联失败问题

### 1.2、异步通讯

异步调用则可以避免上述问题

异步调用常见实现就是事件驱动模式

![image4](assets/image4.png)

我们以购买商品为例，用户支付后需要调用订单服务完成订单状态修改，调用物流服务，从仓库分配响应的库存并准备发货。

在事件模式中，支付服务是事件发布者（publisher），在支付完成后只需要发布一个支付成功的事件（event），事件中带上订单 id。

订单服务和物流服务是事件订阅者（Consumer），订阅支付成功的事件，监听到事件后完成自己业务即可。

为了解除事件发布者与订阅者之间的耦合，两者并不是直接通信，而是有一个中间人（Broker）。发布者发布事件到 Broker，不关心谁来订阅事件。订阅者从 Broker 订阅事件，不关心谁发来的消息。

![image5](assets/image5.png)

Broker 是一个像数据总线一样的东西，所有的服务要接收数据和发送数据都发到这个总线上，这个总线就像协议一样，让服务间的通讯变得标准和可控。

**事件驱动优势**：

1. 服务解耦
   * 增加业务需求只需去订阅事件，取消业务只需取消订阅事件
2. 性能提升，吞吐量提高
   * 原本需要 500 ms，采用事件驱动后只需要 60 ms，因为订单、仓储、短信服务都与支付服务没关系了

![image6](assets/image6.png)

3. 服务没有强依赖，不担心级联失败问题
4. 流量削峰
   * 面对大量请求时，broker 起一个缓冲作用，高并发变成低并发

![image7](assets/image7.png)

总结：

> 异步通信的优点：
>
> - 吞吐量提升：无需等待订阅者处理完成，响应更快速
>
> - 故障隔离：服务没有直接调用，不存在级联失败问题
> - 调用间没有阻塞，不会造成无效的资源占用
> - 耦合度极低，每个服务都可以灵活插拔，可替换
> - 流量削峰：不管发布事件的流量波动多大，都由Broker接收，订阅者可以按照自己的速度去处理事件
>
> 异步通信的缺点：
>
> - 架构复杂了，业务没有明显的流程线，不好管理
> - 需要依赖于 Broker 的可靠、安全、性能

好在现在开源软件或云平台上 Broker 的软件是非常成熟的，比较常见的一种就是我们今天要学习的 MQ 技术。

## 2、技术对比

MQ，中文是消息队列（MessageQueue），字面来看就是存放消息的队列。也就是事件驱动架构中的 Broker。

比较常见的 MQ 实现：

- ActiveMQ
- RabbitMQ
- RocketMQ
- Kafka

几种常见 MQ 的对比：

|            |      **RabbitMQ**       |          **ActiveMQ**          | **RocketMQ** | **Kafka**  |
| :--------: | :---------------------: | :----------------------------: | :----------: | :--------: |
| 公司/社区  |         Rabbit          |             Apache             |     阿里     |   Apache   |
|  开发语言  |         Erlang          |              Java              |     Java     | Scala&Java |
|  协议支持  | AMQP，XMPP，SMTP，STOMP | OpenWire,STOMP，REST,XMPP,AMQP |  自定义协议  | 自定义协议 |
|   可用性   |           高            |              一般              |      高      |     高     |
| 单机吞吐量 |          一般           |               差               |      高      |   非常高   |
|  消息延迟  |         微秒级          |             毫秒级             |    毫秒级    |  毫秒以内  |
| 消息可靠性 |           高            |              一般              |      高      |    一般    |

追求可用性：Kafka、 RocketMQ 、RabbitMQ

追求可靠性：RabbitMQ、RocketMQ

追求吞吐能力：RocketMQ、Kafka

追求消息低延迟：RabbitMQ、Kafka

# 二、快速入门

RabbitMQ 是基于 Erlang 语言开发的开源消息通信中间件，官网地址：[https://www.rabbitmq.com/](https://www.rabbitmq.com/)

## 1、安装 RabbitMQ

单机部署 RabbitMQ，我们可以在 Centos7 虚拟机中使用 Docker 来安装。

### 1.1、下载镜像

方式一：在线拉取

``` sh
docker pull rabbitmq:3-management
```

方式二：将本地镜像包上传到虚拟机中后，使用命令加载镜像即可：

```sh
docker load -i mq.tar
```

### 1.2、安装 MQ

执行下面的命令来运行 MQ 容器：

```sh
docker run \
 -e RABBITMQ_DEFAULT_USER=itcast \
 -e RABBITMQ_DEFAULT_PASS=123321 \
 --name mq \
 --hostname mq1 \
 -p 15672:15672 \
 -p 5672:5672 \
 -d \
 rabbitmq:3-management
```



RabbitMQ 的基本结构：

![image8](assets/image8.png)

RabbitMQ 中的一些角色：

- publisher：生产者
- consumer：消费者
- exchange：交换机，负责消息路由
- queue：队列，存储消息
- virtualHost：虚拟主机，是对 queue、exchange 等资源的逻辑分组，隔离不同租户的 exchange、queue、消息
- channel：操作 MQ 的工具



## 2、RabbitMQ 消息模型

RabbitMQ 官方提供了 5 个不同的 Demo 示例，对应了不同的消息模型：

![image9](assets/image9.png)

## 3、导入 Demo 工程

导入课前资料提供的 mq-demo 工程，导入后可以看到结构如下：

![image10](assets/image10.png)

包括三部分：

- mq-demo：父工程，管理项目依赖
- publisher：消息的发送者
- consumer：消息的消费者

实现步骤：

* 运行 publisher 服务中的测试类 PublisherTest 中的测试方法 testSendMessage()

* 查看 RabbitMQ 控制台的消息
* 启动 consumer 服务，查看是否能接收消息

## 4、入门案例

简单队列模式的模型图：

 ![image11](assets/image11.png)

官方的 HelloWorld 是基于最基础的消息队列模型来实现的，只包括三个角色：

- publisher：消息发布者，将消息发送到队列 queue
- queue：消息队列，负责接受并缓存消息
- consumer：订阅队列，处理队列中的消息

### 4.1、publisher 实现

思路：

- 建立连接
- 创建 Channel
- 声明队列
- 发送消息
- 关闭连接和 channel

代码实现：

```java
package cn.itcast.mq.helloworld;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import org.junit.Test;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class PublisherTest {
    @Test
    public void testSendMessage() throws IOException, TimeoutException {
        // 1.建立连接
        ConnectionFactory factory = new ConnectionFactory();
        // 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
        factory.setHost("192.168.150.101");
        factory.setPort(5672);
        factory.setVirtualHost("/");
        factory.setUsername("itcast");
        factory.setPassword("123321");
        // 1.2.建立连接
        Connection connection = factory.newConnection();

        // 2.创建通道Channel
        Channel channel = connection.createChannel();

        // 3.创建队列
        String queueName = "simple.queue";
        channel.queueDeclare(queueName, false, false, false, null);

        // 4.发送消息
        String message = "hello, rabbitmq!";
        channel.basicPublish("", queueName, null, message.getBytes());
        System.out.println("发送消息成功：【" + message + "】");

        // 5.关闭通道和连接
        channel.close();
        connection.close();
    }
}
```

### 4.2、consumer 实现

代码思路：

- 建立连接
- 创建 Channel
- 声明队列
- 订阅消息

代码实现：

```java
package cn.itcast.mq.helloworld;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class ConsumerTest {
    public static void main(String[] args) throws IOException, TimeoutException {
        // 1.建立连接
        ConnectionFactory factory = new ConnectionFactory();
        // 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
        factory.setHost("192.168.150.101");
        factory.setPort(5672);
        factory.setVirtualHost("/");
        factory.setUsername("itcast");
        factory.setPassword("123321");
        // 1.2.建立连接
        Connection connection = factory.newConnection();

        // 2.创建通道Channel
        Channel channel = connection.createChannel();

        // 3.创建队列
        String queueName = "simple.queue";
        channel.queueDeclare(queueName, false, false, false, null);

        // 4.订阅消息
        channel.basicConsume(queueName, true, new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                // 5.处理消息
                String message = new String(body);
                System.out.println("接收到消息：【" + message + "】");
            }
        });
        System.out.println("等待接收消息。。。。");
    }
}
```

## 5、总结

基本消息队列的消息发送流程：

1. 建立 connection

2. 创建 channel

3. 利用 channel 声明队列

4. 利用 channel 向队列发送消息

基本消息队列的消息接收流程：

1. 建立 connection

2. 创建 channel

3. 利用 channel声明队列

4. 定义 consumer 的消费行为 handleDelivery()

5. 利用 channel 将消费者与队列绑定

# 三、SpringAMQP

SpringAMQP 是基于 RabbitMQ 封装的一套模板，并且还利用 SpringBoot 对其实现了自动装配，使用起来非常方便。

SpringAMQP 的官方地址：[https://spring.io/projects/spring-amqp](https://spring.io/projects/spring-amqp)

![image12](assets/image12.png)

![image13](assets/image13.png)

SpringAMQP 提供了三个功能：

- 自动声明队列、交换机及其绑定关系
- 基于注解的监听器模式，异步接收消息
- 封装了 RabbitTemplate 工具，用于发送消息

## 1、Basic Queue 简单队列模型

在父工程 mq-demo 中引入 spring-amqp 的依赖

```xml
<!--AMQP依赖，包含RabbitMQ-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 1.1、消息发送

首先配置 MQ 地址，在 publisher 服务的 application.yml 中添加配置：

```yaml
spring:
  rabbitmq:
    host: 192.168.150.101 # 主机名，即rabbitMQ的IP地址
    port: 5672 # 端口
    virtual-host: / # 虚拟主机
    username: itcast # 用户名
    password: 123321 # 密码
```

然后在 publisher 服务中编写测试类 SpringAmqpTest，并利用 RabbitTemplate 实现消息发送：

```java
package cn.itcast.mq.spring;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringAmqpTest {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    public void testSimpleQueue() {
        // 队列名称
        String queueName = "simple.queue";
        // 消息
        String message = "hello, spring amqp!";
        // 发送消息
        rabbitTemplate.convertAndSend(queueName, message);
    }
}
```

### 1.2、消息接收

首先配置 MQ 地址，在 consumer 服务的 application.yml 中添加配置：

```yaml
spring:
  rabbitmq:
    host: 192.168.150.101 # 主机名
    port: 5672 # 端口
    virtual-host: / # 虚拟主机
    username: itcast # 用户名
    password: 123321 # 密码
```

然后在 consumer 服务的 `cn.itcast.mq.listener` 包中新建一个类 SpringRabbitListener，添加 @Component 注解，编写消费逻辑，绑定 simple.queue 这个队列。代码如下：

```java
package cn.itcast.mq.listener;

import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class SpringRabbitListener {
    @RabbitListener(queues = "simple.queue")
    public void listenSimpleQueueMessage(String msg) throws InterruptedException {
        System.out.println("spring消费者接收到simple.queue的消息：【" + msg + "】");
    }
}
```

### 1.3、测试

启动 consumer 服务，然后在 publisher 服务中运行测试代码，发送 MQ 消息

![image14](assets/image14.png)

> 注意：消息一旦消费就会从队列删除，RabbitMQ 没有消息回溯功能

## 2、WorkQueue

Work queues，也被称为 Task queues，任务模型。简单来说就是**让多个消费者绑定到一个队列，共同消费队列中的消息**。

![image15](assets/image15.png)

当消息处理比较耗时的时候，可能生产消息的速度会远远大于消息的消费速度。长此以往，消息就会堆积越来越多，无法及时处理。

此时就可以使用 work 模型，多个消费者共同处理消息处理，速度就能大大提高了。

模拟 WorkQueue 实现一个队列绑定多个消费者，基本思路如下：

1. 在 publisher 服务中定义测试方法，每秒产生 50 条消息，发送到 simple.queue

2. 在 consumer 服务中定义两个消息监听者，都监听 simple.queue 队列

3. 消费者 1 每秒处理 50 条消息，消费者 2 每秒处理 10 条消息

### 2.1、消息发送

这次我们循环发送，模拟大量消息堆积现象。

在 publisher 服务中的 SpringAmqpTest 类中添加一个测试方法，循环发送 50 条消息到 simple.queue 队列：

```java
/**
 * workQueue
 * 向队列中不停发送消息，模拟消息堆积。
 */
@Test
public void testWorkQueue() throws InterruptedException {
    // 队列名称
    String queueName = "simple.queue";
    // 消息
    String message = "hello, message__";
    for (int i = 1; i <= 50; i++) {
        // 发送消息
        rabbitTemplate.convertAndSend(queueName, message + i);
        Thread.sleep(20);
    }
}
```

### 2.2、消息接收

要模拟多个消费者绑定同一个队列，我们在 consumer 服务的 SpringRabbitListener 中添加 2 个新的方法：

```java
@RabbitListener(queues = "simple.queue")
public void listenWorkQueue1(String msg) throws InterruptedException {
    System.out.println("消费者 1 接收到消息：【" + msg + "】" + LocalTime.now());
    Thread.sleep(20);
}

@RabbitListener(queues = "simple.queue")
public void listenWorkQueue2(String msg) throws InterruptedException {
    System.err.println("消费者 2 接收到消息：【" + msg + "】" + LocalTime.now());
    Thread.sleep(200);
}
```

注意到这个消费者 sleep 了 1000 秒，模拟任务耗时。

### 2.3、测试

启动 ConsumerApplication 后，再执行 publisher 服务中刚刚编写的发送测试方法 testWorkQueue。

可以看到消费者 1 很快完成了自己的 25 条消息。消费者 2 却在缓慢的处理自己的 25 条消息。

也就是说消息是平均分配给每个消费者，并没有考虑到消费者的处理能力。这样显然是有问题的。

<img src="assets/image16.png" alt="image16" style="zoom:50%" />

### 2.4、能者多劳

在 spring 中有一个简单的配置，可以解决这个问题。我们修改 consumer 服务的 application.yml 文件，添加配置：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 1 # 每次只能获取一条消息，处理完成才能获取下一个消息
```

### 2.5、总结

Work 模型的使用：

- 多个消费者绑定到一个队列，同一条消息只会被一个消费者处理
- 通过设置 prefetch 来控制消费者预取的消息数量

## 3、发布/订阅

发布订阅模式与之前案例的区别就是允许将同一消息发送给多个消费者。

发布订阅的模型如图：

![image17](assets/image17.png)

可以看到，在订阅模型中，多了一个 exchange 角色，而且过程略有变化：

- Publisher：生产者，也就是要发送消息的程序，但是不再发送到队列中，而是发给交换机
- Exchange：交换机。一方面，接收生产者发送的消息。另一方面，知道如何处理消息，例如递交给某个特别队列、递交给所有队列、或是将消息丢弃。到底如何操作，取决于 Exchange 的类型。Exchange 有以下 3 种类型：
  - Fanout：广播，将消息交给所有绑定到交换机的队列
  - Direct：路由 / 定向，把消息交给符合指定 routing key 的队列
  - Topic：话题 / 通配符，把消息交给符合 routing pattern（路由模式）的队列
- Consumer：消费者，与以前一样，订阅队列，没有变化
- Queue：消息队列也与以前一样，接收消息、缓存消息。

> **Exchange（交换机）只负责转发消息，不具备存储消息的能力**，因此如果没有任何队列与 Exchange 绑定，或者没有符合路由规则的队列，那么消息会丢失！

## 4、Fanout

Fanout，英文翻译是扇出，在 MQ 中叫广播更合适。

Fanout Exchange 会将接收到的消息广播到每一个跟其绑定的 queue

![image18](assets/image18.png)

在广播模式下，消息发送流程是这样的：

- 1）可以有多个队列
- 2）每个队列都要绑定到 Exchange（交换机）
- 3）生产者发送的消息，只能发送到交换机，交换机来决定要发给哪个队列，生产者无法决定
- 4）交换机把消息发送给绑定过的所有队列
- 5）订阅队列的消费者都能拿到消息



我们的计划是这样的：

- 创建一个交换机 itcast.fanout，类型是 Fanout
- 创建两个队列 fanout.queue1 和 fanout.queue2，绑定到交换机 itcast.fanout

![image19](assets/image19.png)

### 4.1、声明队列和交换机

SpringAMQP 提供了声明交换机、队列、绑定关系的 API，Spring 提供了一个接口 Exchange，来表示所有不同类型的交换机：

![image20](assets/image20.png)

在 consumer 中创建一个类，添加 @Configuration 注解，声明队列和交换机，绑定关系对象 Binding：

```java
package cn.itcast.mq.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FanoutConfig {
    /**
     * 声明交换机
     * @return Fanout类型交换机
     */
    @Bean
    public FanoutExchange fanoutExchange(){
        return new FanoutExchange("itcast.fanout");
    }

    /**
     * 声明第1个队列
     */
    @Bean
    public Queue fanoutQueue1(){
        return new Queue("fanout.queue1");
    }

    /**
     * 绑定队列1和交换机
     */
    @Bean
    public Binding bindingQueue1(Queue fanoutQueue1, FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue1).to(fanoutExchange);
    }

    /**
     * 声明第2个队列
     */
    @Bean
    public Queue fanoutQueue2(){
        return new Queue("fanout.queue2");
    }

    /**
     * 绑定队列2和交换机
     */
    @Bean
    public Binding bindingQueue2(Queue fanoutQueue2, FanoutExchange fanoutExchange){
        return BindingBuilder.bind(fanoutQueue2).to(fanoutExchange);
    }
}
```

### 4.2、消息发送

在 publisher 服务的 SpringAmqpTest 类中添加测试方法：

```java
@Test
public void testFanoutExchange() {
    // 交换机名称
    String exchangeName = "itcast.fanout";
    // 消息
    String message = "hello, everyone!";
    // 发送消息，参数分别是：交互机名称、RoutingKey（暂时为空）、消息
    rabbitTemplate.convertAndSend(exchangeName, "", message);
}
```

### 4.3、消息接收

在 consumer 服务的 SpringRabbitListener 中添加两个方法，作为消费者，分别监听 fanout.queue1 和 fanout.queue2：

```java
@RabbitListener(queues = "fanout.queue1")
public void listenFanoutQueue1(String msg) {
    System.out.println("消费者1接收到Fanout消息：【" + msg + "】");
}

@RabbitListener(queues = "fanout.queue2")
public void listenFanoutQueue2(String msg) {
    System.out.println("消费者2接收到Fanout消息：【" + msg + "】");
}
```

### 4.4、总结

交换机的作用是什么？

- 接收 publisher 发送的消息
- 将消息按照规则路由到与之绑定的队列
- 不能缓存消息，路由失败，消息丢失
- FanoutExchange 的会将消息路由到每个绑定的队列

声明队列、交换机、绑定关系的 Bean 是什么？

- Queue
- FanoutExchange
- Binding

## 5、Direct

在 Fanout 模式中，一条消息，会被所有订阅的队列都消费。但是，在某些场景下，我们希望不同的消息被不同的队列消费。这时就要用到 Direct 类型的 Exchange。

Direct Exchange 会将接收到的消息根据规则路由到指定的 Queue，因此称为路由模式（routes）

![image21](assets/image21.png)

 在 Direct 模型下：

- 队列与交换机的绑定，不能是任意绑定了，而是要指定一个 `BindingKey`
- 消息的发送方在向 Exchange 发送消息时，也必须指定消息的 `RoutingKey`（路由 key ）
- Exchange 不再把消息路由给每一个绑定的队列，而是根据消息的 `Routing Key` 进行判断，只有队列的 `Bindingkey` 与消息的 `Routing key` 完全一致，才会接收到消息



案例需求：利用 SpringAMQP 演示 DirectExchange 的使用

实现思路如下：

1. 利用 @RabbitListener 声明 Exchange、Queue、RoutingKey

2. 在 consumer 服务中，编写两个消费者方法，分别监听 direct.queue1 和 direct.queue2

3. 在 publisher 中编写测试方法，向 itcast. direct 发送消息

![image22](assets/image22.png)

### 5.1、基于注解声明队列和交换机

基于 @Bean 的方式声明队列和交换机比较麻烦，Spring 还提供了基于 @RabbitListener 注解方式来声明。

在 consumer 的 SpringRabbitListener 中添加两个消费者，同时基于注解来声明队列和交换机：

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue1"),
    exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
    key = {"red", "blue"}
))
public void listenDirectQueue1(String msg){
    System.out.println("消费者1接收到direct.queue1的消息：【" + msg + "】");
}

@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "direct.queue2"),
    exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
    key = {"red", "yellow"}
))
public void listenDirectQueue2(String msg){
    System.out.println("消费者2接收到direct.queue2的消息：【" + msg + "】");
}
```

### 5.2、消息发送

在 publisher 服务的 SpringAmqpTest 类中添加测试方法：

```java
@Test
public void testSendDirectExchange() {
    // 交换机名称
    String exchangeName = "itcast.direct";
    // 消息
    String message = "红色警报！日本乱排核废水，导致海洋生物变异，惊现哥斯拉！";
    // 发送消息，参数依次为：交换机名称，RoutingKey，消息
    rabbitTemplate.convertAndSend(exchangeName, "red", message);
}
```

### 5.3、总结

描述下 Direct 交换机与 Fanout 交换机的差异？

- Fanout 交换机将消息路由给每一个与之绑定的队列
- Direct 交换机根据 RoutingKey 判断路由给哪个队列
- 如果多个队列具有相同的 RoutingKey，则与 Fanout 功能类似

基于 @RabbitListener 注解声明队列和交换机有哪些常见注解？

- @Queue
- @Exchange

## 6、Topic

### 6.1、说明

`Topic` 类型的 `Exchange` 与 `Direct` 类似，都是可以根据 `RoutingKey` 把消息路由到不同的队列。只不过 `Topic` 类型 `Exchange` 可以让队列在绑定 `Routing key` 的时候使用通配符！

`Routingkey` 一般都是由一个或多个单词组成，多个单词之间以 “.” 分割，例如： `item.insert`

 通配符规则：

`#`：匹配一个或多个词

`*`：匹配不多不少恰好 1 个词

举例：

`item.#`：能够匹配 `item.spu.insert` 或者 `item.spu`

`item.*`：只能匹配 `item.spu`

图示：

 ![image23](assets/image23.png)

解释：

- Queue1：绑定的是 `china.#` ，因此凡是以 `china.` 开头的 `routing key` 都会被匹配到。包括 china.news 和 china.weather
- Queue2：绑定的是 `#.news` ，因此凡是以 `.news` 结尾的 `routing key` 都会被匹配。包括 china.news 和 japan.news



案例需求：利用 SpringAMQP 演示 TopicExchange 的使用

实现思路如下：

1. 利用 @RabbitListener 声明 Exchange、Queue、RoutingKey

2. 在 consumer 服务中，编写两个消费者方法，分别监听 topic.queue1 和 topic.queue2

3. 在 publisher 中编写测试方法，向 itcast. topic 发送消息

![image24](assets/image24.png)

### 6.2、消息发送

在 publisher 服务的 SpringAmqpTest 类中添加测试方法：

```java
/**
 * topicExchange
 */
@Test
public void testSendTopicExchange() {
    // 交换机名称
    String exchangeName = "itcast.topic";
    // 消息
    String message = "杭州亚运会结束了";
    // 发送消息
    rabbitTemplate.convertAndSend(exchangeName, "china.news", message);
}
```

### 6.3、消息接收

在 consumer 服务的 SpringRabbitListener 中添加两个消费者方法，分别监听 topic.queue1 和 topic.queue2，并利用 @RabbitListener 声明 Exchange、Queue、RoutingKey：

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "topic.queue1"),
    exchange = @Exchange(name = "itcast.topic", type = ExchangeTypes.TOPIC),
    key = "china.#"
))
public void listenTopicQueue1(String msg){
    System.out.println("消费者1接收到topic.queue1的消息：【" + msg + "】");
}

@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "topic.queue2"),
    exchange = @Exchange(name = "itcast.topic", type = ExchangeTypes.TOPIC),
    key = "#.news"
))
public void listenTopicQueue2(String msg){
    System.out.println("消费者2接收到topic.queue2的消息：【" + msg + "】");
}
```

### 6.4、总结

描述下 Direct 交换机与 Topic 交换机的差异？

- Topic 交换机接收的消息 RoutingKey 必须是多个单词，以 `**.**` 分割
- Topic 交换机与队列绑定时的 bindingKey 可以指定通配符
  - `#`：代表 0 个或多个词
  - `*`：代表 1 个词


## 7、消息转换器

之前说过，Spring 会把你发送的消息序列化为字节发送给 MQ，接收消息的时候，还会把字节反序列化为 Java 对象。

![image25](assets/image25.png)

只不过，默认情况下 Spring 采用的序列化方式是 JDK 序列化。众所周知，JDK 序列化存在下列问题：

- 数据体积过大
- 有安全漏洞
- 可读性差

我们来测试一下。

### 7.1、测试默认转换器

说明：在 SpringAMQP 的发送方法中，接收消息的类型是 Object，也就是说我们可以发送任意对象类型的消息，SpringAMQP 会帮我们序列化为字节后发送。

我们在 consumer 中利用 @Bean 声明一个队列：

```java
@Bean
public Queue objectQueue() {
    return new Queue("object.queue");
}
```

在 publisher 中发送消息以测试，发送一个 Map 对象：

```java
@Test
public void testSendObjectQueue() {
    // 准备消息
    Map<String,Object> msg = new HashMap<>();
    msg.put("name","柳岩");
    msg.put("age",21);
    // 发送消息
    rabbitTemplate.convertAndSend("object.queue",msg);
}
```

重启 consumer 服务，发送消息后查看 RabbitMQ 控制台：

![image26](assets/image26.png)

### 7.2、配置 JSON 转换器

Spring 对消息对象的处理是由 org.springframework.amqp.support.converter.MessageConverter 来处理的。而默认实现是 SimpleMessageConverter，基于 JDK 的 ObjectOutputStream 完成序列化。

显然，JDK 序列化方式并不合适。我们希望消息体的体积更小、可读性更高，因此可以使用 JSON 方式来做序列化和反序列化。

如果要修改只需要定义一个 MessageConverter 类型的 Bean 即可。推荐用 JSON 方式序列化，步骤如下：

1）在 publisher 和 consumer 两个服务中都引入依赖，或者直接在父工程中引入依赖：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

2）在 publisher 服务声明 MessageConverter：

配置消息转换器，在启动类中添加一个 Bean 即可：

```java
@Bean
public MessageConverter messageConverter() {
    return new Jackson2JsonMessageConverter();
}
```

3）在 consumer 服务的启动类中定义 MessageConverter：

```java
@Bean
public MessageConverter messageConverter() {
    return new Jackson2JsonMessageConverter();
}
```

4）定义一个消费者，监听 object.queue 队列并消费消息：

```java
@RabbitListener(queues = "object.queue")
public void listenObjectQueue(Map<String,Object> msg) {
    System.out.println("接收到object.queue的消息：" + msg);
}
```

重启 consumer 服务，发送消息后查看控制台：

![image27](assets/image27.png)

> 总结：SpringAMQP 中消息的序列化和反序列化是怎么实现的？
>
> * 利用 MessageConverter 实现的，默认是 JDK 的序列化
> * 注意发送方与接收方必须使用相同的 MessageConverter

