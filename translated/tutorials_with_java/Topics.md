# Topics

在之前的教程中，我们改进了日志系统。我们没有使用只能进行虚拟广播的扇出交换机，而是使用了直接交换机，从而获得了选择性接收日志的可能性。

虽然使用直接交换机改进了我们的系统，但它仍然有局限性——它不能基于多个标准进行路由。

在我们的日志系统中，我们可能不仅希望根据严重性订阅日志，还希望根据发出日志的来源订阅日志。您可能从[`syslog`](http://en.wikipedia.org/wiki/Syslog)unix工具中了解到这个概念，它根据严重性（info/warn/crit…）和设施（auth/cron/kern…）来路由日志。

这将给我们带来很大的灵活性——我们可能希望只监听来自`cron`的关键错误，同时也监听来自`kern`的所有日志。

要在我们的日志系统中实现这一点，我们需要了解一个更复杂的主题交换机。

## 主题交换机

发送到主题交换机的消息不能有任意的`routing_key`——它必须是一个由点分隔的单词列表。单词可以是任何东西，但通常它们指定了与信息相关的一些特征。几个有效的routing key示例：`stock.usd.nyse`、`nyse.vmw`、`quick.orange.rabbit`。routing key中可以有任意多的字，最多255字节的限制。

绑定键也必须是相同的形式。主题交换机背后的逻辑类似于直接交换机——使用特定routing key发送的消息将被发送到使用匹配binding key绑定的所有队列。然而，对于绑定键有两个重要的特殊情况:

- `*`只能替换一个单词。

- `#`可以替代零个或多个单词。

用一个例子来解释这一点最简单：

![img](https://www.rabbitmq.com/img/tutorials/python-five.png)

在这个例子中，我们将发送所有描述动物的信息。消息将通过一个由三个单词（两个点）组成的路由键发送。routing key中的第一个词将描述速度，第二个词描述颜色，第三个词描述物种：“speed.colour.species”。

我们创建了三个绑定：Q1使用“`*.orange.*`”绑定，Q2使用“`*.*.rabbit`”和“`lazy.#`”绑定

这些绑定可以概括为:

- Q1对所有橙色的动物感兴趣。


- Q2想要听到关于兔子和懒惰动物的一切。

一个路由键设置为“`quick.orange.rabbit`”的消息将被发送到两个队列。路由键为“`lazy.orange.elephant`”的消息也会去匹配两个队列。另一方面，“`quick.orange.fox`”只会去第一个队列，“`lazy.brown.fox`”只到第二个队列。“`lazy.pink.rabbit`”将只被发送到第二个队列一次，即使它同时匹配Q2的两个绑定。“`quick.brown.fox`”不匹配任何绑定，因此它将被丢弃。

如果我们打破约定，用一个或四个词，比如“`orange`”或“`quick.orange.male.rabbit`”来传递信息，会发生什么？这些消息不会匹配任何绑定，会丢失的。

另一方面，“`lazy.orange.male.rabbit`”，尽管它有四个单词，但它将匹配最后一个绑定，并将被发送到第二个队列。

> ### 主题交换机
>
> 主题交换机很强大，可以像其他交换机一样。
>
> 当队列绑定“#”绑定键时，它将接收所有的消息，而不管路由键是什么——就像在扇出交换机中一样。
>
> 当特殊字符"*"和"#"在绑定中不使用时，主题交换的行为就像直接交换一样。

## 把它们放在一起

我们将在日志记录系统中使用主题交换机。我们首先假设日志的路由键有两个单词：“`<facility>.<severity>`”。

代码几乎与前一教程中的相同。

`EmitLogTopic.java`的代码：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class EmitLogTopic {

  private static final String EXCHANGE_NAME = "topic_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    try (Connection connection = factory.newConnection();
         Channel channel = connection.createChannel()) {

        channel.exchangeDeclare(EXCHANGE_NAME, "topic");

        String routingKey = getRouting(argv);
        String message = getMessage(argv);

        channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes("UTF-8"));
        System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");
    }
  }
  //..
}
```

`ReceiveLogsTopic.java`的代码：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class ReceiveLogsTopic {

  private static final String EXCHANGE_NAME = "topic_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "topic");
    String queueName = channel.queueDeclare().getQueue();

    if (argv.length < 1) {
        System.err.println("Usage: ReceiveLogsTopic [binding_key]...");
        System.exit(1);
    }

    for (String bindingKey : argv) {
        channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);
    }

    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");
        System.out.println(" [x] Received '" +
            delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
    };
    channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
  }
}
```

编译并运行示例，包括[教程1](https://www.rabbitmq.com/tutorials/tutorial-one-java.html)中的类路径——在Windows上使用%CP%。

编译：

```bash
javac -cp $CP ReceiveLogsTopic.java EmitLogTopic.java
```

接收所有日志：

```bash
java -cp $CP ReceiveLogsTopic "#"
```

接收所有来自“`kern`”的日志：

```bash
java -cp $CP ReceiveLogsTopic "kern.*"
```

或者只接收和“`critical`”相关的日志：

```bash
java -cp $CP ReceiveLogsTopic "*.critical"
```

你可以创建多个绑定规则：

```bash
java -cp $CP ReceiveLogsTopic "kern.*" "*.critical"
```

发出一个带有路由键为“`kern.critical`”的日志

```bash
java -cp $CP EmitLogTopic "kern.critical" "A critical kernel error"
```

享受玩这些程序的乐趣。请注意，代码没有对路由键或绑定键进行任何假设，您可能需要使用两个以上的路由键参数。

完整代码： [EmitLogTopic.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLogTopic.java)，[ReceiveLogsTopic.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogsTopic.java)

接下来，在[教程6](https://www.rabbitmq.com/tutorials/tutorial-six-java.html)中了解如何作为远程过程调用执行往返消息