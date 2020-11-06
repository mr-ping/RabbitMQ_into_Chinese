>原文：[Java Client API Guide](https://www.rabbitmq.com/api-guide.html)  
>翻译：[mr-ping](http://rabbitmq.mr-ping.com)  
>状态：[待校对](http://rabbitmq.mr-ping.com)  

![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)


# Java客户端接口（API）指南

## [概览](https://www.rabbitmq.com/api-guide.html#overview)

本篇覆盖了 [RabbitMQ Java 客户端](https://www.rabbitmq.com/java-client.html) 和它的公共接口(API)。这里假设您使用的是[客户端的最新的主版本](http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22amqp-client%22)，并且已经对[基本操作](https://www.rabbitmq.com/getstarted.html)有所熟悉。

指南包含的关键部分有：

[toc]

[API参考](https://rabbitmq.github.io/rabbitmq-java-client/api/current/)（JavaDoc）单独可用。

## [支持的时间线](https://www.rabbitmq.com/api-guide.html#support-timeline)

访问 [RabbitMQ Java 库支持页面](https://www.rabbitmq.com/java-versions.html) 了解所支持的时间线。

## [JDK 和 Android 版本支持](https://www.rabbitmq.com/api-guide.html#jdk-versions)

本库的5.x系列的编译和运行需要[JDK 8](https://www.rabbitmq.com/java-versions.html),。对安卓来说，意味着只支持 [Android 7.0 或以上](https://developer.android.com/guide/platform/j8-jack.html) 版本。

4.x系列支持[JDK 6](https://www.rabbitmq.com/java-versions.html) 以及安卓7.0之前的版本。

## [许可](https://www.rabbitmq.com/api-guide.html#license)

The library is open source, developed [on GitHub](https://github.com/rabbitmq/rabbitmq-java-client/), and is triple-licensed under

本库在 [GitHub](https://github.com/rabbitmq/rabbitmq-java-client/)上开源，遵循以下三个许可

- [Apache Public License 2.0](https://www.apache.org/licenses/LICENSE-2.0.html)
- [Mozilla Public License 2.0](https://www.mozilla.org/MPL/2.0/)
- [GPL 2.0](http://www.gnu.org/licenses/gpl-2.0.html)

这意味着用户可以根据需要遵循列出的三个许可中的任意一个即可。例如，用户可以选择依照Apache Public License 2.0许可将客户端应用于商业产品中。GPLv2下的代码库可以选择依照GPLv2许可，以此类推。

## [概览](https://www.rabbitmq.com/api-guide.html#overview)

客户端API提供了 [AMQP 0-9-1 protocol model](https://www.rabbitmq.com/tutorials/amqp-concepts.html) 中的关键内容，并且额外提供了易于使用的抽象。

RabbitMQ Java 客户端使用`com.rabbitmq.client`作为它的顶级包。关键的类和接口有：

- Channel: 代表 AMQP 0-9-1通道，并提供了大多数操作（协议方法）。
- Connection: 代表 AMQP 0-9-1 连接
- ConnectionFactory: 构建`Connection`实例
- Consumer: 代表消息的消费者
- DefaultConsumer: 消费者通用的基类
- BasicProperties: 消息的属性（元信息）
- BasicProperties.Builder: `BasicProperties`的构建器

通过`Channel`（通道）的接口可以对协议进行操作。`Connection`（连接）用于开启通道，注册连接的生命周期内的处理事件，并且关闭不再需要的连接。`ConnectionFactory`用于实例化`Connection`对象，并且可以通过`ConnectionFactory`来进行诸如vhost、username等属性的设置。

## [Connections（连接） 和 Channels（通道）](https://www.rabbitmq.com/api-guide.html#connections-and-channels)

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

All of these parameters have sensible defaults for a stock RabbitMQ server running locally.

对于已经运行在本地的RabbitMQ服务器来说，所有这些参数都有合适的默认值。

Successful and unsuccessful client connection events can be [observed in server node logs](https://www.rabbitmq.com/logging.html).

成功和不成功的客户端连接都可以在[服务器节点日志](https://www.rabbitmq.com/logging.html)中找到。

Note that [user guest can only connect from localhost](https://www.rabbitmq.com/access-control.html) by default. This is to limit well-known credential use in production systems.

需要注意的是，默认情况下[guest（来宾）用户只能用本地进行连接](https://www.rabbitmq.com/access-control.html)。目的是为了限制已知凭证在生产系统中的使用。

Application developers can [assign a custom name to a connection](https://www.rabbitmq.com/api-guide.html#client-provided-names). If set, the name will be mentioned in RabbitMQ node logs as well as [management UI](https://www.rabbitmq.com/management.html).

The Connection interface can then be used to open a channel:

接下来，`Connection`接口就可以用来开启通道(Channel)了：

```java
Channel channel = conn.createChannel();
```

The channel can now be used to send and receive messages, as described in subsequent sections.

通道可用于消息的发送和接收，之后的部分里会进行介绍。

## [关闭RabbitMQ连接](https://www.rabbitmq.com/api-guide.html#disconnecting)

通过简单的对通道和连接进行关闭即可关闭掉RabbitMQ的连接：

```java
channel.close();
conn.close();
```

需要注意的是，虽然将通道关闭掉是最佳实践，但并不是必须的操作。因为无论何种情况，通道都会在底层的连接关闭时自动关闭掉。

客户端关闭操作的事件可以在 [服务器节点日志](https://www.rabbitmq.com/networking.html#logging) 中找到。

## [连接(Connection) 和 通道(Channel) 的寿命](https://www.rabbitmq.com/api-guide.html#connection-and-channel-lifspan)

Client [connections](https://www.rabbitmq.com/connections.html) are meant to be long-lived. The underlying protocol is designed and optimized for long running connections. That means that opening a new connection per operation, e.g. a message published, is unnecessary and strongly discouraged as it will introduce a lot of network roundtrips and overhead.

客户端[connections](https://www.rabbitmq.com/connections.html)是长连接。底层协议的设计和优化都考虑到了长连接的需求。这意味着对诸如消息发送之类的每个操作都建立一个连接的形式是极其不推荐的，那样做会产生大量的网络往返和开销。

[Channels](https://www.rabbitmq.com/channels.html) are also meant to be long-lived but since many recoverable protocol errors will result in channel closure, channel lifespan could be shorter than that of its connection. Closing and opening new channels per operation is usually unnecessary but can be appropriate. When in doubt, consider reusing channels first.

[Channels](https://www.rabbitmq.com/channels.html) 虽然也是长期存活的，但是由于有大量的可恢复的协议错误会导致通道关闭，通道的存活期会比连接断一些。虽然每个操作都打开和关闭一个通道不是必须的操作，但是也不是不可行。有的选的情况下，还是优先考虑通道的复用为好。

[Channel-level exceptions](https://www.rabbitmq.com/channels.html#error-handling) such as attempts to consume from a queue that does not exist will result in channel closure. A closed channel can no longer be used and will not receive any more events from the server (such as message deliveries). Channel-level exceptions will be logged by RabbitMQ and will initiate a shutdown sequence for the channel (see below).

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

## [Exchanges（交换机）和  Queues（队列）的使用](https://www.rabbitmq.com/api-guide.html#exchanges-and-queues)

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

This will actively declare:

这会主动进行以下声明：

- 持久化、非自动删除的“直连”交换机
- 拥有既定名称的，持久化、非独占、非自动删除的队列

许多`Channel`接口方法都是被重载的。这里用到的关于 `exchangeDeclare`, `queueDeclare` 和 `queueBind`的短结构的重载方法使用了合适的默认值，更易于使用。当然也有更多参数的长结构的重载方法，使用那些方法可以将一些必要的默认参数进行重写，进行更全面的控制。

这种“短结构、长结构”的模式在客户端接口的使用中涉及。

### [被动声明](https://www.rabbitmq.com/api-guide.html#passive-declaration)

Queues and exchanges can be declared "passively". A passive declare simply checks that the entity with the provided name exists. If it does, the operation is a no-op. For queues successful passive declares will return the same information as non-passive ones, namely the number of consumers and messages in [ready state](https://www.rabbitmq.com/confirms.html) in the queue.

队列和交换机可以被动地进行声明。被动声明会简单地检查提供的名称所对应的实体是否存在。如果不存在则不会做任何操作。对成功检测到的队列来说，被动声明会返回跟非被动声明同样的信息，即队列中处于就绪状态的消费者和消息数量。

If the entity does not exist, the operation fails with a channel level exception. The channel cannot be used after that. A new channel should be opened. It is common to use one-off (temporary) channels for passive declarations.

如果对应的实体不存在，操作会抛出一个通道级别的异常。然后通道就不可以继续使用了，需要打开一个新的通道。通常在进行被动声明的时候使用临时的一次性通道。

Channel#queueDeclarePassive and Channel#exchangeDeclarePassive are the methods used for passive declaration. The following example demonstrates Channel#queueDeclarePassive:

`Channel#queueDeclarePassive` 和 `Channel#exchangeDeclarePassive` 方法被用来进行被动声明。下边演示`Channel#queueDeclarePassive` 的使用：

```java
Queue.DeclareOk response = channel.queueDeclarePassive("queue-name");
// returns the number of messages in Ready state in the queue
response.getMessageCount();
// returns the number of consumers the queue has
response.getConsumerCount();
```

Channel#exchangeDeclarePassive's return value contains no useful information. Therefore if the method returns and no channel exceptions occurs, it means that the exchange does exist.

`Channel#exchangeDeclarePassive` 方法的返回值没包含什么有用的信息。只要方法正确返回，并且没有通道异常发生，就意味着交换机已经存在了。

### [Operations with Optional Responses](https://www.rabbitmq.com/api-guide.html#nowait-methods)

### [可选响应的操作](https://www.rabbitmq.com/api-guide.html#nowait-methods)

Some common operations also have a "no wait" version which won't wait for server response. For example, to declare a queue and instruct the server to not send any response, use

一些常见的操作还带有“非等待”版本，这种版本的操作不会等待服务器的响应。例如，以下方法会声明一个队列并且通知服务器不要发送任何响应

```java
channel.queueDeclareNoWait(queueName, true, false, false, null);
```

The "no wait" versions are more efficient but offer lower safety guarantees, e.g. they are more dependent on the [heartbeat mechanism](https://www.rabbitmq.com/heartbeats.html) for detection of failed operations. When in doubt, start with the standard version. The "no wait" versions are only needed in scenarios with high topology (queue, binding) churn.

“非等待”版本的操作会更具效率，但是安全保障较低，例如，它们更依赖心跳机制去检测失败的操作。如果不确定，就从标准版本的操作用起。“非等待”版本只是在高级拓扑结构（队列、绑定）的情况下需要。

### [Deleting Entities and Purging Messages](https://www.rabbitmq.com/api-guide.html#deleting-entities)

### [删除实体和清除消息](https://www.rabbitmq.com/api-guide.html#deleting-entities)

A queue or exchange can be explicitly deleted:

可以显示地将队列和交换机删除：

```java
channel.queueDelete("queue-name")
```

It is possible to delete a queue only if it is empty:

也可以做到当队列为空时对其进行删除：

```java
channel.queueDelete("queue-name", false, true)
```

or if it is not used (does not have any consumers):

或者当它不再被使用的时候（没有任何消费者对其进行消费）：

```java
channel.queueDelete("queue-name", true, false)
```

A queue can be purged (all of its messages deleted):

队列可以被清除（删除里边的所有消息）：

```java
channel.queuePurge("queue-name")
```

## [Publishing Messages](https://www.rabbitmq.com/api-guide.html#publishing)

## [发布消息](https://www.rabbitmq.com/api-guide.html#publishing)

To publish a message to an exchange, use Channel.basicPublish as follows:

使用 `Channel.basicPublish` 将消息发布到交换机中：

```java
byte[] messageBodyBytes = "Hello, world!".getBytes();
channel.basicPublish(exchangeName, routingKey, null, messageBodyBytes);
```

For fine control, use overloaded variants to specify the mandatory flag, or send messages with pre-set message properties (see the [Publishers guide](https://www.rabbitmq.com/publishers.html) for details):

想要实现更完善的控制，可以使用重载的变体来指定`mandatory`标识，或者发送预设好消息属性的消息（详见[发布指南](https://www.rabbitmq.com/publishers.html) ）

```java
channel.basicPublish(exchangeName, routingKey, mandatory,
                     MessageProperties.PERSISTENT_TEXT_PLAIN,
                     messageBodyBytes);
```

This sends a message with delivery mode 2 (persistent), priority 1 and content-type "text/plain". Use the Builder class to build a message properties object with as many properties as needed, for example:

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

This example publishes a message with custom headers:

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

This example publishes a message with expiration:

以下例子会发布一条具有过期时间属性的消息：

```java
channel.basicPublish(exchangeName, routingKey,
             new AMQP.BasicProperties.Builder()
               .expiration("60000")
               .build(),
               messageBodyBytes);
```

We have not illustrated all the possibilities here.

这里我们并没有展示所有的可能性。

Note that BasicProperties is an inner class of the autogenerated holder class AMQP.

注意`BasicProperties`是AMQP自动生成的持有类的内置类。

Invocations of Channel#basicPublish will eventually block if a [resource-driven alarm](https://www.rabbitmq.com/alarms.html) is in effect.

如果发生 [资源驱动的报警](https://www.rabbitmq.com/alarms.html)，那`Channel#basicPublish `的调用最终会被阻塞掉。

## [Channels and Concurrency Considerations (Thread Safety)](https://www.rabbitmq.com/api-guide.html#concurrency)

## [通道和并发的注意事项（线程安全）](https://www.rabbitmq.com/api-guide.html#concurrency)

As a rule of thumb, sharing Channel instances between threads is something to be avoided. Applications should prefer using a Channel per thread instead of sharing the same Channel across multiple threads.

依经验而言，应该尽量避免在线程间共享通道对象。应用应该尽可能每个线程都使用单独的通道，而不是将通道共享给多个线程。

While some operations on channels are safe to invoke concurrently, some are not and will result in incorrect frame interleaving on the wire, double acknowledgements and so on.

虽然可以安全地并发调用通道上的某些操作，但有些操作则不能并发调用，如果那样做会导致错误的帧交错在网络上，或造成重复确认等问题。

Concurrent publishing on a shared channel can result in incorrect frame interleaving on the wire, triggering a connection-level protocol exception and immediate connection closure by the broker. It therefore requires explicit synchronization in application code (Channel#basicPublish must be invoked in a critical section). Sharing channels between threads will also interfere with [Publisher Confirms](https://www.rabbitmq.com/confirms.html). Concurrent publishing on a shared channel is best avoided entirely, e.g. by using a channel per thread.

在共享的通道上并发执行发布会导致错误的帧交错在网络上，触发连接级别的协议异常并导致连接被代理直接关闭。因此，需要在应用程序代码中进行显式同步（必须在关键部分调用`Channel＃basicPublish`）。线程之间共享通道也会干扰[发布者确认](https://www.rabbitmq.com/confirms.html)。最好能够完全避免在共享的通道上上进行并发发布，例如通过每个线程使用一个通道的方式实现并发。

It is possible to use channel pooling to avoid concurrent publishing on a shared channel: once a thread is done working with a channel, it returns it to the pool, making the channel available for another thread. Channel pooling can be thought of as a specific synchronization solution. It is recommended that an existing pooling library is used instead of a homegrown solution. For example, [Spring AMQP](https://projects.spring.io/spring-amqp/) which comes with a ready-to-use channel pooling feature.

也可以通过通道池的方式来避免在共享通道上并发发布消息：一旦一个线程使用完了某个通道，就将通道归还到池中，使得通道可以被其他线程再次使用。通道池可以视为一个特殊的同步解决方案。建议使用现成的池库来实现，而不是自己实现。例如开箱即用的 [Spring AMQP](https://projects.spring.io/spring-amqp/) 。

Channels consume resources and in most cases applications very rarely need more than a few hundreds open channels in the same JVM process. If we assume that the application has a thread for each channel (as channel shouldn't be used concurrently), thousands of threads for a single JVM is already a fair amount of overhead that likely can be avoided. Moreover a few fast publishers can easily saturate a network interface and a broker node: publishing involves less work than routing, storing and delivering messages.

通道是吃资源的，而且大多数应用情景下同一个JVM进程很少会开放小几百的通道出来。设想我们应用的每个线程都持有一个通道（由于同一个通道不应被用于并发操作），单个JVM里上千个线程已经会是一个相当大的开销，这些开销本来是可以避免的。此外，一小部分快速的发布者可以很轻松地占满网络接口和代理节点，这种情况通常发生在发布行为的工作量小于路由、存储和消息投递的工作量的情况下。

A classic anti-pattern to be avoided is opening a channel for each published message. Channels are supposed to be reasonably long-lived and opening a new one is a network round-trip which makes this pattern extremely inefficient.

一个需要避免的经典的反模式就是为每个发布的消息开放单独的通道。通道应该是长时间存货的，并且开放一个通道是一个网络往返的过程，所以上边提到的这种模式是相当没效率的。

Consuming in one thread and publishing in another thread on a shared channel can be safe.

一个线程用于消费，另一个线程在共享通道上推送是安全的。

Server-pushed deliveries (see the section below) are dispatched concurrently with a guarantee that per-channel ordering is preserved. The dispatch mechanism uses a [java.util.concurrent.ExecutorService](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html), one per connection. It is possible to provide a custom executor that will be shared by all connections produced by a single ConnectionFactory using the ConnectionFactory#setSharedExecutor setter.

服务推送投递（下边介绍）是以并发的方式分发的，并能确保每个通道顺序的固定。分发机制在每个连接中使用一个[java.util.concurrent.ExecutorService](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html)。使用`ConnectionFactory#setSharedExecutor` setter 的`ConnectionFactory`生成的所有连接都可以共享一个自定义的`executor`。

When [manual acknowledgements](https://www.rabbitmq.com/confirms.html) are used, it is important to consider what thread does the acknowledgement. If it's different from the thread that received the delivery (e.g. Consumer#handleDelivery delegated delivery handling to a different thread), acknowledging with the multiple parameter set to true is unsafe and will result in double-acknowledgements, and therefore a channel-level protocol exception that closes the channel. Acknowledging a single message at a time can be safe.

当使用[手动确认](https://www.rabbitmq.com/confirms.html) 的时候，需要考虑到是线程完成的确认动作。这根线程收取投递（例如`Consumer#handleDelivery`将交付处理委派给其他线程）不同，确认操作将`multiple`这个参数设置成`true`是不安全的，可能会导致两次确认，还会触发通道级别异常并且关闭通道。在同一时间单独确认一条独立的消息是没问题的。

## [Receiving Messages by Subscription ("Push API")](https://www.rabbitmq.com/api-guide.html#consuming)

## [通过订阅来接收消息 ("推送接口")](https://www.rabbitmq.com/api-guide.html#consuming)

```java
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
```

The most efficient way to receive messages is to set up a subscription using the Consumer interface. The messages will then be delivered automatically as they arrive, rather than having to be explicitly requested.

接收消息最高效的方式是使用`Consumer`接口设置订阅。消息在到达时被自动投递到其中，而不是显示的去请求。

When calling the API methods relating to Consumers, individual subscriptions are always referred to by their consumer tags. A consumer tag is a consumer identifier which can be either client- or server-generated. To let RabbitMQ generate a node-wide unique tag, use a Channel#basicConsume override that doesn't take a consumer tag argument or pass an empty string for consumer tag and use the value returned by Channel#basicConsume. Consumer tags are used to cancel consumers.

当调用`Consumers`相关的接口方法时，单个订阅始终由其消费者标签引用。消费者标签可以由客户端或者服务器来生成，用于消费者的身份识别。想让RabbitMQ生成一个节点范围内的唯一标签，可以使用不含有消费者标签属性的`Channel#basicConsume` 重载，或者传递一个空字符串做为消费者标签，然后使用`Channel#basicConsume`返回的值。消费者标签同样用于清除消费者之用。

Distinct Consumer instances must have distinct consumer tags. Duplicate consumer tags on a connection is strongly discouraged and can lead to issues with [automatic connection recovery](https://www.rabbitmq.com/api-guide.html#connection-recovery) and confusing monitoring data when consumers are monitored.

不同的消费者实例必须持有不同的消费者标签。非常不建议在同一个连接上出现重复的消费者标签，这回导致 [自动连接覆盖](https://www.rabbitmq.com/api-guide.html#connection-recovery) 问题，并在监控消费者时混淆监控数据。

The easiest way to implement a Consumer is to subclass the convenience class DefaultConsumer. An object of this subclass can be passed on a basicConsume call to set up the subscription:

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

Here, since we specified autoAck = false, it is necessary to acknowledge messages delivered to the Consumer, most conveniently done in the handleDelivery method, as illustrated.

这里由于我们设置了`autoAck = false`，需要手动对投递到`Consumer`的消息进行确认。最简便的方式就是如上边介绍的一样在`handleDelivery`中进行。

More sophisticated Consumers will need to override further methods. In particular, handleShutdownSignal is called when channels and connections close, and handleConsumeOk is passed the consumer tag before any other callbacks to that Consumer are called.

更复杂的消费者需要去覆写其他方法。特别说明的是，当通道和连接关闭时，`handleShutdownSignal`会被调用，`handleConsumeOk`会在调用其他`Consumer`回调之前被传递给消费者标签。

Consumers can also implement the handleCancelOk and handleCancel methods to be notified of explicit and implicit cancellations, respectively.

`Consumers`同样可以通过实现`handleCancelOk`和`handleCancel`方法来分别被告知是通过显式还是隐式方式进行取消。

You can explicitly cancel a particular Consumer with Channel.basicCancel:

你可以通过`Channel.basicCancel`显式地取消一个指定的`Consumer`。

```java
channel.basicCancel(consumerTag);
```

passing the consumer tag.

传递消费者标签。

Just like with publishers, it is important to consider concurrency hazard safety for consumers.

就像发布者一样，这里同样也需要考虑到消费者的并发安全性。

Callbacks to Consumers are dispatched in a thread pool separate from the thread that instantiated its Channel. This means that Consumers can safely call blocking methods on the Connection or Channel, such as Channel#queueDeclare or Channel#basicCancel.

消费者的回调的调度是在一个独立的线程池里完成的，这个线程池跟通道实例化的那个池是分开的。这表示`Consumers`可以安全的调用类似于`Channel#queueDeclare`和`Channel#basicCancel`这种链接和通道的阻塞方法。

Each Channel has its own dispatch thread. For the most common use case of one Consumer per Channel, this means Consumers do not hold up other Consumers. If you have multiple Consumers per Channel be aware that a long-running Consumer may hold up dispatch of callbacks to other Consumers on that Channel.

每个通道都有自己的调度线程。对于大多数常见的每个`Channel`一个`Consumer`的场景下，这意味着消费者之间不会相互影响。需要注意，如果一个通道里有多个消费者，长时间运行的消费者会阻挡通道中其他消费者回调方法的调度。

Please refer to the Concurrency Considerations (Thread Safety) section for other topics related to concurrency and concurrency hazard safety.

请参考并发注意事项（线程安全）部分，来获取有关于并发和并发安全危害的其他主题。

## [Retrieving Individual Messages ("Pull API")](https://www.rabbitmq.com/api-guide.html#getting)

## [检索单条消息 ("拉取接口")](https://www.rabbitmq.com/api-guide.html#getting)

It is also possible to retrieve individual messages on demand ("pull API" a.k.a. polling). This approach to consumption is highly inefficient as it is effectively polling and applications repeatedly have to ask for results even if the vast majority of the requests yield no results. Therefore using this approach **is highly discouraged**.

按需检索单条消息也是可以的 ("pull API" 又名 polling)。这种消费方法的效率是极低的，比如它使用的是轮训的方式，即使大多数请求结果尚未生成，应用也会重复地去请求结果。因此这种方法是**强烈不建议使用的**。

To "pull" a message, use the Channel.basicGet method. The returned value is an instance of GetResponse, from which the header information (properties) and message body can be extracted:

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

and since this example uses [manual acknowledgements](https://www.rabbitmq.com/confirms.html) (the autoAck = false above), you must also call Channel.basicAckto acknowledge that you have successfully received the message:

由于此例使用[手动确认](https://www.rabbitmq.com/confirms.html)(the autoAck = false above)，所以你需要调用`Channel.basicAckto`确认已经成功接收到了消息（译者注：一般在成功对接收到的消息处理完毕后进行确认）。

```java
// ...
channel.basicAck(method.deliveryTag, false); // acknowledge receipt of the message
}
```

## [Handling unroutable messages](https://www.rabbitmq.com/api-guide.html#returning)

## [处理无法路由的消息](https://www.rabbitmq.com/api-guide.html#returning)

If a message is published with the "mandatory" flags set, but cannot be routed, the broker will return it to the sending client (via an AMQP.Basic.Return command).

如果发布的消息设置了`mandatory`标识，但是没有被成功路由，代理会将其返回给发送的客户端（通过`AMQP.Basic.Return`命令）。

To be notified of such returns, clients can implement the ReturnListener interface and call Channel.addReturnListener. If the client has not configured a return listener for a particular channel, then the associated returned messages will be silently dropped.

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

A return listener will be called, for example, if the client publishes a message with the "mandatory" flag set to an exchange of "direct" type which is not bound to a queue.

例如，客户端发布了一条带有`mandatory`标识的消息，此消息设置了交换机类型为“直连”，但是交换机并没有绑定到队列上，此时退还监听就会被调用。

## [Shutdown Protocol](https://www.rabbitmq.com/api-guide.html#shutdown)

## [关闭协议](https://www.rabbitmq.com/api-guide.html#shutdown)

### [Overview of the Client Shutdown Process](https://www.rabbitmq.com/api-guide.html#shutdown-overview)

### [客户端关闭进程概览](https://www.rabbitmq.com/api-guide.html#shutdown-overview)

The AMQP 0-9-1 connection and channel share the same general approach to managing network failure, internal failure, and explicit local shutdown.

AMQP 0-9-1连接和通道使用相同的通用方法来管理网络故障，内部故障和显式本地关闭。

The AMQP 0-9-1 connection and channel have the following lifecycle states:

AMQP 0-9-1连接和通道具有以下生命周期状态：

- open: the object is ready to use
- 打开：对象可以使用了
- closing: the object has been explicitly notified to shut down locally, has issued a shutdown request to any supporting lower-layer objects, and is waiting for their shutdown procedures to complete
- 正在关闭：已经明确通知对象在本地进行关闭，已经向所有支持的底层对象发出了关闭请求，并且正在等待其关闭过程完成
- closed: the object has received all shutdown-complete notification(s) from any lower-layer objects, and as a consequence has shut itself down
- 已关闭：对象已经接收到所有底层对方发出的关闭完成的通知，然后自己也完成了关闭操作。

Those objects always end up in the closed state, regardless of the reason that caused the closure, like an application request, an internal client library failure, a remote network request or network failure.

这些对象只管完成关闭状态，而不关心造成关闭的原因是什么。像应用请求、客户端内部库错误、远程网络请求或者网络错误一概不管。

The connection and channel objects possess the following shutdown-related methods:

连接和通道对象会处理如下所示的跟关闭有关的方法：

- addShutdownListener(ShutdownListener listener) and
- `addShutdownListener(ShutdownListener listener)` 和
- removeShutdownListener(ShutdownListener listener), to manage any listeners, which will be fired when the object transitions to closed state. Note that, adding a ShutdownListener to an object that is already closed will fire the listener immediately
- `removeShutdownListener(ShutdownListener listener)`用来管理监听器，当对象转换为关闭状态时触发。需要注意的是，给一个已经关闭的对象添加关闭监听器会立即出发监听行为。
- getCloseReason(), to allow the investigation of what was the reason of the object's shutdown
- `getCloseReason()`用来获取对象关闭的原因
- isOpen(), useful for testing whether the object is in an open state
- `isOpen()`在测试对象开启状态时很有用
- close(int closeCode, String closeMessage), to explicitly notify the object to shut down
- `close(int closeCode, String closeMessage)`用来显式地通知对象执行关闭操作

Simple usage of listeners would look like:

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

### [Information about the circumstances of a shutdown](https://www.rabbitmq.com/api-guide.html#shutdown-cause)

### [有关关闭情况的信息](https://www.rabbitmq.com/api-guide.html#shutdown-cause)

One can retrieve the ShutdownSignalException, which contains all the information available about the close reason, either by explicitly calling the getCloseReason() method or by using the cause parameter in the service(ShutdownSignalException cause) method of the ShutdownListener class.

可以通过显式调用`getCloseReason()`方法或通过使用带有`cause`参数的`ShutdownListener`类的`service(ShutdownSignalException cause)`方法来获取`ShutdownSignalException`，其中包含有关关闭原因的所有可用信息。

The ShutdownSignalException class provides methods to analyze the reason of the shutdown. By calling the isHardError()method we get information whether it was a connection or a channel error, and getReason() returns information about the cause, in the form an AMQP method - either AMQP.Channel.Close or AMQP.Connection.Close (or null if the cause was some exception in the library, such as a network communication failure, in which case that exception can be retrieved with getCause()).

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

Use of the isOpen() method of channel and connection objects is not recommended for production code, because the value returned by the method is dependent on the existence of the shutdown cause. The following code illustrates the possibility of race conditions:

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

Instead, we should normally ignore such checking, and simply attempt the action desired. If during the execution of the code the channel of the connection is closed, a ShutdownSignalException will be thrown indicating that the object is in an invalid state. We should also catch for IOException caused either by SocketException, when broker closes the connection unexpectedly, or ShutdownSignalException, when broker initiated clean close.

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

## [Advanced Connection options](https://www.rabbitmq.com/api-guide.html#advanced-connection)

## [高级连接选项](https://www.rabbitmq.com/api-guide.html#advanced-connection)

### [Consumer Operation Thread Pool](https://www.rabbitmq.com/api-guide.html#consumer-thread-pool)

### [消费者操作线程池](https://www.rabbitmq.com/api-guide.html#consumer-thread-pool)

Consumer threads (see [Receiving](https://www.rabbitmq.com/api-guide.html#consuming) below) are automatically allocated in a new ExecutorService thread pool by default. If greater control is required supply an ExecutorService on the newConnection() method, so that this pool of threads is used instead. Here is an example where a larger thread pool is supplied than is normally allocated:

默认情况下，消费者线程（参见下边的 [接收](https://www.rabbitmq.com/api-guide.html#consuming) ）会通过一个新的`ExecutorService`线程池分配。如果需要更大的控制权，可以使用`newConnection()`去应用`ExecutorService`以进行替代。这是一个应用一个比常规分配额更大的线程池的示例：

```java
ExecutorService es = Executors.newFixedThreadPool(20);
Connection conn = factory.newConnection(es);
```

Both Executors and ExecutorService classes are in the java.util.concurrent package.

`Executors`和`ExecutorService`类都在` java.util.concurrent package`里。

When the connection is closed a default ExecutorService will be shutdown(), but a user-supplied ExecutorService (like esabove) will *not* be shutdown(). Clients that supply a custom ExecutorService must ensure it is shutdown eventually (by calling its shutdown() method), or else the pool's threads may prevent JVM termination.

当连接关闭时，默认提供的`ExecutorService`也会执行`shutdown()`，但是用户提供的`ExecutorService`（如上所示）则*不会*执行`shutdown()`。提供自定义`ExecutorService`的客户端必须确保其最终会被关闭（即调用`shutdown()` 方法），否则线程池会影响JVM的中止。

The same executor service may be shared between multiple connections, or serially re-used on re-connection but it cannot be used after it is shutdown().

相同的executor服务可能会被多个连接共享，或者接连不断的重复使用、重复连接，但是无论如何当它关闭后是不可以再用的。

Use of this feature should only be considered if there is evidence that there is a severe bottleneck in the processing of Consumer callbacks. If there are no Consumer callbacks executed, or very few, the default allocation is more than sufficient. The overhead is initially minimal and the total thread resources allocated are bounded, even if a burst of consumer activity may occasionally occur.

应该在有证据表明处理消费回调存在严重瓶颈时才去考虑使用这个功能。如果没有或者只有少量消费者回调需要执行，那默认分配的线程就足够了。即使偶尔会有消费者活动陡增的情况，最初的负载是很小的，并且线程资源的分配和不能无限扩大。

### [Using Lists of Hosts](https://www.rabbitmq.com/api-guide.html#address-array)

### [主机列表的使用](https://www.rabbitmq.com/api-guide.html#address-array)

It is possible to pass an Address array to newConnection(). An Address is simply a convenience class in the com.rabbitmq.client package with *host* and *port* components.

把一个`Address`数组传给`newConnection()`是没问题的。`Address`是一个`com.rabbitmq.client package`中包含 *主机* 和 *端口*组件的简单的便捷类。

For example:

例如：

```java
Address[] addrArr = new Address[]{ new Address(hostname1, portnumber1)
                                 , new Address(hostname2, portnumber2)};
Connection conn = factory.newConnection(addrArr);
```

will attempt to connect to hostname1:portnumber1, and if that fails to hostname2:portnumber2. The connection returned is the first in the array that succeeds (without throwing IOException). This is entirely equivalent to repeatedly setting host and port on a factory, calling factory.newConnection() each time, until one of them succeeds.

这样会先去尝试连接` hostname1:portnumber1`，失败的话会再尝试`hostname2:portnumber2`。返回的连接对象是第一次成功的数组元素的(没抛出`IOException`的话)。这跟分别设置主机和端口然后依次调用`factory.newConnection()`直到成功的操作一毛一样。

If an ExecutorService is provided as well (using the form factory.newConnection(es, addrArr)) the thread pool is associated with the (first) successful connection.

如果同时也提供了`ExecutorService`（在`factory.newConnection(es, addrArr)`中使用），那线程池也是对应的第一次成功连接的那个。

If you want more control over the host to connect to, see [the support for service discovery](https://www.rabbitmq.com/api-guide.html#service-discovery-with-address-resolver).

如果想要实现连接的更多控制，参见 [服务发现支持](https://www.rabbitmq.com/api-guide.html#service-discovery-with-address-resolver).

### [Service discovery with the AddressResolver interface](https://www.rabbitmq.com/api-guide.html#service-discovery-with-address-resolver)

### [使用AddressResolver接口实现服务发现](https://www.rabbitmq.com/api-guide.html#service-discovery-with-address-resolver)

It is possible to use an implementation of AddressResolver to change the endpoint resolution algorithm used at connection time:

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

Just like with [a list of hosts](https://www.rabbitmq.com/api-guide.html#address-array), the first Address returned will be tried first, then the second if the client fails to connect to the first, and so on.

就跟 [主机列表](https://www.rabbitmq.com/api-guide.html#address-array)一样，先尝试返回的第一个`Address`，如果失败了再试第二个，直到成功为止。

If an ExecutorService is provided as well (using the form factory.newConnection(es, addressResolver)) the thread pool is associated with the (first) successful connection.

如果同时也提供了`ExecutorService`（在`factory.newConnection(es, addrArr)`中使用），那线程池也是对应的第一次成功连接的那个。

The AddressResolver is the perfect place to implement custom service discovery logic, which is especially useful in a dynamic infrastructure. Combined with [automatic recovery](https://www.rabbitmq.com/api-guide.html#recovery), the client can automatically connect to nodes that weren't even up when it was first started. Affinity and load balancing are other scenarios where a custom AddressResolver could be useful.

`AddressResolver`是实现自定义服务发现逻辑的最佳方式，在动态基础设施的状况下尤其有用。结合 [自动发现]](https://www.rabbitmq.com/api-guide.html#recovery)，客户端可以自动连接到首次启动时尚未出现故障的节点。姻亲和负载均衡是自定义`AddressResolver`能做的另外两个情景。

The Java client ships with the following implementations (see the javadoc for details):

Java客户端附带了以下实现（详见javadoc）：

- DnsRecordIpAddressResolver: given the name of a host, returns its IP addresses (resolution against the platform DNS server). This can be useful for simple DNS-based load balancing or failover.
- `DnsRecordIpAddressResolver`：根据给定的主机名，返回其IP地址（针对DNS服务器平台的解析）。这对简单的基于DNS的如在均衡很帮助很大。
- DnsSrvRecordAddressResolver: given the name of a service, returns hostname/port pairs. The search is implemented as a DNS SRV request. This can be useful when using a service registry like [HashiCorp Consul](https://www.consul.io/).
- `DnsSrvRecordAddressResolver`：根据给定的服务的名字，返回其所在的主机名/端口对。搜索服务基于DNS SRV请求实现。如果需要类似于[HashiCorp Consul](https://www.consul.io/)的服务注册功能的话，这也相当实用。

### [Heartbeat Timeout](https://www.rabbitmq.com/api-guide.html#heartbeats-timeout)

### [心跳超时](https://www.rabbitmq.com/api-guide.html#heartbeats-timeout)

See the [Heartbeats guide](https://www.rabbitmq.com/heartbeats.html) for more information about heartbeats and how to configure them in the Java client.

想了解更多关于心跳的信息和如何在java客户端进行配置，请参阅[Heartbeats guide](https://www.rabbitmq.com/heartbeats.html) 

### [Custom Thread Factories](https://www.rabbitmq.com/api-guide.html#thread-factories)

### [自定义线程工厂](https://www.rabbitmq.com/api-guide.html#thread-factories)

Environments such as Google App Engine (GAE) can [restrict direct thread instantiation](https://developers.google.com/appengine/docs/java/#Java_The_sandbox). To use RabbitMQ Java client in such environments, it's necessary to configure a custom ThreadFactory that uses an appropriate method to instantiate threads, e.g. GAE's ThreadManager.

类似 Google App Engine (GAE) 的环境能够 [限制直接将线程实例化](https://developers.google.com/appengine/docs/java/#Java_The_sandbox).想要在这种环境里使用RabbitMQ Java客户端，就需要使用适当的方法配置自定义的`ThreadFactory`来实例化线程，例如GAE 的 `ThreadManager`.

Below is an example for Google App Engine.

以下是针对Google App Engine的示例：

```java
import com.google.appengine.api.ThreadManager;

ConnectionFactory cf = new ConnectionFactory();
cf.setThreadFactory(ThreadManager.backgroundThreadFactory());
```

### [Support for Java non-blocking IO](https://www.rabbitmq.com/api-guide.html#java-nio)

### [支持Java的非阻塞IO]](https://www.rabbitmq.com/api-guide.html#java-nio)

Version 4.0 of the Java client brings support for Java non-blocking IO (a.k.a Java NIO). NIO isn't supposed to be faster than blocking IO, it simply allows to control resources (in this case, threads) more easily.

Java客户端4.0版本带来了对Java非阻塞IO（也叫Java NIO）的支持。NIO的目的不是为了比阻塞IO更快，而是为了更方便的实现简单的资源控制（这里指的是线程）。

With the default blocking IO mode, each connection uses a thread to read from the network socket. With the NIO mode, you can control the number of threads that read and write from/to the network socket.

在默认的阻塞IO模式下，每个连接使用一个线程去从读取网络socket读取内容。在NIO模式下，你可以控制读取和写入网络socket的线程的数量。

Use the NIO mode if your Java process uses many connections (dozens or hundreds). You should use fewer threads than with the default blocking mode. With the appropriate number of threads set, you shouldn't experience any decrease in performance, especially if the connections are not so busy.

如果你的Java进程使用了很多连接（数百个之多）的情况下，可以使用NIO模式。你需要使用比默认阻塞模式下更少的线程。设置合适的线程数量的情况下，你不会损失任何性能，特别是连接不是特别繁忙的情况下。

NIO must be enabled explicitly:

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

The NIO mode uses reasonable defaults, but you may need to change them according to your own workload. Some of the settings are: the total number of IO threads used, the size of buffers, a service executor to use for the IO loops, parameters for the in-memory write queue (write requests are enqueued before being sent on the network). Please read the Javadoc for details and defaults.

虽然NIO模式默认值是合理的，但是你也有可能需要根据自身的工作负载来对其进行修改。其中一些设置包括：使用的总的IO线程数量，缓存大小，IO循环所使用的服务执行器（service executor），内存中写队列的参数（将请求发送到网络之前写入队列）。阅读Javadoc来了解更多细节和默认值。

## [Automatic Recovery From Network Failures](https://www.rabbitmq.com/api-guide.html#recovery)

## [自动恢复网络故障](https://www.rabbitmq.com/api-guide.html#recovery)

### [Connection Recovery](https://www.rabbitmq.com/api-guide.html#connection-recovery)

### [恢复连接](https://www.rabbitmq.com/api-guide.html#connection-recovery)

Network connection between clients and RabbitMQ nodes can fail. RabbitMQ Java client supports automatic recovery of connections and topology (queues, exchanges, bindings, and consumers).

客户端和RabbitMQ节点之间的网络连接会发生失败。RabbitMQ的Java客户端支持自动恢复连接和拓扑（队列，交换机，绑定和消费者）。

The automatic recovery process for many applications follows the following steps:

多应用的自动恢复过程依照一下步骤进行：

- Reconnect
- 重连
- Restore connection listeners
- 恢复连接监听
- Re-open channels
- 重开通道
- Restore channel listeners
- 恢复通道监听
- Restore channel basic.qos setting, publisher confirms and transaction settings
- 恢复通道的`basic.qos`设置，发布确认和事务设置

Topology recovery includes the following actions, performed for every channel
拓扑的恢复包含以下动作，会应用到每个通道

- Re-declare exchanges (except for predefined ones)
- 重新声明交换机（预定义的除外）
- Re-declare queues
- 重新声明队列
- Recover all bindings
- 恢复所有绑定
- Recover all consumers
- 恢复所有消费者

As of version 4.0.0 of the Java client, automatic recovery is enabled by default (and thus topology recovery as well).
在Java客户端4.0.0版本中，自动恢复默认是开启的（拓扑的恢复也一样）。

Topology recovery relies on a per-connection cache of entities (queues, exchanges, bindings, consumers). When, say, a queue is declared on a connection, it will be added to the cache. When it is deleted or is scheduled for deletion (e.g. because it is [auto-deleted](https://www.rabbitmq.com/queues.html)) it will be removed. This model has some limitations covered below.

拓扑恢复依赖于实体的每个连接缓存（队列，交换机，绑定，消费者）。当声明一个队列的时候，此队列也会被添加到缓存中。当它被删除或者列入删除计划时（例如是一个 [自动删除](https://www.rabbitmq.com/queues.html)队列），缓存会被移除。此模型有以下限制。

To disable or enable automatic connection recovery, use the factory.setAutomaticRecoveryEnabled(boolean) method. The following snippet shows how to explicitly enable automatic recovery (e.g. for Java client prior 4.0.0):

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

If recovery fails due to an exception (e.g. RabbitMQ node is still not reachable), it will be retried after a fixed time interval (default is 5 seconds). The interval can be configured:

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

### [When Will Connection Recovery Be Triggered?](https://www.rabbitmq.com/api-guide.html#recovery-triggers)

### [连接的自动恢复何时会被触发?](https://www.rabbitmq.com/api-guide.html#recovery-triggers)

Automatic connection recovery, if enabled, will be triggered by the following events:

如果开启了连接自动恢复，会依照一下时间来进行触发：

- An I/O exception is thrown in connection's I/O loop
- 连接的I/O循环中抛出了I/O异常
- A socket read operation times out
- socket读操作超时
- Missed server [heartbeats](https://www.rabbitmq.com/heartbeats.html) are detected
- 检测到服务器丢失 [心跳](https://www.rabbitmq.com/heartbeats.html)
- Any other unexpected exception is thrown in connection's I/O loop
- 连接的I/O循环中抛出了其他不可预期的异常

whichever happens first.

以先发生的为准。

If initial client connection to a RabbitMQ node fails, automatic connection recovery won't kick in. Applications developers are responsible for retrying such connections, logging failed attempts, implementing a limit to the number of retries and so on. Here's a very basic example:

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

When a connection is closed by the application via the Connection.Close method, connection recovery will not be initiated.

当连接被应用使用`Connection.Close`方法关闭的的情况下，连接恢复不会启动。

Channel-level exceptions will not trigger any kind of recovery as they usually indicate a semantic issue in the application (e.g. an attempt to consume from a non-existent queue).

通道级异常不会触发任何恢复，因为这些异常通常指出的是应用程序中的语义问题。

### [Recovery Listeners](https://www.rabbitmq.com/api-guide.html#recovery-listeners)

### [恢复监听](https://www.rabbitmq.com/api-guide.html#recovery-listeners)

It is possible to register one or more recovery listeners on recoverable connections and channels. When connection recovery is enabled, connections returned by ConnectionFactory#newConnection and Connection#createChannel implement com.rabbitmq.client.Recoverable, providing two methods with fairly descriptive names:

可以在可恢复的连接上注册一个或多个恢复监听。当连接恢复开启时，`ConnectionFactory#newConnection`和`Connection#createChannel`返回的连接实现了 `com.rabbitmq.client.Recoverable`，并且提供了两个相当具有描述性名字的方法。

- `addRecoveryListener`
- `removeRecoveryListener`

Note that you currently need to cast connections and channels to Recoverable in order to use those methods.

请注意，当前需要将连接和通道强制转换为`Recoverable`才能使用这些方法。

### [Effects on Publishing](https://www.rabbitmq.com/api-guide.html#publishers)

### [对发布的影响](https://www.rabbitmq.com/api-guide.html#publishers)

Messages that are published using Channel.basicPublish when connection is down will be lost. The client does not enqueue them for delivery after connection has recovered. To ensure that published messages reach RabbitMQ applications need to use [Publisher Confirms](https://www.rabbitmq.com/confirms.html) and account for connection failures.

当连接失效时，通过`Channel.basicPublish`发布的消息会丢失掉。客户端不会将其放入队列以用连接恢复后进行投递。想要确认发布的消息是否已经到达RabbitMQ，应用需要使用 [发布确认](https://www.rabbitmq.com/confirms.html) 并且解决连接失败。

### [Topology Recovery](https://www.rabbitmq.com/api-guide.html#topology-recovery)

### [拓扑恢复](https://www.rabbitmq.com/api-guide.html#topology-recovery)

Topology recovery involves recovery of exchanges, queues, bindings and consumers. It is enabled by default when automatic recovery is enabled. Topology recovery is enabled by default in modern versions of the client.

拓扑的恢复涉及到交换机、队列、绑定和消费者的恢复。自动恢复启用时拓扑恢复也会随之启用。客户端的当代版本都默认启用了拓扑恢复。

Topology recovery can be disabled explicitly if needed:
有需要的话，拓扑恢复可以显式的关闭：

```java
ConnectionFactory factory = new ConnectionFactory();

Connection conn = factory.newConnection();
// enable automatic recovery (e.g. Java client prior 4.0.0)
factory.setAutomaticRecoveryEnabled(true);
// disable topology recovery
factory.setTopologyRecoveryEnabled(false);
```

### [Failure Detection and Recovery Limitations](https://www.rabbitmq.com/api-guide.html#automatic-recovery-limitations)

### [故障检测和恢复的限制](https://www.rabbitmq.com/api-guide.html#automatic-recovery-limitations)

Automatic connection recovery has a number of limitations and intentional design decisions that applications developers need to be aware of.

连接自动恢复有一些局限性和应用程序开发人员需要注意的有意设计的策略。

Topology recovery relies on a per-connection cache of entities (queues, exchanges, bindings, consumers). When, say, a queue is declared on a connection, it will be added to the cache. When it is deleted or is scheduled for deletion (e.g. because it is [auto-deleted](https://www.rabbitmq.com/queues.html)) it will be removed. This makes it possible to declare and delete entities on different channels without having unexpected results. It also means that consumer tags (a channel-specific identifier) must be unique across all channels on connections that use automatic connection recovery.

拓扑恢复依赖于实体的每个连接缓存（队列，交换机，绑定，消费者）。当声明一个队列的时候，此队列也会被添加到缓存中。当它被删除或者列入删除计划时（例如是一个 [自动删除](https://www.rabbitmq.com/queues.html)队列），缓存会被移除。这样就可以在不同的通道上声明和删除实体，而不会产生意外的结果。这也意味着，使用自动连接恢复的消费者标签（特定通道的标识符）在所有通道上必须是唯一的。

When a connection is down or lost, it [takes time to detect](https://www.rabbitmq.com/heartbeats.html). Therefore there is a window of time in which both the library and the application are unaware of effective connection failure. Any messages published during this time frame are serialised and written to the TCP socket as usual. Their delivery to the broker can only be guaranteed via [publisher confirms](https://www.rabbitmq.com/confirms.html): publishing in AMQP 0-9-1 is entirely asynchronous by design.

当连接断开或丢失时，需要花费一些时间进行检测。因此，库和应用程序意识到有连接失败之前有一个窗口期。在这端时间内发布的所有消息都将照常进行序列化并写入TCP套接字。只有通过[发布者确认](https://www.rabbitmq.com/confirms.html)才能保证将它们成功交付给了代理：按照设计，AMQP 0-9-1的发布过程完全是异步的。

When a socket or I/O operation error is detected by a connection with automatic recovery enabled, recovery begins after a configurable delay, 5 seconds by default. This design assumes that even though a lot of network failures are transient and generally short lived, they do not go away in an instant. Having a delay also avoids an inherent race condition between server-side resource cleanup (such as [exclusive or auto-delete queue](https://www.rabbitmq.com/queues.html) deletion) and operations performed on a newly opened connection on the same resources.

如果在启用了自动恢复的连接中检测到套接字或I / O操作错误，则恢复将在默认的5秒延迟后开始（这个延迟时间是可配置的）。 该设计假定即使许多网络故障是暂时的并且通常持续时间很短，但也不会立即就恢复。 延迟还可以避免在相同连接上发生服务器端资源清除（例如[独占或自动删除队列](https://www.rabbitmq.com/queues.html)删除）和打开新连接之间的资源竞争。

Connection recovery attempts by default will continue at identical time intervals until a new connection is successfully opened. Recovery delay can be made dynamic by providing a RecoveryDelayHandler implementation instance to ConnectionFactory#setRecoveryDelayHandler. Implementations that use dynamically computed delay intervals should avoid values that are too low (as a rule of thumb, lower than 2 seconds).

默认情况下，连接恢复尝试将以相同的时间间隔进行，直到成功打开新连接为止。 通过将实现`RecoveryDelayHandler`的实例化对象提供给`ConnectionFactory＃setRecoveryDelayHandler`，可以实现恢复延迟的动态化。 实现动态计算的延迟间隔应避免使用过低的值（根据经验，小于2秒就算过低了）。

When a connection is in the recovering state, any publishes attempted on its channels will be rejected with an exception. The client currently does not perform any internal buffering of such outgoing messages. It is an application developer's responsibility to keep track of such messages and republish them when recovery succeeds. [Publisher confirms](https://www.rabbitmq.com/confirms.html) is a protocol extension that should be used by publishers that cannot afford message loss.

当连接处于恢复状态时，在其通道上尝试进行的任何发布都将被拒绝，但也有例外。 客户端当前不对此类传出消息执行任何内部缓冲。 跟踪此类消息并在恢复成功后重新发布它们是应用程序开发人员的责任。 [发布者确认](https://www.rabbitmq.com/confirms.html)是协议的扩展，发布者可以利用其避免消息丢失。

Connection recovery will not kick in when a channel is closed due to a channel-level exception. Such exceptions often indicate application-level issues. The library cannot make an informed decision about when that's the case.

当发生通道级别异常而导致通道干壁时，连接恢复不会生效。 此类异常通常表示的是应用程序级别的问题。 库无法在这种情况下采取适合的应对措施。

Closed channels won't be recovered even after connection recovery kicks in. This includes both explicitly closed channels and the channel-level exception case above.

如果通道的关闭是通过显式关闭的或者是由于上述通道级别异常引发的。 即使启动了连接恢复，也不会恢复已关闭的通道。

### [Manual Acknowledgements and Automatic Recovery](https://www.rabbitmq.com/api-guide.html#recovery-and-acknowledgements)

### [手动确认和自动恢复](https://www.rabbitmq.com/api-guide.html#recovery-and-acknowledgements)

When manual acknowledgements are used, it is possible that network connection to RabbitMQ node fails between message delivery and acknowledgement. After connection recovery, RabbitMQ will reset delivery tags on all channels.

当使用手动确认的情况下，可能会发生连接在消息投递成功但并未进行确认的空挡中失效的情况。在连接恢复后，RabbitMQ会在通道里重置投递标签。

This means that *basic.ack*, *basic.nack*, and *basic.reject* with old delivery tags will cause a channel exception. To avoid this, RabbitMQ Java client keeps track of and updates delivery tags to make them monotonically growing between recoveries.

这意味着旧的投递标签的*basic.ack*, *basic.nack*, 和 *basic.reject*  会导致通道异常发生。为了避免这种情况，RabbitMQ的Java客户端会保持对投递标签的追踪和更新，以使它们在恢复过程中单调增长。

Channel.basicAck, Channel.basicNack, and Channel.basicReject then translate adjusted delivery tags into those used by RabbitMQ.

之后，`Channel.basicAck`, `Channel.basicNack`, 和 `Channel.basicReject`会将调整后的传递标签转换为RabbitMQ使用的传递标签。

Acknowledgements with stale delivery tags will not be sent. Applications that use manual acknowledgements and automatic recovery must be capable of handling redeliveries.

带有过时的投递标签的确认将不会发送。使用手动确认和自动恢复的应用必须能够对重新投递的消息进行处理。

### [Channels Lifecycle and Topology Recovery](https://www.rabbitmq.com/api-guide.html#recovery-channel-lifecycle)

### [通道的生命周期和拓扑恢复](https://www.rabbitmq.com/api-guide.html#recovery-channel-lifecycle)

Automatic connection recovery is meant to be as transparent as possible for the application developer, that's why Channelinstances remain the same even if several connections fail and recover behind the scenes. Technically, when automatic recovery is on, Channel instances act as proxies or decorators: they delegate the AMQP business to an actual AMQP channel implementation and implement some recovery machinery around it. That is why you shouldn't close a channel after it has created some resources (queues, exchanges, bindings) or topology recovery for those resources will fail later, as the channel has been closed. Instead, leave creating channels open for the life of the application.

连接自动恢复对应用程序开发人员来说应尽可能透明，这就是为什么即使好几个连接失效，然后在后台恢复的情况下，`Channel`实例任然会保持相同的原因。 从技术上讲，启用自动恢复时，通道实例充当代理或装饰器：它们将AMQP业务委派给实际的AMQP通道实现，并围绕它实施一些恢复机制。 这就是为什么您不应该在通道完成了一些资源创建（队列，交换，绑定）之后对其进行关闭，这会导致稍后的扑恢复失败。 在应用程序的整个生命周期中都应该保持通道的打开状态。

## [Unhandled Exceptions](https://www.rabbitmq.com/api-guide.html#unhandled-exceptions)

## [未处理的异常](https://www.rabbitmq.com/api-guide.html#unhandled-exceptions)

Unhandled exceptions related to connection, channel, recovery, and consumer lifecycle are delegated to the exception handler. Exception handler is any object that implements the ExceptionHandler interface. By default, an instance of DefaultExceptionHandler is used. It prints exception details to the standard output.

跟连接、通道、恢复和消费者生命周期有关的未处理的异常会委托给“异常处理”。“异常处理”是对`ExceptionHandler`接口的一个实现。默认情况下，会使用`DefaultExceptionHandler`实例。它会把异常的细节打印到标准输出中。

It is possible to override the handler using ConnectionFactory#setExceptionHandler. It will be used for all connections created by the factory:

也可以用`ConnectionFactory#setExceptionHandler`来覆盖默认异常处理。这会应用到所有通过工厂创建的连接中。

```java
ConnectionFactory factory = new ConnectionFactory();
cf.setExceptionHandler(customHandler);
```

Exception handlers should be used for exception logging. 

异常处理应该将异常记录到日志当中。

## [Metrics and Monitoring](https://www.rabbitmq.com/api-guide.html#metrics)

## [指标和监控](https://www.rabbitmq.com/api-guide.html#metrics)

The client collects runtime metrics (e.g. number of published messages) for active connections. Metric collection is optional feature that should be set up at the ConnectionFactory level, using the setMetricsCollector(metricsCollector) method. This method expects a MetricsCollector instance, which is called in several places of the client code.

客户端会收集活动的连接的运行时指标（例如发布消息的数量）。指标收集是需要在`ConnectionFactory`级别使用`setMetricsCollector(metricsCollector)`方法进行配置的可选功能。此方法需要的`MetricsCollector`实例会在客户端代码中的多处用到。

The client supports [Micrometer](http://micrometer.io/) (as of version 4.3) and [Dropwizard Metrics](http://metrics.dropwizard.io/) out of the box.

4.3版本的客户端开始支持 [Micrometer](http://micrometer.io/) 和 [Dropwizard Metrics](http://metrics.dropwizard.io/) ，开箱即用。

Here are the collected metrics:

以下是收集的指标：

- Number of open connections
- 开启的连接数量
- Number of open channels
- 开启的通道数量
- Number of published messages
- 发布的消息数量
- Number of consumed messages
- 消费的消息数量
- Number of acknowledged messages
- 确认的消息数量
- Number of rejected messages
- 拒绝的消息数量

Both Micrometer and Dropwizard Metrics provide counts, but also mean rate, last five minute rate, etc, for messages-related metrics. They also support common tools for monitoring and reporting (JMX, Graphite, Ganglia, Datadog, etc). See the dedicated sections below for more details.

Micrometer 和 Dropwizard Metrics 都提供了与消息指标相关的计数器，也提供了平均速率，最后5分钟速率等。他们也支持用于监控和报告的通用工具（如 JMX, Graphite, Ganglia, Datadog等）。更多详细信息，下边会专门进行说明。

Developers should keep a few things in mind when enabling metric collection.

启用指标收集时，开发人员应牢记一些注意事项。

- Don't forget to add the appropriate dependencies (in Maven, Gradle, or even as JAR files) to JVM classpath when using Micrometer or Dropwizard Metrics. Those are optional dependencies and will not be pulled automatically with the Java client. You may also need to add other dependencies depending on the reporting backend(s) used.
- 要使用Micrometer 或者 Dropwizard Metrics 的话，别忘了添加相关依赖（ (在Maven, Gradle, 或者 JAR 文件里)）到JVM classpath中。
- Metrics collection is extensible. Implementing a custom MetricsCollector for specific needs is encouraged.
- 指标的收集是可以扩展的。推荐为特定目的实现自定义的`MetricsCollector`。
- The MetricsCollector is set at the ConnectionFactory level but can be shared across different instances.
- 虽然`MetricsCollector`是在`ConnectionFactory`层定义的，但是也可以在不同的实例中共享。
- Metrics collection doesn't support transactions. E.g. if an acknowledgment is sent in a transaction and the transaction is then rolled back, the acknowledgment is counted in the client metrics (but not by the broker obviously). Note the acknowledgment is actually sent to the broker and then cancelled by the transaction rollback, so the client metrics are correct in term of acknowledgments sent. As a summary, don't use client metrics for critical business logic, they're not guaranteed to be perfectly accurate. They are meant to be used to simplify reasoning about a running system and make operations more efficient.
- 指标的收集不支持事务。举例来说，如果一个确认（acknowledgment）通过事务发送，然后事务回滚了，那这个确认就已经被客户端指标（很显然不是通过代理）累计了。需要注意的是，确认（acknowledgment）确实已经发送到代理了，然后又被事务回滚给清除了，所以客户端指标对于确认的处理是没问题的。总之，不要把客户端指标用作关键的业务逻辑，因为不保证它们完全准确无误。它们存在的目的在于简单的解释系统的运行情况并且让操作更具效率。

### [Micrometer support](https://www.rabbitmq.com/api-guide.html#metrics-micrometer)

### [支持Micrometer](https://www.rabbitmq.com/api-guide.html#metrics-micrometer)

Metric collection has to be enabled first:

首先指标收集已经被开启了：

[Micrometer](http://micrometer.io/) the following way:

[Micrometer](http://micrometer.io/) 按照以下方式:

```java
ConnectionFactory connectionFactory = new ConnectionFactory();
MicrometerMetricsCollector metrics = new MicrometerMetricsCollector();
connectionFactory.setMetricsCollector(metrics);
...
metrics.getPublishedMessages(); // get Micrometer's Counter object
```

Micrometer supports [several reporting backends](http://micrometer.io/docs): Netflix Atlas, Prometheus, Datadog, Influx, JMX, etc.

Micrometer支持 [多种报告后台](http://micrometer.io/docs)：Netflix Atlas, Prometheus, Datadog, Influx, JMX, 等。

You would typically pass in an instance of MeterRegistry to the MicrometerMetricsCollector. Here is an example with JMX:

通常情况下会将`MeterRegistry`的实例传给`MicrometerMetricsCollector`，这里是使用JMX的示例：

```java
JmxMeterRegistry registry = new JmxMeterRegistry();
MicrometerMetricsCollector metrics = new MicrometerMetricsCollector(registry);
ConnectionFactory connectionFactory = new ConnectionFactory();
connectionFactory.setMetricsCollector(metrics);
```

### [Dropwizard Metrics support](https://www.rabbitmq.com/api-guide.html#metrics-dropwizard-metrics)

### [支持Dropwizard Metrics](https://www.rabbitmq.com/api-guide.html#metrics-dropwizard-metrics)

Enable metrics collection with [Dropwizard](http://metrics.dropwizard.io/) like so:

如下开启指标收集的 [Dropwizard](http://metrics.dropwizard.io/) 支持：

```java
ConnectionFactory connectionFactory = new ConnectionFactory();
StandardMetricsCollector metrics = new StandardMetricsCollector();
connectionFactory.setMetricsCollector(metrics);
...
metrics.getPublishedMessages(); // get Metrics' Meter object
```

Dropwizard Metrics supports [several reporting backends](http://metrics.dropwizard.io/3.2.3/getting-started.html): console, JMX, HTTP, Graphite, Ganglia, etc.

You would typically pass in an instance of MetricsRegistry to the StandardMetricsCollector. Here is an example with JMX:

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

## [RabbitMQ Java Client on Google App Engine](https://www.rabbitmq.com/api-guide.html#gae-pitfalls)

## [Google App Engine上的RabbitMQ Java客户端](https://www.rabbitmq.com/api-guide.html#gae-pitfalls)

Using RabbitMQ Java client on Google App Engine (GAE) requires using a custom thread factory that instantiates thread using GAE's ThreadManager (see above). In addition, it is necessary to set a low heartbeat interval (4-5 seconds) to avoid running into the low InputStream read timeouts on GAE:

要在Google App Engine (GAE)上使用RabbitMQ Java客户端的话，需要使用GAE's ThreadManager (see above)这个自定义的线程工具去实例化线程。另外需要设置一个比较低的心跳间隔（4-5秒）来避免GAE上的InputStream读取超时过低。

```java
ConnectionFactory factory = new ConnectionFactory();
cf.setRequestedHeartbeat(5);
```

## [Caveats and Limitations](https://www.rabbitmq.com/api-guide.html#cache-pitfalls)

## [注意事项和限制](https://www.rabbitmq.com/api-guide.html#cache-pitfalls)

To make topology recovery possible, RabbitMQ Java client maintains a cache of declared queues, exchanges, and bindings. The cache is per-connection. Certain RabbitMQ features make it impossible for clients to observe some topology changes, e.g. when a queue is deleted due to TTL. RabbitMQ Java client tries to invalidate cache entries in the most common cases:

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

However, the client cannot track these topology changes beyond a single connection. Applications that rely on auto-delete queues or exchanges, as well as queue TTL (note: not message TTL!), and use [automatic connection recovery](https://www.rabbitmq.com/api-guide.html#connection-recovery), should explicitly delete entities know to be unused or deleted, to purge client-side topology cache. This is facilitated by Channel#queueDelete, Channel#exchangeDelete, Channel#queueUnbind, and Channel#exchangeUnbind being idempotent in RabbitMQ 3.3.x (deleting what's not there does not result in an exception).

但是，客户端无法跟踪单个连接以外的拓扑变化。 依赖于自动删除队列或交换机以及队列TTL（请注意：不是消息TTL！）和使用[连接自动恢复](https://www.rabbitmq.com/api-guide.html#connection-recovery)的应用应该显式删除已知的未被使用或已被删除的实体，以清除客户端拓扑缓存。 `Channel＃queueDelete`，`Channel＃exchangeDelete`，`Channel＃queueUnbind`和`Channel＃exchangeUnbind`在RabbitMQ 3.3.x中是幂等的（删除不存在的内容不会导致异常）这个特性有助于实现这一操作。

## [The RPC (Request/Reply) Pattern: an Example](https://www.rabbitmq.com/api-guide.html#rpc)

## [远程过程调用-RPC (请求/回复)模式: 示例](https://www.rabbitmq.com/api-guide.html#rpc)

As a programming convenience, the Java client API offers a class RpcClient which uses a temporary reply queue to provide simple [RPC-style communication](https://www.rabbitmq.com/tutorials/tutorial-six-java.html) facilities via AMQP 0-9-1.

为了方便编写程序，Java客户端提供了使用一个临时回复队列来实现的`RpcClient`类，这样就通过AMQP 0-9-1 实现了简单的 [RPC-风格通讯](https://www.rabbitmq.com/tutorials/tutorial-six-java.html)。

The class doesn't impose any particular format on the RPC arguments and return values. It simply provides a mechanism for sending a message to a given exchange with a particular routing key, and waiting for a response on a reply queue.

此类没有在RPC属性和返回值方面新增任何特殊格式。它只是简单的实现了附带路由键发送消息到给定的交换机，并且等待在回复队列里等待回应的机制。

```java
import com.rabbitmq.client.RpcClient;

RpcClient rpc = new RpcClient(channel, exchangeName, routingKey);
```

(The implementation details of how this class uses AMQP 0-9-1 are as follows: request messages are sent with the basic.correlation_id field set to a value unique for this RpcClient instance, and with basic.reply_to set to the name of the reply queue.)

（此类使用AMQP 0-9-1的实现细节如下：发送请求消息时，其`basic.correlation_id`字段设置为该`RpcClient`实例的唯一值，而`basic.reply_to`设置为回复队列的名称。）

Once you have created an instance of this class, you can use it to send RPC requests by using any of the following methods:

一旦创建了此类的实例，就可以使用一下任一方法来发送RPC请求了：

```java
byte[] primitiveCall(byte[] message);
String stringCall(String message)
Map mapCall(Map message)
Map mapCall(Object[] keyValuePairs)
```

The primitiveCall method transfers raw byte arrays as the request and response bodies. The method stringCall is a thin convenience wrapper around primitiveCall, treating the message bodies as String instances in the default character encoding.

`primitiveCall`方法将原始字节数组作为请求和响应主体进行传输。 方法`stringCall`是`primitiveCall`的一个轻量化封装，将消息正文作为默认字符编码的String实例来处理。

The mapCall variants are a little more sophisticated: they encode a java.util.Map containing ordinary Java values into an AMQP 0-9-1 binary table representation, and decode the response in the same way. (Note that there are some restrictions on what value types can be used here - see the javadoc for details.)

`mapCall`这种变体稍微有点复杂：他们将包含普通Java值的`java.util.Map`编码为AMQP 0-9-1二进制表来表示，并以相同的方式来对收到的响应进行解码。 （注意，此处可以使用哪种值类型有一些限制，详细信息请参见javadoc。）

All the marshalling/unmarshalling convenience methods use primitiveCall as a transport mechanism, and just provide a wrapping layer on top of it.

所有编组/解组的便捷方法都使用`primitiveCall`作为传输机制，仅在它之上提供包装层。

## [TLS Support](https://www.rabbitmq.com/api-guide.html#tls)

## [TLS 支持](https://www.rabbitmq.com/api-guide.html#tls)

It's possible to encrypt the communication between the client and the broker [using TLS](https://www.rabbitmq.com/ssl.html). Client and server authentication (a.k.a. peer verification) is also supported. Here is the simplest, most naive way to use encryption with the Java client:

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

Note the client doesn't enforce any server authentication ([peer certificate chain verification](https://www.rabbitmq.com/ssl.html#peer-verification)) in the above sample as the default, "trust all certificates" TrustManager is used. This is convenient for local development but **prone to man-in-the-middle attacks** and therefore [not recommended for production](https://www.rabbitmq.com/production-checklist.html).

注意，以上样例客户端默认不强制任何服务器验证（[对等证书链认证](https://www.rabbitmq.com/ssl.html#peer-verification)），使用了”信任所有证书“的`TrustManager`。这在本地开发的时候很是方便，但是**容易收到中间人攻击**，所以 [不推荐在用在生产环境中](https://www.rabbitmq.com/production-checklist.html)

To learn more about TLS support in RabbitMQ, see the [TLS guide](https://www.rabbitmq.com/ssl.html). If you only want to configure the Java client (especially the peer verification and trust manager parts), read [the appropriate section](https://www.rabbitmq.com/ssl.html#java-client) of the TLS guide.

想要对RabbitMQ的TSL支持进行进一步学习，可以参见[TLS 指南](https://www.rabbitmq.com/ssl.html)。如果只打算配置Java客户端（特别是对等认证和信任管理者部分），可以只阅读一下TLS指南的 [相关部分](https://www.rabbitmq.com/ssl.html#java-client) 即可。

## [OAuth 2 Support](https://www.rabbitmq.com/api-guide.html#oauth2-support)

## [OAuth 2 支持](https://www.rabbitmq.com/api-guide.html#oauth2-support)

The client can authenticate against an OAuth 2 server like [UAA](https://github.com/cloudfoundry/uaa). The [OAuth 2 plugin](https://github.com/rabbitmq/rabbitmq-auth-backend-oauth2) must be enabled on the server side and configured to use the same OAuth 2 server as the client.

客户端可以通过 [UAA](https://github.com/cloudfoundry/uaa)这样的OAuth 2 服务器来进行身份认证。服务器端需要启用 [OAuth 2 插件](https://github.com/rabbitmq/rabbitmq-auth-backend-oauth2)，并配置跟客户端使用同一个OAuth 2 服务器 。

### [Getting the OAuth 2 Token](https://www.rabbitmq.com/api-guide.html#oauth2-getting-token)

### [获取 OAuth 2 令牌](https://www.rabbitmq.com/api-guide.html#oauth2-getting-token)

The Java client provides the OAuth2ClientCredentialsGrantCredentialsProvider class to get a JWT token using the [OAuth 2 Client Credentials flow](https://tools.ietf.org/html/rfc6749#section-4.4). The client will send the JWT token in the password field when opening a connection. The broker will then verify the JWT token signature, validity, and permissions before authorising the connection and granting access to the requested virtual host.

Java客户端提供了`OAuth2ClientCredentialsGrantCredentialsProvider`类，用来从[OAuth 2 客户端凭证流](https://tools.ietf.org/html/rfc6749#section-4.4)获取JWT令牌。客户端会在打开连接的时候将令牌放在password字段中进行发送。然后代理会在授权之前验证JWT令牌的签名、有效性和权限，并授予对请求的虚拟主机的访问权限。

Prefer the use of OAuth2ClientCredentialsGrantCredentialsProviderBuilder to create an OAuth2ClientCredentialsGrantCredentialsProvider instance and then set it up on the ConnectionFactory. The following snippet shows how to configure and create an instance of the OAuth 2 credentials provider for the [example setup of the OAuth 2 plugin](https://github.com/rabbitmq/rabbitmq-auth-backend-oauth2#examples):

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

In production, make sure to use HTTPS for the token endpoint URI and configure the SSLContext if necessary for the HTTPS requests (to verify and trust the identity of the OAuth 2 server). The following snippet does so by using the tls().sslContext() method from OAuth2ClientCredentialsGrantCredentialsProviderBuilder:

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

Please consult the [Javadoc](https://rabbitmq.github.io/rabbitmq-java-client/api/current/com/rabbitmq/client/impl/OAuth2ClientCredentialsGrantCredentialsProvider.html) to see all the available options.

更多选项请参照[Javadoc](https://rabbitmq.github.io/rabbitmq-java-client/api/current/com/rabbitmq/client/impl/OAuth2ClientCredentialsGrantCredentialsProvider.html)。

### [Refreshing the Token](https://www.rabbitmq.com/api-guide.html#oauth2-refreshing-token)

### [刷新令牌](https://www.rabbitmq.com/api-guide.html#oauth2-refreshing-token)

Tokens expire and the broker will refuse operations on connections with expired tokens. To avoid this, it is possible to call CredentialsProvider#refresh() before expiration and send the new token to the server. This is cumbersome from an application point of view, so the Java client provides help with the DefaultCredentialsRefreshService. This utility tracks used tokens, refreshes them before they expire, and send the new tokens for the connections it is responsible for.

令牌是会或过期的，代理会拒绝带有过期令牌的连接所请求的操作。可以使用`CredentialsProvider#refresh()`在令牌过期前使用新令牌发送请求，以防止此情况的发生。应用自己来实现是比较麻烦的，所以Java客户端提供了`DefaultCredentialsRefreshService`来给予一定帮助。这个工具用来追踪使用的令牌，在过期前进行刷新，并将新令牌发送给所负责的连接。

The following snippet shows how to create a DefaultCredentialsRefreshService instance and set it up on the ConnectionFactory:

以下代码片段展示了如何创建`DefaultCredentialsRefreshService`实例，并且将其配置到`ConnectionFactory`上。

```java
import com.rabbitmq.client.impl.DefaultCredentialsRefreshService.
        DefaultCredentialsRefreshServiceBuilder;
...
CredentialsRefreshService refreshService =
  new DefaultCredentialsRefreshServiceBuilder().build();
cf.setCredentialsRefreshService(refreshService);
```

The DefaultCredentialsRefreshService schedules a refresh after 80% of the token validity time, e.g. if the token expires in 60 minutes, it will be refreshed after 48 minutes. This is the default behaviour, please consult the [Javadoc](https://rabbitmq.github.io/rabbitmq-java-client/api/current/com/rabbitmq/client/impl/DefaultCredentialsRefreshService.html) for more information.

`DefaultCredentialsRefreshService`会在令牌有效期超过80%后进行刷新，例如，如果令牌在60分钟后过期，`DefaultCredentialsRefreshService`会在48分钟的时候进行刷新。这是默认的行为，更多的细节可以通过 [Javadoc](https://rabbitmq.github.io/rabbitmq-java-client/api/current/com/rabbitmq/client/impl/DefaultCredentialsRefreshService.html) 了解。

## Getting Help and Providing Feedback

## 获取帮助和提供建议

If you have questions about the contents of this guide or any other topic related to RabbitMQ, don't hesitate to ask them on the [RabbitMQ mailing list](https://groups.google.com/forum/#!forum/rabbitmq-users).

如果你对本指南的内容或RabbitMQ的其他主题有任何疑问。可以通多 [RabbitMQ 邮件列表](https://groups.google.com/forum/#!forum/rabbitmq-users)进行提问。
