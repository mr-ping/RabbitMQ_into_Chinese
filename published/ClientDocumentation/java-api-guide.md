>原文：[Java Client API Guide](https://www.rabbitmq.com/api-guide.html)  
>翻译：[mr-ping](http://rabbitmq.mr-ping.com)  
>状态：[待校对](https://github.com/mr-ping/RabbitMQ_into_Chinese/blob/master/published/ClientDocumentation/java-api-guide.md)  
>许可：![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)


# Java客户端接口（API）指南

## [概览](https://www.rabbitmq.com/api-guide.html#overview)

本篇覆盖了 [RabbitMQ Java 客户端](https://www.rabbitmq.com/java-client.html) 和它的公共接口(API)。这里假设您使用的是[客户端最新的主版本](http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22amqp-client%22)，并且对[基本操作](https://www.rabbitmq.com/getstarted.html)已经有所了解。

指南包含以下的关键部分：

[toc]

也可以单独使用 [API参考](https://rabbitmq.github.io/rabbitmq-java-client/api/current/)（JavaDoc）。

## [支持的时间线](https://www.rabbitmq.com/api-guide.html#support-timeline)

访问 [RabbitMQ Java 库支持页面](https://www.rabbitmq.com/java-versions.html) 了解所支持的时间线。

## [JDK 和 Android 版本支持](https://www.rabbitmq.com/api-guide.html#jdk-versions)

本库的5.x系列的编译和运行需要[JDK 8](https://www.rabbitmq.com/java-versions.html),。对安卓来说，代表着只支持 [Android 7.0 或以上](https://developer.android.com/guide/platform/j8-jack.html) 版本。

4.x系列支持[JDK 6](https://www.rabbitmq.com/java-versions.html) 以及安卓7.0之前的版本。

## [许可](https://www.rabbitmq.com/api-guide.html#license)


本库在 [GitHub](https://github.com/rabbitmq/rabbitmq-java-client/)上开源，并遵循以下三个许可

- [Apache Public License 2.0](https://www.apache.org/licenses/LICENSE-2.0.html)
- [Mozilla Public License 2.0](https://www.mozilla.org/MPL/2.0/)
- [GPL 2.0](http://www.gnu.org/licenses/gpl-2.0.html)

这意味着用户可以根据需要遵循所列出的三个许可中的任意一个即可。例如，用户可以选择依照Apache Public License 2.0许可将客户端应用于商业产品中。GPLv2下的代码库可以选择依照GPLv2许可，以此类推。

## [概览](https://www.rabbitmq.com/api-guide.html#overview)

客户端API提供了 [AMQP 0-9-1 协议模型](/AMQP/AMQP_0-9-1_Model_Explained.html) 中的关键内容，并且额外提供了易于使用的抽象。

RabbitMQ Java 客户端使用`com.rabbitmq.client`作为它的顶级包。关键的类和接口有：

- Channel: 代表 AMQP 0-9-1通道，并提供了大多数操作（协议方法）。
- Connection: 代表 AMQP 0-9-1 连接
- ConnectionFactory: 构建`Connection`实例
- Consumer: 代表消息的消费者
- DefaultConsumer: 消费者通用的基类
- BasicProperties: 消息的属性（元信息）
- BasicProperties.Builder: `BasicProperties`的构建器

通过`Channel`（通道）的接口可以对协议进行操作。`Connection`（连接）用于开启通道，注册连接的生命周期内的处理事件，并且关闭不再需要的连接。`ConnectionFactory`用于实例化`Connection`对象，并且可以通过`ConnectionFactory`来进行诸如vhost、username等属性的设置。

## [连接（Connections） 和 通道（Channels）](https://www.rabbitmq.com/api-guide.html#connections-and-channels)

核心的接口类为`Connection`和`Channel`，分别代表AMQP 0-9-1的连接和通道。通常在使用之前将他们引入：

```java
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
```

## [连接到RabbitMQ](https://www.rabbitmq.com/api-guide.html#connecting)

以下代码演示使用给定的参数（如主机名、端口号）连接到RabbitMQ节点：

```java
ConnectionFactory factory = new ConnectionFactory();
// "guest"/"guest" by default, limited to localhost connections
factory.setUsername(userName);
factory.setPassword(password);
factory.setVirtualHost(virtualHost);
factory.setHost(hostName);
factory.setPort(portNumber);

Connection conn = factory.newConnection();
```

对于运行在本地的RabbitMQ节点而言，这些参数都有合适的默认值。

如果在创建连接前没有指定参数值，则会使用默认参数：

| Property     | Default Value                                                |
| ------------ | ------------------------------------------------------------ |
| Username     | "guest"                                                      |
| Password     | "guest"                                                      |
| Virtual host | "/"                                                          |
| Hostname     | "localhost"                                                  |
| port         | 5672 for regular connections, 5671 for [connections that use TLS](https://www.rabbitmq.com/ssl.html) |

也可以使用 [URIs](https://www.rabbitmq.com/uri-spec.html) 代替：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://userName:password@hostName:portNumber/virtualHost");
Connection conn = factory.newConnection();
```

对于已经运行在本地的RabbitMQ服务器来说，所有这些参数都有合适的默认值。

成功和不成功的客户端连接都可以在[服务器节点日志](https://www.rabbitmq.com/logging.html)中找到。

需要注意的是，默认情况下[guest（来宾）用户只能用本地进行连接](https://www.rabbitmq.com/access-control.html)。目的是为了限制已知凭证在生产系统中的使用。

应用的开发者可以[自定义连接名称](https://www.rabbitmq.com/api-guide.html#client-provided-names)。如果配置了，自定义的名字也会在RabbitMQ节点日志和 [管理界面](https://www.rabbitmq.com/management.html)里体现出来。

接下来，`Connection`接口就可以用来开启通道(Channel)了：

```java
Channel channel = conn.createChannel();
```

通道可用于消息的发送和接收，之后的部分里会进行介绍。

## [关闭RabbitMQ连接](https://www.rabbitmq.com/api-guide.html#disconnecting)

通过简单的对通道和连接进行关闭即可关闭掉RabbitMQ的连接：

```java
channel.close();
conn.close();
```

需要注意的是，虽然将通道关闭掉是最佳实践，但并不是必须的操作。因为无论何种情况，通道都会在底层的连接关闭时自动关闭掉。

客户端关闭操作的事件可以在 [服务器节点日志](https://www.rabbitmq.com/networking.html#logging) 中找到。

## [连接 和 通道 的寿命](https://www.rabbitmq.com/api-guide.html#connection-and-channel-lifspan)

客户端[connections](https://www.rabbitmq.com/connections.html)是长连接。底层协议的设计和优化都考虑到了长连接的需求。这意味着对诸如消息发送之类的每个操作都建立一个连接的形式是极其不推荐的，那样做会产生大量的网络往返和开销。

[Channels](https://www.rabbitmq.com/channels.html) 虽然也是长期存活的，但是由于有大量的可恢复的协议错误会导致通道关闭，通道的存活期会比连接断一些。虽然每个操作都打开和关闭一个通道不是必须的操作，但是也不是不可行。有的选的情况下，还是优先考虑通道的复用为好。

类似于尝试从一个不存在的队列里消费消息这种 [通道级别的异常](https://www.rabbitmq.com/channels.html#error-handling) 会导致通道关闭。已经关闭的通道不可以再被使用，也不会再接收到如*消息投递*之类的服务器事件。RabbitMQ会记录下通道级别的异常，并且会为通道初始化一个关闭顺序（下边会进行介绍）。

## [由客户端提供的链接名称](https://www.rabbitmq.com/api-guide.html#client-provided-names)

RabbitMQ 节点可以持有有限的的客户端信息：

- 客户端的TCP节点（来源IP地址和端口）
- 使用的凭证

凭借这些信息就可以定位出现问题的应用或者实例，特别是在可以共享凭证且客户端通过负载均衡器进行连接但无法启用代理协议的情况下。

包括RabbitMQ Java客户端在内的AMQP 0-9-1客户端链接可以提供一个自定义标识符，一遍在[服务器日志](https://www.rabbitmq.com/logging.html) 和[管理界面](https://www.rabbitmq.com/management.html)中方便地对客户端进行区分。设置好后，日志内容额管理界面中便会对标识符有所体现。标识符即为**客户端提供的连接名称**。名称可以用于标识应用或应用中特定的组件。虽然名称是可选的，但是强烈建议提供一个，这将大大简化某些操作任务。

RabbitMQ Java 客户端的  [`ConnectionFactory#newConnection` 方法 覆写了](https://rabbitmq.github.io/rabbitmq-java-client/api/current/com/rabbitmq/client/ConnectionFactory.html#newConnection(java.util.concurrent.ExecutorService,com.rabbitmq.client.Address[],java.lang.String)) 接收客户端提供的连接名称。这是一个修改过的连接样例，用于提供连接名称：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://userName:password@hostName:portNumber/virtualHost");
// provides a custom connection name
Connection conn = factory.newConnection("app:audit component:event-consumer");
```

## [交换机（Exchanges）和  队列（Queues）的使用](https://www.rabbitmq.com/api-guide.html#exchanges-and-queues)

客户端使用用利用交换机和[队列](https://www.rabbitmq.com/queues.html)这些高级的[协议构建块工作](https://www.rabbitmq.com/tutorials/amqp-concepts.html)。在使用事前必须对他们进行声明。简单来讲，对任何一种对象类型进行声明的目的是为了确保它们已经存在，并在需要的时候对其进行创建。

接着上边的例子，以下代码声明了一个交换机以及一个[服务端命名的队列](https://www.rabbitmq.com/queues.html#server-named-queues)，然后将它们绑定到一起。

```java
channel.exchangeDeclare(exchangeName, "direct", true);
String queueName = channel.queueDeclare().getQueue();
channel.queueBind(queueName, exchangeName, routingKey);
```

这将会主动声明以下对象，这两个对象都可以使用附加参数进行自定义。但在这里，没有给他们俩定义特殊的参数。

- 持久化、非自动删除的“直连”形交换机
- 具有系统生成的名称的，非持久化、独占、自动删除的队列

上边调用的函数使用给定的路由键将队列和交换机绑定起来。

注意，当只有一个客户端打算工作于次队列时，这是一个典型的队列声明方式。队列不需要既定的名称，没有其他客户端使用此队列（独占），队列会被自动清理掉（自动删除）。如果有多个客户端消费打算消费一个既定名称的队列，一下代码更为合适：

```java
channel.exchangeDeclare(exchangeName, "direct", true);
channel.queueDeclare(queueName, true, false, false, null);
channel.queueBind(queueName, exchangeName, routingKey);
```

这将会主动进行以下声明：

- 持久化、非自动删除的“直连”交换机
- 拥有既定名称的，持久化、非独占、非自动删除的队列

许多`Channel`接口方法都是被重载的。这里用到的关于 `exchangeDeclare`, `queueDeclare` 和 `queueBind`的短结构的重载方法使用了合适的默认值，更易于使用。当然也有更多参数的长结构的重载方法，使用那些方法可以将一些必要的默认参数进行重写，进行更全面的控制。

这种“短结构、长结构”的模式在客户端接口的使用中涉及。

### [被动声明](https://www.rabbitmq.com/api-guide.html#passive-declaration)

队列和交换机可以被动地进行声明。被动声明会简单地检查提供的名称所对应的实体是否存在。如果不存在则不会做任何操作。对成功检测到的队列来说，被动声明会返回跟非被动声明同样的信息，即队列中处于就绪状态的消费者和消息数量。

如果对应的实体不存在，操作会抛出一个通道级别的异常。然后通道就不可以继续使用了，需要打开一个新的通道。通常在进行被动声明的时候使用临时的一次性通道。

`Channel#queueDeclarePassive` 和 `Channel#exchangeDeclarePassive` 方法被用来进行被动声明。下边演示`Channel#queueDeclarePassive` 的使用：

```java
Queue.DeclareOk response = channel.queueDeclarePassive("queue-name");
// returns the number of messages in Ready state in the queue
response.getMessageCount();
// returns the number of consumers the queue has
response.getConsumerCount();
```

`Channel#exchangeDeclarePassive` 方法的返回值没包含什么有用的信息。只要方法正确返回，并且没有通道异常发生，就意味着交换机已经存在了。

### [可选响应的操作](https://www.rabbitmq.com/api-guide.html#nowait-methods)

一些常见的操作还带有“非等待”版本，这种版本的操作不会等待服务器的响应。例如，以下方法会声明一个队列并且通知服务器不要发送任何响应

```java
channel.queueDeclareNoWait(queueName, true, false, false, null);
```

“非等待”版本的操作会更具效率，但是安全保障较低，例如，它们更依赖心跳机制去检测失败的操作。如果不确定，就从标准版本的操作用起。“非等待”版本只是在高级拓扑结构（队列、绑定）的情况下需要。

### [删除实体和清除消息](https://www.rabbitmq.com/api-guide.html#deleting-entities)

可以显示地将队列和交换机删除：

```java
channel.queueDelete("queue-name")
```

也可以做到当队列为空时对其进行删除：

```java
channel.queueDelete("queue-name", false, true)
```

或者当它不再被使用的时候（没有任何消费者对其进行消费）：

```java
channel.queueDelete("queue-name", true, false)
```

队列可以被清除（删除里边的所有消息）：

```java
channel.queuePurge("queue-name")
```

## [发布消息](https://www.rabbitmq.com/api-guide.html#publishing)

使用 `Channel.basicPublish` 将消息发布到交换机中：

```java
byte[] messageBodyBytes = "Hello, world!".getBytes();
channel.basicPublish(exchangeName, routingKey, null, messageBodyBytes);
```

想要实现更完善的控制，可以使用重载的变体来指定`mandatory`标识，或者发送预设好消息属性的消息（详见[发布指南](https://www.rabbitmq.com/publishers.html) ）

```java
channel.basicPublish(exchangeName, routingKey, mandatory,
                     MessageProperties.PERSISTENT_TEXT_PLAIN,
                     messageBodyBytes);
```

以下示例发送消息的时候会指定投递模式为2（持久化），优先级为1并且消息体类型（content-type）为"text/plain"。使用`Builder`类去创建一个需要指定多个属性的消息属性对象，例如：

```java
channel.basicPublish(exchangeName, routingKey,
             new AMQP.BasicProperties.Builder()
               .contentType("text/plain")
               .deliveryMode(2)
               .priority(1)
               .userId("bob")
               .build(),
               messageBodyBytes);
```

以下是发布带有自定义headers消息的示例：

```java
Map<String, Object> headers = new HashMap<String, Object>();
headers.put("latitude",  51.5252949);
headers.put("longitude", -0.0905493);

channel.basicPublish(exchangeName, routingKey,
             new AMQP.BasicProperties.Builder()
               .headers(headers)
               .build(),
               messageBodyBytes);
```

以下例子会发布一条具有过期时间属性的消息：

```java
channel.basicPublish(exchangeName, routingKey,
             new AMQP.BasicProperties.Builder()
               .expiration("60000")
               .build(),
               messageBodyBytes);
```

这里我们并没有展示所有的可能性。

注意`BasicProperties`是AMQP自动生成的持有类的内置类。

如果发生 [资源驱动的报警](https://www.rabbitmq.com/alarms.html)，那`Channel#basicPublish `的调用最终会被阻塞掉。

## [通道和并发的注意事项（线程安全）](https://www.rabbitmq.com/api-guide.html#concurrency)

依经验而言，应该尽量避免在线程间共享通道对象。应用应该尽可能每个线程都使用单独的通道，而不是将通道共享给多个线程。

虽然可以安全地并发调用通道上的某些操作，但有些操作则不能并发调用，如果那样做会导致错误的帧交错在网络上，或造成重复确认等问题。

在共享的通道上并发执行发布会导致错误的帧交错在网络上，触发连接级别的协议异常并导致连接被代理直接关闭。因此，需要在应用程序代码中进行显式同步（必须在关键部分调用`Channel＃basicPublish`）。线程之间共享通道也会干扰[发布者确认](https://www.rabbitmq.com/confirms.html)。最好能够完全避免在共享的通道上上进行并发发布，例如通过每个线程使用一个通道的方式实现并发。

也可以通过通道池的方式来避免在共享通道上并发发布消息：一旦一个线程使用完了某个通道，就将通道归还到池中，使得通道可以被其他线程再次使用。通道池可以视为一个特殊的同步解决方案。建议使用现成的池库来实现，而不是自己实现。例如开箱即用的 [Spring AMQP](https://projects.spring.io/spring-amqp/) 。

通道是吃资源的，而且大多数应用情景下同一个JVM进程很少会开放小几百的通道出来。设想我们应用的每个线程都持有一个通道（由于同一个通道不应被用于并发操作），单个JVM里上千个线程已经会是一个相当大的开销，这些开销本来是可以避免的。此外，一小部分快速的发布者可以很轻松地占满网络接口和代理节点，这种情况通常发生在发布行为的工作量小于路由、存储和消息投递的工作量的情况下。

一个需要避免的经典的反模式就是为每个发布的消息开放单独的通道。通道应该是长时间存货的，并且开放一个通道是一个网络往返的过程，所以上边提到的这种模式是相当没效率的。

一个线程用于消费，另一个线程在共享通道上推送是安全的。

服务推送投递（下边介绍）是以并发的方式分发的，并能确保每个通道顺序的固定。分发机制在每个连接中使用一个[java.util.concurrent.ExecutorService](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html)。使用`ConnectionFactory#setSharedExecutor` setter 的`ConnectionFactory`生成的所有连接都可以共享一个自定义的`executor`。

当使用[手动确认](https://www.rabbitmq.com/confirms.html) 的时候，需要考虑到是线程完成的确认动作。这根线程收取投递（例如`Consumer#handleDelivery`将交付处理委派给其他线程）不同，确认操作将`multiple`这个参数设置成`true`是不安全的，可能会导致两次确认，还会触发通道级别异常并且关闭通道。在同一时间单独确认一条独立的消息是没问题的。

## [通过订阅来接收消息 ("推送接口")](https://www.rabbitmq.com/api-guide.html#consuming)

```java
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
```

接收消息最高效的方式是使用`Consumer`接口设置订阅。消息在到达时被自动投递到其中，而不是显示的去请求。

当调用`Consumers`相关的接口方法时，单个订阅始终由其消费者标签引用。消费者标签可以由客户端或者服务器来生成，用于消费者的身份识别。想让RabbitMQ生成一个节点范围内的唯一标签，可以使用不含有消费者标签属性的`Channel#basicConsume` 重载，或者传递一个空字符串做为消费者标签，然后使用`Channel#basicConsume`返回的值。消费者标签同样用于清除消费者之用。

不同的消费者实例必须持有不同的消费者标签。非常不建议在同一个连接上出现重复的消费者标签，这回导致 [自动连接覆盖](https://www.rabbitmq.com/api-guide.html#connection-recovery) 问题，并在监控消费者时混淆监控数据。

实现`Consumer`最简单的方式是子类化`DefaultConsumer`。此子类的实例化对象可以当做`basicConsume`调用时的参数进行传递，用于设置订阅：

```java
boolean autoAck = false;
channel.basicConsume(queueName, autoAck, "myConsumerTag",
     new DefaultConsumer(channel) {
         @Override
         public void handleDelivery(String consumerTag,
                                    Envelope envelope,
                                    AMQP.BasicProperties properties,
                                    byte[] body)
             throws IOException
         {
             String routingKey = envelope.getRoutingKey();
             String contentType = properties.getContentType();
             long deliveryTag = envelope.getDeliveryTag();
             // (process the message components here ...)
             channel.basicAck(deliveryTag, false);
         }
     });
```

这里由于我们设置了`autoAck = false`，需要手动对投递到`Consumer`的消息进行确认。最简便的方式就是如上边介绍的一样在`handleDelivery`中进行。

更复杂的消费者需要去覆写其他方法。特别说明的是，当通道和连接关闭时，`handleShutdownSignal`会被调用，`handleConsumeOk`会在调用其他`Consumer`回调之前被传递给消费者标签。

`Consumers`同样可以通过实现`handleCancelOk`和`handleCancel`方法来分别被告知是通过显式还是隐式方式进行取消。

你可以通过`Channel.basicCancel`显式地取消一个指定的`Consumer`。

```java
channel.basicCancel(consumerTag);
```

传递消费者标签。

就像发布者一样，这里同样也需要考虑到消费者的并发安全性。

消费者的回调的调度是在一个独立的线程池里完成的，这个线程池跟通道实例化的那个池是分开的。这表示`Consumers`可以安全的调用类似于`Channel#queueDeclare`和`Channel#basicCancel`这种链接和通道的阻塞方法。

每个通道都有自己的调度线程。对于大多数常见的每个`Channel`一个`Consumer`的场景下，这意味着消费者之间不会相互影响。需要注意，如果一个通道里有多个消费者，长时间运行的消费者会阻挡通道中其他消费者回调方法的调度。

请参考并发注意事项（线程安全）部分，来获取有关于并发和并发安全危害的其他主题。

## [检索单条消息 ("拉取接口")](https://www.rabbitmq.com/api-guide.html#getting)

按需检索单条消息也是可以的 ("pull API" 又名 polling)。这种消费方法的效率是极低的，比如它使用的是轮训的方式，即使大多数请求结果尚未生成，应用也会重复地去请求结果。因此这种方法是**强烈不建议使用的**。

使用`Channel.basicGet`来进行消息的“拉取”。返回值是包含有头信息（属性）和消息体的`GetResponse`对象实例。

```java
boolean autoAck = false;
GetResponse response = channel.basicGet(queueName, autoAck);
if (response == null) {
    // No message retrieved.
} else {
    AMQP.BasicProperties props = response.getProps();
    byte[] body = response.getBody();
    long deliveryTag = response.getEnvelope().getDeliveryTag();
    // ...
```

由于此例使用[手动确认](https://www.rabbitmq.com/confirms.html)(the autoAck = false above)，所以你需要调用`Channel.basicAckto`确认已经成功接收到了消息（译者注：一般在成功对接收到的消息处理完毕后进行确认）。

```java
// ...
channel.basicAck(method.deliveryTag, false); // acknowledge receipt of the message
}
```

## [处理无法路由的消息](https://www.rabbitmq.com/api-guide.html#returning)

如果发布的消息设置了`mandatory`标识，但是没有被成功路由，代理会将其返回给发送的客户端（通过`AMQP.Basic.Return`命令）。

客户端可以通过实现`ReturnListener`接口并调用`Channel.addReturnListener`来收到此类退还通知。如果客户端没有为特定的通道配置退还监听，那返回的相应消息会被默默地丢弃掉。

```java
channel.addReturnListener(new ReturnListener() {
    public void handleReturn(int replyCode,
                                  String replyText,
                                  String exchange,
                                  String routingKey,
                                  AMQP.BasicProperties properties,
                                  byte[] body)
    throws IOException {
        ...
    }
});
```

例如，客户端发布了一条带有`mandatory`标识的消息，此消息设置了交换机类型为“直连”，但是交换机并没有绑定到队列上，此时退还监听就会被调用。

## [关闭协议](https://www.rabbitmq.com/api-guide.html#shutdown)

### [客户端关闭进程概览](https://www.rabbitmq.com/api-guide.html#shutdown-overview)

AMQP 0-9-1连接和通道使用相同的通用方法来管理网络故障，内部故障和显式本地关闭。

AMQP 0-9-1连接和通道具有以下生命周期状态：

- 打开：对象可以使用了
- 正在关闭：已经明确通知对象在本地进行关闭，已经向所有支持的底层对象发出了关闭请求，并且正在等待其关闭过程完成
- 已关闭：对象已经接收到所有底层对方发出的关闭完成的通知，然后自己也完成了关闭操作。

这些对象只管完成关闭状态，而不关心造成关闭的原因是什么。像应用请求、客户端内部库错误、远程网络请求或者网络错误一概不管。

连接和通道对象会处理如下所示的跟关闭有关的方法：

- `addShutdownListener(ShutdownListener listener)` 和
- `removeShutdownListener(ShutdownListener listener)`用来管理监听器，当对象转换为关闭状态时触发。需要注意的是，给一个已经关闭的对象添加关闭监听器会立即出发监听行为。
- `getCloseReason()`用来获取对象关闭的原因
- `isOpen()`在测试对象开启状态时很有用
- `close(int closeCode, String closeMessage)`用来显式地通知对象执行关闭操作

监听器的简单应用如下：

```java
import com.rabbitmq.client.ShutdownSignalException;
import com.rabbitmq.client.ShutdownListener;

connection.addShutdownListener(new ShutdownListener() {
    public void shutdownCompleted(ShutdownSignalException cause)
    {
        ...
    }
});
```

### [有关关闭情况的信息](https://www.rabbitmq.com/api-guide.html#shutdown-cause)

可以通过显式调用`getCloseReason()`方法或通过使用带有`cause`参数的`ShutdownListener`类的`service(ShutdownSignalException cause)`方法来获取`ShutdownSignalException`，其中包含有关关闭原因的所有可用信息。

`ShutdownSignalException`类提供了用于分析关闭原因的方法。通过调用`isHardError()`方法，我们可以知道是不是因为连接或者通道错误导致，`getReason()`则会以返回AMQP方法的方式提供关闭的有关信息，包括` AMQP.Channel.Close`或者` AMQP.Connection.Close`（如果是客户端库引起的异常，比如网络通讯失败则会返回null，这种情况可以通过`getCause()`来获取异常）

```java
public void shutdownCompleted(ShutdownSignalException cause)
{
  if (cause.isHardError())
  {
    Connection conn = (Connection)cause.getReference();
    if (!cause.isInitiatedByApplication())
    {
      Method reason = cause.getReason();
      ...
    }
    ...
  } else {
    Channel ch = (Channel)cause.getReference();
    ...
  }
}
```

### [原子性和isOpen()方法的使用](https://www.rabbitmq.com/api-guide.html#shutdown-atomicity)

由于通道和连接的isOpen()的返回值依赖于关闭原因是否存在，所以在生产环境中不建议使用这个方法。以下代码对竞争条件的可能性进行了说明：

```java
public void brokenMethod(Channel channel)
{
    if (channel.isOpen())
    {
        // The following code depends on the channel being in open state.
        // However there is a possibility of the change in the channel state
        // between isOpen() and basicQos(1) call
        ...
        channel.basicQos(1);
    }
}
```

相反，我们应该忽略类似的检查，简单地尝试自己所需要执行的动作即可。如果代码执行过程中通道或者连接关闭了，会抛出`ShutdownSignalException`来表示对象是无效的。除此之外，我们还需要捕获由于代理意外关闭连接造成的`SocketException`和代理发起清理关闭所造成的`ShutdownSignalException`所引发的`IOException`异常。

```java
public void validMethod(Channel channel)
{
    try {
        ...
        channel.basicQos(1);
    } catch (ShutdownSignalException sse) {
        // possibly check if channel was closed
        // by the time we started action and reasons for
        // closing it
        ...
    } catch (IOException ioe) {
        // check why connection was closed
        ...
    }
}
```

## [高级连接选项](https://www.rabbitmq.com/api-guide.html#advanced-connection)

### [消费者操作线程池](https://www.rabbitmq.com/api-guide.html#consumer-thread-pool)

默认情况下，消费者线程（参见下边的 [接收](https://www.rabbitmq.com/api-guide.html#consuming) ）会通过一个新的`ExecutorService`线程池分配。如果需要更大的控制权，可以使用`newConnection()`去应用`ExecutorService`以进行替代。这是一个应用一个比常规分配额更大的线程池的示例：

```java
ExecutorService es = Executors.newFixedThreadPool(20);
Connection conn = factory.newConnection(es);
```

`Executors`和`ExecutorService`类都在` java.util.concurrent package`里。

当连接关闭时，默认提供的`ExecutorService`也会执行`shutdown()`，但是用户提供的`ExecutorService`（如上所示）则*不会*执行`shutdown()`。提供自定义`ExecutorService`的客户端必须确保其最终会被关闭（即调用`shutdown()` 方法），否则线程池会影响JVM的中止。

相同的executor服务可能会被多个连接共享，或者接连不断的重复使用、重复连接，但是无论如何当它关闭后是不可以再用的。

应该在有证据表明处理消费回调存在严重瓶颈时才去考虑使用这个功能。如果没有或者只有少量消费者回调需要执行，那默认分配的线程就足够了。即使偶尔会有消费者活动陡增的情况，最初的负载是很小的，并且线程资源的分配和不能无限扩大。

### [主机列表的使用](https://www.rabbitmq.com/api-guide.html#address-array)

把一个`Address`数组传给`newConnection()`是没问题的。`Address`是一个`com.rabbitmq.client package`中包含 *主机* 和 *端口*组件的简单的便捷类。

例如：

```java
Address[] addrArr = new Address[]{ new Address(hostname1, portnumber1)
                                 , new Address(hostname2, portnumber2)};
Connection conn = factory.newConnection(addrArr);
```

这样会先去尝试连接` hostname1:portnumber1`，失败的话会再尝试`hostname2:portnumber2`。返回的连接对象是第一次成功的数组元素的(没抛出`IOException`的话)。这跟分别设置主机和端口然后依次调用`factory.newConnection()`直到成功的操作一毛一样。

如果同时也提供了`ExecutorService`（在`factory.newConnection(es, addrArr)`中使用），那线程池也是对应的第一次成功连接的那个。

如果想要实现连接的更多控制，参见 [服务发现支持](https://www.rabbitmq.com/api-guide.html#service-discovery-with-address-resolver).

### [使用AddressResolver接口实现服务发现](https://www.rabbitmq.com/api-guide.html#service-discovery-with-address-resolver)

我们可以使用`AddressResolver`接口实现来改变连接时的端点解析算法：

```java
Connection conn = factory.newConnection(addressResolver);
```

The AddressResolver interface is like the following:

`AddressResolver`接口类似于：

```java
public interface AddressResolver {

  List<Address> getAddresses() throws IOException;

}
```

就跟 [主机列表](https://www.rabbitmq.com/api-guide.html#address-array)一样，先尝试返回的第一个`Address`，如果失败了再试第二个，直到成功为止。

如果同时也提供了`ExecutorService`（在`factory.newConnection(es, addrArr)`中使用），那线程池也是对应的第一次成功连接的那个。

`AddressResolver`是实现自定义服务发现逻辑的最佳方式，在动态基础设施的状况下尤其有用。结合 [自动发现]](https://www.rabbitmq.com/api-guide.html#recovery)，客户端可以自动连接到首次启动时尚未出现故障的节点。姻亲和负载均衡是自定义`AddressResolver`能做的另外两个情景。

Java客户端附带了以下实现（详见javadoc）：

- `DnsRecordIpAddressResolver`：根据给定的主机名，返回其IP地址（针对DNS服务器平台的解析）。这对简单的基于DNS的如在均衡很帮助很大。
- `DnsSrvRecordAddressResolver`：根据给定的服务的名字，返回其所在的主机名/端口对。搜索服务基于DNS SRV请求实现。如果需要类似于[HashiCorp Consul](https://www.consul.io/)的服务注册功能的话，这也相当实用。

### [心跳超时](https://www.rabbitmq.com/api-guide.html#heartbeats-timeout)

想了解更多关于心跳的信息和如何在java客户端进行配置，请参阅[Heartbeats guide](https://www.rabbitmq.com/heartbeats.html) 

### [自定义线程工厂](https://www.rabbitmq.com/api-guide.html#thread-factories)

类似 Google App Engine (GAE) 的环境能够 [限制直接将线程实例化](https://developers.google.com/appengine/docs/java/#Java_The_sandbox).想要在这种环境里使用RabbitMQ Java客户端，就需要使用适当的方法配置自定义的`ThreadFactory`来实例化线程，例如GAE 的 `ThreadManager`.

以下是针对Google App Engine的示例：

```java
import com.google.appengine.api.ThreadManager;

ConnectionFactory cf = new ConnectionFactory();
cf.setThreadFactory(ThreadManager.backgroundThreadFactory());
```

### [支持Java的非阻塞IO](https://www.rabbitmq.com/api-guide.html#java-nio)

Java客户端4.0版本带来了对Java非阻塞IO（也叫Java NIO）的支持。NIO的目的不是为了比阻塞IO更快，而是为了更方便的实现简单的资源控制（这里指的是线程）。

在默认的阻塞IO模式下，每个连接使用一个线程去从网络套接字(network socket)读取内容。在NIO模式下，你可以控制读取和写入网络套接字的线程的数量。

如果你的Java进程使用了很多连接（数百个之多）的情况下，可以使用NIO模式。你需要使用比默认阻塞模式下更少的线程。设置合适的线程数量的情况下，你不会损失任何性能，特别是连接不是特别繁忙的情况下。

NIO需要显示的开启允许：

```java
ConnectionFactory connectionFactory = new ConnectionFactory();
connectionFactory.useNio();
```

The NIO mode can be configured through the NioParams class:

NIO模式可以通过`NioParams`类进行配置：

```java
  connectionFactory.setNioParams(new NioParams().setNbIoThreads(4));
```

虽然NIO模式默认值是合理的，但是你也有可能需要根据自身的工作负载来对其进行修改。其中一些设置包括：使用的总的IO线程数量，缓存大小，IO循环所使用的服务执行器（service executor），内存中写队列的参数（将请求发送到网络之前写入队列）。阅读Javadoc来了解更多细节和默认值。

## [自动恢复网络故障](https://www.rabbitmq.com/api-guide.html#recovery)

### [恢复连接](https://www.rabbitmq.com/api-guide.html#connection-recovery)

客户端和RabbitMQ节点之间的网络连接会发生失败。RabbitMQ的Java客户端支持自动恢复连接和拓扑（队列，交换机，绑定和消费者）。

多应用的自动恢复过程依照一下步骤进行：

- 重连
- 恢复连接监听
- 重开通道
- 恢复通道监听
- 恢复通道的`basic.qos`设置，发布确认和事务设置

拓扑的恢复包含以下动作，会应用到每个通道

- 重新声明交换机（预定义的除外）
- 重新声明队列
- 恢复所有绑定
- 恢复所有消费者

在Java客户端4.0.0版本中，自动恢复默认是开启的（拓扑的恢复也一样）。

拓扑恢复依赖于实体的每个连接缓存（队列，交换机，绑定，消费者）。当声明一个队列的时候，此队列也会被添加到缓存中。当它被删除或者列入删除计划时（例如是一个 [自动删除](https://www.rabbitmq.com/queues.html)队列），缓存会被移除。此模型有以下限制。

使用`factory.setAutomaticRecoveryEnabled(boolean)`方法开启或停用自动连接恢复。以下代码片段展示了如何显式地开启自动恢复（例如针对Java客户端4.0.0之前的版本）：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(userName);
factory.setPassword(password);
factory.setVirtualHost(virtualHost);
factory.setHost(hostName);
factory.setPort(portNumber);
factory.setAutomaticRecoveryEnabled(true);
// connection that will recover automatically
Connection conn = factory.newConnection();
```

如果因为异常导致恢复失败（比如RabbitMQ节点尚不可用），会在固定的时间间隔（默认5秒）进行重试。间隔可以进行配置：

```java
ConnectionFactory factory = new ConnectionFactory();
// attempt recovery every 10 seconds
factory.setNetworkRecoveryInterval(10000);
```

When a list of addresses is provided, the list is shuffled and all addresses are tried, one after the next:

当提供了地址列表的情况下，列表会被随机重排并逐一尝试：

```java
ConnectionFactory factory = new ConnectionFactory();

Address[] addresses = {new Address("192.168.1.4"), new Address("192.168.1.5")};
factory.newConnection(addresses);
```

### [连接的自动恢复何时会被触发?](https://www.rabbitmq.com/api-guide.html#recovery-triggers)

如果开启了连接自动恢复，会依照一下时间来进行触发：

- 连接的I/O循环中抛出了I/O异常
- 套接字(socket)读操作超时
- 检测到服务器丢失 [心跳](https://www.rabbitmq.com/heartbeats.html)
- 连接的I/O循环中抛出了其他不可预期的异常

以先发生的为准。

如果客户端到RabbitMQ节点的连接初始化失败，连接自动恢复不会生效。应用的开发者需要负责重试连接，记录下失败的尝试，实现重试的次数限制等。这里是一个非常基本的示例：

```java
ConnectionFactory factory = new ConnectionFactory();
// configure various connection settings

try {
  Connection conn = factory.newConnection();
} catch (java.net.ConnectException e) {
  Thread.sleep(5000);
  // apply retry logic
}
```

当连接被应用使用`Connection.Close`方法关闭的的情况下，连接恢复不会启动。

通道级异常不会触发任何恢复，因为这些异常通常指出的是应用程序中的语义问题（例如从一个不存在的队列进行消费）。

### [恢复监听](https://www.rabbitmq.com/api-guide.html#recovery-listeners)

可以在可恢复的连接上注册一个或多个恢复监听。当连接恢复开启时，`ConnectionFactory#newConnection`和`Connection#createChannel`返回的连接实现了 `com.rabbitmq.client.Recoverable`，并且提供了两个相当具有描述性名字的方法。

- `addRecoveryListener`
- `removeRecoveryListener`

请注意，当前需要将连接和通道强制转换为`Recoverable`才能使用这些方法。

### [对发布的影响](https://www.rabbitmq.com/api-guide.html#publishers)

当连接失效时，通过`Channel.basicPublish`发布的消息会丢失掉。客户端不会将其放入队列以用连接恢复后进行投递。想要确认发布的消息是否已经到达RabbitMQ，应用需要使用 [发布确认](https://www.rabbitmq.com/confirms.html) 并且解决连接失败。

### [拓扑恢复](https://www.rabbitmq.com/api-guide.html#topology-recovery)

拓扑的恢复涉及到交换机、队列、绑定和消费者的恢复。自动恢复启用时拓扑恢复也会随之启用。客户端的当代版本都默认启用了拓扑恢复。

有需要的话，拓扑恢复可以显式的关闭：

```java
ConnectionFactory factory = new ConnectionFactory();

Connection conn = factory.newConnection();
// enable automatic recovery (e.g. Java client prior 4.0.0)
factory.setAutomaticRecoveryEnabled(true);
// disable topology recovery
factory.setTopologyRecoveryEnabled(false);
```

### [故障检测和恢复的限制](https://www.rabbitmq.com/api-guide.html#automatic-recovery-limitations)

连接自动恢复有一些局限性和应用程序开发人员需要注意的有意设计的策略。

拓扑恢复依赖于实体的每个连接缓存（队列，交换机，绑定，消费者）。当声明一个队列的时候，此队列也会被添加到缓存中。当它被删除或者列入删除计划时（例如是一个 [自动删除](https://www.rabbitmq.com/queues.html)队列），缓存会被移除。这样就可以在不同的通道上声明和删除实体，而不会产生意外的结果。这也意味着，使用自动连接恢复的消费者标签（特定通道的标识符）在所有通道上必须是唯一的。

当连接断开或丢失时，需要花费一些时间进行检测。因此，库和应用程序意识到有连接失败之前有一个窗口期。在这端时间内发布的所有消息都将照常进行序列化并写入TCP套接字。只有通过[发布者确认](https://www.rabbitmq.com/confirms.html)才能保证将它们成功交付给了代理：按照设计，AMQP 0-9-1的发布过程完全是异步的。

如果在启用了自动恢复的连接中检测到套接字或I / O操作错误，则恢复将在默认的5秒延迟后开始（这个延迟时间是可配置的）。 该设计假定即使许多网络故障是暂时的并且通常持续时间很短，但也不会立即就恢复。 延迟还可以避免在相同连接上发生服务器端资源清除（例如[独占或自动删除队列](https://www.rabbitmq.com/queues.html)删除）和打开新连接之间的资源竞争。

默认情况下，连接恢复尝试将以相同的时间间隔进行，直到成功打开新连接为止。 通过将实现`RecoveryDelayHandler`的实例化对象提供给`ConnectionFactory＃setRecoveryDelayHandler`，可以实现恢复延迟的动态化。 实现动态计算的延迟间隔应避免使用过低的值（根据经验，小于2秒就算过低了）。

当连接处于恢复状态时，在其通道上尝试进行的任何发布都将被拒绝，但也有例外。 客户端当前不对此类传出消息执行任何内部缓冲。 跟踪此类消息并在恢复成功后重新发布它们是应用程序开发人员的责任。 [发布者确认](https://www.rabbitmq.com/confirms.html)是协议的扩展，发布者可以利用其避免消息丢失。

当发生通道级别异常而导致通道干壁时，连接恢复不会生效。 此类异常通常表示的是应用程序级别的问题。 库无法在这种情况下采取适合的应对措施。

如果通道的关闭是通过显式关闭的或者是由于上述通道级别异常引发的。 即使启动了连接恢复，也不会恢复已关闭的通道。

### [手动确认和自动恢复](https://www.rabbitmq.com/api-guide.html#recovery-and-acknowledgements)

当使用手动确认的情况下，可能会发生连接在消息投递成功但并未进行确认的空挡中失效的情况。在连接恢复后，RabbitMQ会在通道里重置投递标签。

这意味着旧的投递标签的*basic.ack*, *basic.nack*, 和 *basic.reject*  会导致通道异常发生。为了避免这种情况，RabbitMQ的Java客户端会保持对投递标签的追踪和更新，以使它们在恢复过程中单调增长。

之后，`Channel.basicAck`, `Channel.basicNack`, 和 `Channel.basicReject`会将调整后的传递标签转换为RabbitMQ使用的传递标签。

带有过时的投递标签的确认将不会发送。使用手动确认和自动恢复的应用必须能够对重新投递的消息进行处理。

### [通道的生命周期和拓扑恢复](https://www.rabbitmq.com/api-guide.html#recovery-channel-lifecycle)

连接自动恢复对应用程序开发人员来说应尽可能透明，这就是为什么即使好几个连接失效，然后在后台恢复的情况下，`Channel`实例任然会保持相同的原因。 从技术上讲，启用自动恢复时，通道实例充当代理或装饰器：它们将AMQP业务委派给实际的AMQP通道实现，并围绕它实施一些恢复机制。 这就是为什么您不应该在通道完成了一些资源创建（队列，交换，绑定）之后对其进行关闭，这会导致稍后的扑恢复失败。 在应用程序的整个生命周期中都应该保持通道的打开状态。

## [未处理的异常](https://www.rabbitmq.com/api-guide.html#unhandled-exceptions)

跟连接、通道、恢复和消费者生命周期有关的未处理的异常会委托给“异常处理”。“异常处理”是对`ExceptionHandler`接口的一个实现。默认情况下，会使用`DefaultExceptionHandler`实例。它会把异常的细节打印到标准输出中。

也可以用`ConnectionFactory#setExceptionHandler`来覆盖默认异常处理。这会应用到所有通过工厂创建的连接中。

```java
ConnectionFactory factory = new ConnectionFactory();
cf.setExceptionHandler(customHandler);
```

异常处理应该将异常记录到日志当中。

## [指标和监控](https://www.rabbitmq.com/api-guide.html#metrics)

客户端会收集活动的连接的运行时指标（例如发布消息的数量）。指标收集是需要在`ConnectionFactory`级别使用`setMetricsCollector(metricsCollector)`方法进行配置的可选功能。此方法需要的`MetricsCollector`实例会在客户端代码中的多处用到。

4.3版本的客户端开始支持 [Micrometer](http://micrometer.io/) 和 [Dropwizard Metrics](http://metrics.dropwizard.io/) ，开箱即用。

以下是收集的指标：

- 开启的连接数量
- 开启的通道数量
- 发布的消息数量
- 消费的消息数量
- 确认的消息数量
- 拒绝的消息数量

Micrometer 和 Dropwizard Metrics 都提供了与消息指标相关的计数器，也提供了平均速率，最后5分钟速率等。他们也支持用于监控和报告的通用工具（如 JMX, Graphite, Ganglia, Datadog等）。更多详细信息，下边会专门进行说明。

启用指标收集时，开发人员应牢记一些注意事项。

- 要使用Micrometer 或者 Dropwizard Metrics 的话，别忘了添加相关依赖（ (在Maven, Gradle, 或者 JAR 文件里)）到JVM classpath中。
- 指标的收集是可以扩展的。推荐为特定目的实现自定义的`MetricsCollector`。
- 虽然`MetricsCollector`是在`ConnectionFactory`层定义的，但是也可以在不同的实例中共享。
- 指标的收集不支持事务。举例来说，如果一个确认（acknowledgment）通过事务发送，然后事务回滚了，那这个确认就已经被客户端指标（很显然不是通过代理）累计了。需要注意的是，确认（acknowledgment）确实已经发送到代理了，然后又被事务回滚给清除了，所以客户端指标对于确认的处理是没问题的。总之，不要把客户端指标用作关键的业务逻辑，因为不保证它们完全准确无误。它们存在的目的在于简单的解释系统的运行情况并且让操作更具效率。

### [Micrometer的支持](https://www.rabbitmq.com/api-guide.html#metrics-micrometer)

首先指标收集已经被开启了：

[Micrometer](http://micrometer.io/) 按照以下方式:

```java
ConnectionFactory connectionFactory = new ConnectionFactory();
MicrometerMetricsCollector metrics = new MicrometerMetricsCollector();
connectionFactory.setMetricsCollector(metrics);
...
metrics.getPublishedMessages(); // get Micrometer's Counter object
```

Micrometer支持 [多种报告后台](http://micrometer.io/docs)：Netflix Atlas, Prometheus, Datadog, Influx, JMX, 等。

通常情况下会将`MeterRegistry`的实例传给`MicrometerMetricsCollector`，这里是使用JMX的示例：

```java
JmxMeterRegistry registry = new JmxMeterRegistry();
MicrometerMetricsCollector metrics = new MicrometerMetricsCollector(registry);
ConnectionFactory connectionFactory = new ConnectionFactory();
connectionFactory.setMetricsCollector(metrics);
```

### [Dropwizard Metrics的支持](https://www.rabbitmq.com/api-guide.html#metrics-dropwizard-metrics)

如下开启指标收集的 [Dropwizard](http://metrics.dropwizard.io/) 支持：

```java
ConnectionFactory connectionFactory = new ConnectionFactory();
StandardMetricsCollector metrics = new StandardMetricsCollector();
connectionFactory.setMetricsCollector(metrics);
...
metrics.getPublishedMessages(); // get Metrics' Meter object
```

Dropwizard Metrics支持[多种报告后台](http://metrics.dropwizard.io/3.2.3/getting-started.html): console, JMX, HTTP, Graphite, Ganglia, 等.

通常你可以将`MetricsRegistry`实例传给`StandardMetricsCollector`，以下是关于JMX的示例：

```java
MetricRegistry registry = new MetricRegistry();
StandardMetricsCollector metrics = new StandardMetricsCollector(registry);

ConnectionFactory connectionFactory = new ConnectionFactory();
connectionFactory.setMetricsCollector(metrics);

JmxReporter reporter = JmxReporter
  .forRegistry(registry)
  .inDomain("com.rabbitmq.client.jmx")
  .build();
reporter.start();
```

## [Google App Engine上的RabbitMQ Java客户端](https://www.rabbitmq.com/api-guide.html#gae-pitfalls)

Using RabbitMQ Java client on Google App Engine (GAE) requires using a custom thread factory that instantiates thread using GAE's ThreadManager (see above). In addition, it is necessary to set a low heartbeat interval (4-5 seconds) to avoid running into the low InputStream read timeouts on GAE:

要在Google App Engine (GAE)上使用RabbitMQ Java客户端的话，需要使用GAE's ThreadManager (see above)这个自定义的线程工具去实例化线程。另外需要设置一个比较低的心跳间隔（4-5秒）来避免GAE上的InputStream读取超时过低。

```java
ConnectionFactory factory = new ConnectionFactory();
cf.setRequestedHeartbeat(5);
```

## [注意事项和限制](https://www.rabbitmq.com/api-guide.html#cache-pitfalls)

为了实现拓扑的自动恢复，RabbitMQ Java客户端维护了一个用于声明队列、交换机和绑定的缓存。缓存是按连接来对应的。欧谢RabbitMQ的功能会导致客户端无法观察到拓扑功能的变更，例如当队列因TTL被删除的情况。大多数情况下，RabbitMQ Java客户端会尝试让缓存实体无效：

- When a queue is deleted.
- 当队列被删除的时候。
- When an exchange is deleted.
- 当交换机被删除的时候。
- When a binding is deleted.
- 当绑定被删除的时候。
- When a consumer is cancelled on an auto-deleted queue.
- 当消费者因队列的自动删除动作而被清理掉的时候。
- When a queue or exchange is unbound from an auto-deleted exchange.
- 当交换机或队列从自动删的的交换机上解绑的时候。

但是，客户端无法跟踪单个连接以外的拓扑变化。 依赖于自动删除队列或交换机以及队列TTL（请注意：不是消息TTL！）和使用[连接自动恢复](https://www.rabbitmq.com/api-guide.html#connection-recovery)的应用应该显式删除已知的未被使用或已被删除的实体，以清除客户端拓扑缓存。 `Channel＃queueDelete`，`Channel＃exchangeDelete`，`Channel＃queueUnbind`和`Channel＃exchangeUnbind`在RabbitMQ 3.3.x中是幂等的（删除不存在的内容不会导致异常）这个特性有助于实现这一操作。

## [远程过程调用-RPC (请求/回复)模式: 示例](https://www.rabbitmq.com/api-guide.html#rpc)

为了方便编写程序，Java客户端提供了使用一个临时回复队列来实现的`RpcClient`类，这样就通过AMQP 0-9-1 实现了简单的 [RPC-风格通讯](https://www.rabbitmq.com/tutorials/tutorial-six-java.html)。

此类没有在RPC属性和返回值方面新增任何特殊格式。它只是简单的实现了附带路由键发送消息到给定的交换机，并且等待在回复队列里等待回应的机制。

```java
import com.rabbitmq.client.RpcClient;

RpcClient rpc = new RpcClient(channel, exchangeName, routingKey);
```

（此类使用AMQP 0-9-1的实现细节如下：发送请求消息时，其`basic.correlation_id`字段设置为该`RpcClient`实例的唯一值，而`basic.reply_to`设置为回复队列的名称。）

一旦创建了此类的实例，就可以使用一下任一方法来发送RPC请求了：

```java
byte[] primitiveCall(byte[] message);
String stringCall(String message)
Map mapCall(Map message)
Map mapCall(Object[] keyValuePairs)
```

`primitiveCall`方法将原始字节数组作为请求和响应主体进行传输。 方法`stringCall`是`primitiveCall`的一个轻量化封装，将消息正文作为默认字符编码的String实例来处理。

`mapCall`这种变体稍微有点复杂：他们将包含普通Java值的`java.util.Map`编码为AMQP 0-9-1二进制表来表示，并以相同的方式来对收到的响应进行解码。 （注意，此处可以使用哪种值类型有一些限制，详细信息请参见javadoc。）

所有编组/解组的便捷方法都使用`primitiveCall`作为传输机制，仅在它之上提供包装层。

## [TLS 的支持](https://www.rabbitmq.com/api-guide.html#tls)

可以通过 [使用 TLS](https://www.rabbitmq.com/ssl.html) 加密客户端到代理之间的通讯。也可以支持客户端\服务器身份认证（也叫对等认证）。以下是一个使用Java客户端实现的最原始的简单示例：

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
factory.setPort(5671);

// Only suitable for development.
// This code will not perform peer certificate chain verification and prone
// to man-in-the-middle attacks.
// See the main TLS guide to learn about peer verification and how to enable it.
factory.useSslProtocol();
```

注意，以上样例客户端默认不强制任何服务器验证（[对等证书链认证](https://www.rabbitmq.com/ssl.html#peer-verification)），使用了”信任所有证书“的`TrustManager`。这在本地开发的时候很是方便，但是**容易收到中间人攻击**，所以 [不推荐在用在生产环境中](https://www.rabbitmq.com/production-checklist.html)

想要对RabbitMQ的TSL支持进行进一步学习，可以参见[TLS 指南](https://www.rabbitmq.com/ssl.html)。如果只打算配置Java客户端（特别是对等认证和信任管理者部分），可以只阅读一下TLS指南的 [相关部分](https://www.rabbitmq.com/ssl.html#java-client) 即可。

## [OAuth 2 的支持](https://www.rabbitmq.com/api-guide.html#oauth2-support)

客户端可以通过 [UAA](https://github.com/cloudfoundry/uaa)这样的OAuth 2 服务器来进行身份认证。服务器端需要启用 [OAuth 2 插件](https://github.com/rabbitmq/rabbitmq-auth-backend-oauth2)，并配置跟客户端使用同一个OAuth 2 服务器 。

### [获取 OAuth 2 令牌](https://www.rabbitmq.com/api-guide.html#oauth2-getting-token)

Java客户端提供了`OAuth2ClientCredentialsGrantCredentialsProvider`类，用来从[OAuth 2 客户端凭证流](https://tools.ietf.org/html/rfc6749#section-4.4)获取JWT令牌。客户端会在打开连接的时候将令牌放在password字段中进行发送。然后代理会在授权之前验证JWT令牌的签名、有效性和权限，并授予对请求的虚拟主机的访问权限。

优先使用`OAuth2ClientCredentialsGrantCredentialsProviderBuilder`来创建`OAuth2ClientCredentialsGrantCredentialsProvider`实例，然后用它来配置`ConnectionFactory`。以下片段展示了如何为[配置 OAuth 2 插件的示例](https://github.com/rabbitmq/rabbitmq-auth-backend-oauth2#examples)设置和创建OAuth 2 credentials provider实例：

```java
import com.rabbitmq.client.impl.OAuth2ClientCredentialsGrantCredentialsProvider.
        OAuth2ClientCredentialsGrantCredentialsProviderBuilder;
...
CredentialsProvider credentialsProvider =
  new OAuth2ClientCredentialsGrantCredentialsProviderBuilder()
    .tokenEndpointUri("http://localhost:8080/uaa/oauth/token/")
    .clientId("rabbit_client").clientSecret("rabbit_secret")
    .grantType("password")
    .parameter("username", "rabbit_super")
    .parameter("password", "rabbit_super")
    .build();

connectionFactory.setCredentialsProvider(credentialsProvider);
```

在生产环境中，确认令牌断电URI使用的是HTTPS，并且根据需要为HTTPS请求配置了`SSLContext`（用来验证和信任OAuth 2 服务器的身份）。以下代码片段使用`OAuth2ClientCredentialsGrantCredentialsProviderBuilder`的`tls().sslContext() `方法实现了上边所提及事项：

```java
SSLContext sslContext = ... // create and initialise SSLContext

CredentialsProvider credentialsProvider =
  new OAuth2ClientCredentialsGrantCredentialsProviderBuilder()
    .tokenEndpointUri("http://localhost:8080/uaa/oauth/token/")
    .clientId("rabbit_client").clientSecret("rabbit_secret")
    .grantType("password")
    .parameter("username", "rabbit_super")
    .parameter("password", "rabbit_super")
    .tls()                    // configure TLS
      .sslContext(sslContext) // set SSLContext
      .builder()              // back to main configuration
    .build();
```

更多选项请参照[Javadoc](https://rabbitmq.github.io/rabbitmq-java-client/api/current/com/rabbitmq/client/impl/OAuth2ClientCredentialsGrantCredentialsProvider.html)。

### [刷新令牌](https://www.rabbitmq.com/api-guide.html#oauth2-refreshing-token)

令牌是会或过期的，代理会拒绝带有过期令牌的连接所请求的操作。可以使用`CredentialsProvider#refresh()`在令牌过期前使用新令牌发送请求，以防止此情况的发生。应用自己来实现是比较麻烦的，所以Java客户端提供了`DefaultCredentialsRefreshService`来给予一定帮助。这个工具用来追踪使用的令牌，在过期前进行刷新，并将新令牌发送给所负责的连接。

以下代码片段展示了如何创建`DefaultCredentialsRefreshService`实例，并且将其配置到`ConnectionFactory`上。

```java
import com.rabbitmq.client.impl.DefaultCredentialsRefreshService.
        DefaultCredentialsRefreshServiceBuilder;
...
CredentialsRefreshService refreshService =
  new DefaultCredentialsRefreshServiceBuilder().build();
cf.setCredentialsRefreshService(refreshService);
```

`DefaultCredentialsRefreshService`会在令牌有效期超过80%后进行刷新，例如，如果令牌在60分钟后过期，`DefaultCredentialsRefreshService`会在48分钟的时候进行刷新。这是默认的行为，更多的细节可以通过 [Javadoc](https://rabbitmq.github.io/rabbitmq-java-client/api/current/com/rabbitmq/client/impl/DefaultCredentialsRefreshService.html) 了解。

## 获取帮助和提供建议

如果你对本指南的内容或RabbitMQ的其他主题有任何疑问。可以通多 [RabbitMQ 邮件列表](https://groups.google.com/forum/#!forum/rabbitmq-users)进行提问。
