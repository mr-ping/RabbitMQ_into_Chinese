# Routing

在之前的教程中，我们构建了一个简单的日志记录系统。我们能够向许多接收器广播日志信息。

在本教程中，我们将为它添加一个特性——我们将使只订阅消息的一个子集成为可能。例如，我们将能够只将关键错误消息定向到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有日志消息。

## 绑定

在前面的示例中，我们已经创建了绑定。你可能会想起这样的代码：

```java
channel.queueBind(queueName, EXCHANGE_NAME, "");
```

绑定是交换机和队列之间的关系。可以简单地理解为：队列对来自此交换的消息感兴趣。

绑定可以接受一个额外的`routingKey`参数。为了避免与`basic_publish`参数混淆，我们将其称为`绑定键（binding key）`。下面是我们如何创建一个带有键的绑定：

```java
channel.queueBind(queueName, EXCHANGE_NAME, "black");
```

绑定键的含义取决于交换类型。我们之前使用的扇出交换机，完全忽略了它的价值。

## 直接交换机（Direct Exchange）

在前面的教程中，我们的日志系统将所有消息广播给所有用户。我们希望扩展它，允许根据消息的严重性过滤消息。例如，我们可能希望程序将日志消息写入磁盘，以便只接收关键错误，而不浪费磁盘空间用于警告（warning）或信息（info）类型的日志消息。

我们使用的是扇出交换机（fanout），这没有给我们太多的灵活性——它只能进行无脑的广播。

我们将改用直接交换机（direct）。直接交换背后的路由算法很简单——消息发送到绑定键（binding key）与消息的路由键（routing key）完全匹配的队列。

为了说明这一点，考虑以下设置：

![img](https://www.rabbitmq.com/img/tutorials/direct-exchange.png)

在这个设置中，我们可以看到绑定了两个队列的直接交换机X。第一个队列绑定的绑定键为`orange`，第二个队列有两个绑定，一个绑定键为`black`，另一个绑定键为`green`。

在这种设置中，使用路由键`orange`发布到交换器的消息将被路由到队列Q1。带有`black`或`green`路由键的消息将转到Q2。所有其他消息将被丢弃。

## 多重绑定

![img](https://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

使用相同的绑定键绑定多个队列是完全合法的。在我们的示例中，我们可以使用绑定键`black`在X和Q1之间添加一个绑定。在这种情况下，直接交换机将表现得像扇出，并将消息广播到所有匹配的队列。路由键为`black`的消息将同时发送到Q1和Q2。

## 发送日志

我们将在日志系统中使用这个模型。我们将发送信息到一个直接交换机，而不是扇出交换机。我们将提供日志严重级别作为路由键。这样，接收程序将能够选择它想要接收的严重性。让我们首先关注发出日志。

与往常一样，我们需要首先创建一个交换：

```java
channel.exchangeDeclare(EXCHANGE_NAME, "direct");
```

并且我们已经准备好发送消息

```java
channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());
```

为了简化问题，我们假设“严重性”可以是“info”、“warning”、“error”中的一个。

## 订阅

接收消息的工作方式与之前的教程一样，只有一个例外——我们将为每个感兴趣的严重性创建一个新的绑定。

```java
String queueName = channel.queueDeclare().getQueue();

for(String severity : argv){
  channel.queueBind(queueName, EXCHANGE_NAME, severity);
}
```

## 把它们放在一起

![img](https://www.rabbitmq.com/img/tutorials/python-four.png)

`EmitLogDirect.java`的源代码：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class EmitLogDirect {

  private static final String EXCHANGE_NAME = "direct_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    try (Connection connection = factory.newConnection();
         Channel channel = connection.createChannel()) {
        channel.exchangeDeclare(EXCHANGE_NAME, "direct");

        String severity = getSeverity(argv);
        String message = getMessage(argv);

        channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes("UTF-8"));
        System.out.println(" [x] Sent '" + severity + "':'" + message + "'");
    }
  }
  //..
}
```

`ReceiveLogsDirect.java`的源代码：

```java
import com.rabbitmq.client.*;

public class ReceiveLogsDirect {

  private static final String EXCHANGE_NAME = "direct_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "direct");
    String queueName = channel.queueDeclare().getQueue();

    if (argv.length < 1) {
        System.err.println("Usage: ReceiveLogsDirect [info] [warning] [error]");
        System.exit(1);
    }

    for (String severity : argv) {
        channel.queueBind(queueName, EXCHANGE_NAME, severity);
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

像往常一样编译（参见教程一了解编译和类路径建议）。为了方便起见，在运行示例时，我们将使用环境变量$CP（在Windows上是%CP%）作为类路径。

```bash
javac -cp $CP ReceiveLogsDirect.java EmitLogDirect.java
```

如果你只想保存`warning`和`error`（而不是`info`）日志消息到文件中，只需打开控制台并输入：

```bash
java -cp $CP ReceiveLogsDirect warning error > logs_from_rabbit.log
```

如果你想在你的屏幕上看到所有的日志信息，打开一个新的终端并执行：

```bash
java -cp $CP ReceiveLogsDirect info warning error
# => [*] Waiting for logs. To exit press CTRL+C
```

例如要发出`error`日志消息，只需输入：

```bash
java -cp $CP EmitLogDirect error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
```

完整代码：[EmitLogDirect.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/EmitLogDirect.java)，[ReceiveLogsDirect.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/ReceiveLogsDirect.java)

移步[第5教程](https://www.rabbitmq.com/tutorials/tutorial-five-java.html)，了解如何基于模式侦听消息。