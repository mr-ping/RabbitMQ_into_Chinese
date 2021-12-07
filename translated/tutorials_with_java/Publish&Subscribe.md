# Publish/Subscribe

在[之前的教程](https://www.rabbitmq.com/tutorials/tutorial-two-java.html)中，我们创建了一个工作队列。工作队列背后的假设是，每个任务都恰好交付给一个工作程序。在这一部分中，我们将做一些完全不同的事情——我们将向多个消费者传递一条消息。这种模式称为“发布/订阅”。

为了演示该模式，我们将构建一个简单的日志记录系统。它由两个程序组成——第一个程序将发送日志消息，第二个程序将接收并打印日志消息。

在我们的日志记录系统中，接收程序的每个运行副本都将获得消息。这样我们就可以运行一个接收器并将日志直接发送到磁盘上；同时，我们可以运行另一个接收器，并在屏幕上看到日志。

基本上，已发布的日志消息将被广播到所有的接收器。

## 交换机

在本教程的前面部分中，我们向队列发送和接收消息。现在是时候介绍RabbitMQ中的完整消息传递模型了。

让我们快速回顾一下我们在之前的教程中所涵盖的内容：

- 生产者是发送消息的用户应用程序。
- 队列是存储消息的缓冲区。
- 消费者是接收消息的用户应用程序。

RabbitMQ消息模型的核心思想是生产者从不直接向队列发送任何消息。实际上，生产者常常根本不知道消息是否会被传递到任何队列。

相反，生产者只能向交换机发送消息。交换是一件很简单的事情。一方面它从生产者那里接收消息，另一方面它将消息推送到队列中。交换机必须确切地知道如何处理接收到的消息。它应该被附加到一个特定的队列吗？是否应该将其添加到多个队列中？或者它应该被丢弃。这些规则由*交换类型*定义。

![img](https://www.rabbitmq.com/img/tutorials/exchanges.png)

有几种交换类型可用：直接交换（direct）、主题交换（topic）、头文件交换（headers）和扇出交换（fanout）。我们来关注最后一个——扇出交换（fanout）。让我们创建一个这种类型的交换机，并将其称为logs：

```java
channel.exchangeDeclare("logs", "fanout");
```

扇出交换非常简单。您可能可以从名称中猜到，它只是将接收到的所有消息广播给它知道的所有队列。这正是我们的日志记录系统所需要的。

> ### 获取交换机清单
>
> 要列出服务器上的交换机，你可以运行非常有用的rabbitmqctl：
>
> ```bash
> sudo rabbitmqctl list_exchanges
> ```
>
> 在这个列表中会有一些`amq.*`交换机和默认（未命名）交换机。它们是默认创建的，但目前不太可能需要使用它们。
>
> ### 无名的交换机
>
> 在本教程的前几部分中，我们对交换机一无所知，但仍然能够向队列发送消息。这是可能的，因为我们使用了默认交换机，我们用空字符串""标识它。
>
> 回想一下我们之前是如何发布消息的：
>
> ```java
> channel.basicPublish("", "hello", null, message.getBytes());
> ```
>
> 第一个参数是交换的名称。空字符串表示默认的或没有名称的交换：消息被路由到由`routingKey`指定的名称的队列，如果它存在的话。

现在，我们可以将其发布到命名交换器：

```java
channel.basicPublish( "logs", "", null, message.getBytes());
```

## 临时队列

您可能还记得，之前我们使用了具有特定名称的队列（还记得`hello`和`task_queue`吗？）能够命名一个队列对我们来说至关重要——我们需要将工作程序指向同一个队列。当你想在生产者和消费者之间共享队列时，给队列一个名字是很重要的。

但对于我们的日志系统来说，情况并非如此。我们希望听到所有日志消息，而不仅仅是其中的一个子集。我们也只对当前流动的消息感兴趣，而不是对旧的消息。要解决这个问题，我们需要两样东西。

首先，当我们连接到RabbitMQ时，我们需要一个新的空队列。为此，我们可以创建一个带有随机名称的队列，或者，更好的是——让服务器为我们选择一个随机队列名称。

其次，一旦我们断开了消费者，队列应该被自动删除。

在Java客户端，当我们不给`queueDeclare()`提供参数时，我们会创建一个非持久的、排他的、自动删除的队列，并生成一个名称：

```java
String queueName = channel.queueDeclare().getQueue();
```

您可以在[队列指南](https://www.rabbitmq.com/queues.html)中了解更多关于独占标志和其他队列属性的信息。

此时`queueName`包含一个随机队列名称。例如，它可能看起来像amq.gen-JzTY20BRgKO-HjmUJj0wLg。

## 绑定

![img](https://www.rabbitmq.com/img/tutorials/bindings.png)

我们已经创建了一个扇出交换机和一个队列。现在我们需要告诉交换机将消息发送到我们的队列。交换机和队列之间的关系称为绑定。

```java
channel.queueBind(queueName, "logs", "");
```

从现在开始，`logs`交换机将把消息附加到我们的队列中。

> ### 获取绑定清单
>
> 您可以使用如下命令列出存在的绑定：
>
> ```bash
> rabbitmqctl list_bindings
> ```

## 把它们放在一起

![img](https://www.rabbitmq.com/img/tutorials/python-three-overall.png)

发送日志消息的生产者程序与前面的教程没有太大的不同。最重要的更改是，我们现在希望将消息发布到我们的`logs`交换器，而不是未命名的交换机。我们需要在发送时提供一个`routingKey`，但是它的值在扇出交换机中被忽略。下面是`EmitLog.java`程序的代码：

```java
public class EmitLog {

  private static final String EXCHANGE_NAME = "logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    try (Connection connection = factory.newConnection();
         Channel channel = connection.createChannel()) {
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        String message = argv.length < 1 ? "info: Hello World!" :
                            String.join(" ", argv);

        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes("UTF-8"));
        System.out.println(" [x] Sent '" + message + "'");
    }
  }
}
```

[EmitLog.java源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLog.java)

如您所见，在建立连接之后，我们声明了交换机。这一步是必要的，因为禁止发布消息到不存在的交换机。

如果没有队列绑定到交换机，消息将会丢失，但这对我们来说没有问题；如果没有消费者在收听，我们可以安全地丢弃消息。

`ReceiveLogs.java`代码如下：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class ReceiveLogs {
  private static final String EXCHANGE_NAME = "logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
    String queueName = channel.queueDeclare().getQueue();
    channel.queueBind(queueName, EXCHANGE_NAME, "");

    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");
        System.out.println(" [x] Received '" + message + "'");
    };
    channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
  }
}
```

[ReceiveLogs.java源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogs.java)

像之前一样编译，然后我们就完成了。

```bash
javac -cp $CP EmitLog.java ReceiveLogs.java
```

如果你想将日志保存到文件中，只需打开控制台并输入：

```bash
java -cp $CP ReceiveLogs > logs_from_rabbit.log
```

如果你想在屏幕上看到日志，生成一个新的终端并运行：

```bash
java -cp $CP ReceiveLogs
```

当然，要运行日志类：

```bash
java -cp $CP EmitLog
```

通过使用`rabbitmqctl list_bindings`，你可以验证代码是否真的像我们想的那样创建了绑定和队列。运行两个`Receivellog .java`程序时，你应该看到如下内容：

```bash
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```

结果的解释很简单：来自`logs`交换机的数据被发送到两个具有服务器分配名称的队列。这正是我们想要的。

为了了解如何监听消息子集，让我们继续学习[教程4](https://www.rabbitmq.com/tutorials/tutorial-four-java.html)