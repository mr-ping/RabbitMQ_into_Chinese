# Work Queues

在[第一个教程](https://www.rabbitmq.com/tutorials/tutorial-one-java.html)中，我们编写了从命名队列发送和接收消息的程序。在本例中，我们将创建一个工作队列，用于在多个工作程序之间分发耗时的任务。

工作队列（又名：任务队列）背后的主要思想是避免立即执行资源密集型任务，并不得不等待它完成。相反，我们把要完成的任务安排到以后。我们将任务封装为消息并将其发送到队列。一个在后台运行的辅助进程将弹出任务并最终执行作业。当您运行多个工作程序时，任务将在它们之间共享。

这个概念在web应用程序中特别有用，因为在一个简短的HTTP请求窗口中不可能处理复杂的任务。

## 准备

在本教程的前一部分中，我们发送了一个包含“Hello World!”的消息。现在我们将发送代表复杂任务的字符串。我们没有一个真实的任务，比如要调整图像大小或要渲染pdf文件，所以让我们假装我们很忙——通过使用`Thread.sleep()`函数。我们用字符串中点的数量作为复杂度；每个点代表一秒钟的“工作”。例如，一个假任务由`Hello…`只需要三秒钟。

我们将稍微修改前面示例中的Send.java代码，以允许从命令行发送任意消息。这个程序将把任务调度到我们的工作队列中，所以我们将它命名为`NewTask.java`：

```java
String message = String.join(" ", argv);

channel.basicPublish("", "hello", null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
```

我们的旧的Recv.java程序也需要一些更改：它需要为消息体中的每个点假装一秒钟的工作。它将处理传递的消息并执行任务，所以我们将其命名为Worker.java：

```java
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
  String message = new String(delivery.getBody(), "UTF-8");

  System.out.println(" [x] Received '" + message + "'");
  try {
    doWork(message);
  } finally {
    System.out.println(" [x] Done");
  }
};
boolean autoAck = true; // acknowledgment is covered below
channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, consumerTag -> { });
```

模拟执行时间的假任务：

```java
private static void doWork(String task) throws InterruptedException {
    for (char ch: task.toCharArray()) {
        if (ch == '.') Thread.sleep(1000);
    }
}
```

像在第一教程中那样编译它们(将jar文件放在工作目录中，并带上将环境变量`CP`参数):

```bash
javac -cp $CP NewTask.java Worker.java
```

## 循环调度

使用任务队列的优点之一是能够轻松地并行处理工作。如果我们积压了大量的工作，我们可以增加更多的工作程序，这样就很容易扩大规模。

首先，让我们尝试同时运行两个工作程序实例。它们都将从队列中获取消息，但具体如何获取呢?让我们来看看。

您需要打开三个控制台。两个将运行工作程序。这些控制台将是我们的两个消费者——C1和C2。

```bash
# shell 1
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
```

```bash
# shell 2
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
```

在第三个控制台中，我们将发布新的任务。一旦你已经开始运行消费者程序，你就可以发布一些消息：

```bash
# shell 3
java -cp $CP NewTask First message.
# => [x] Sent 'First message.'
java -cp $CP NewTask Second message..
# => [x] Sent 'Second message..'
java -cp $CP NewTask Third message...
# => [x] Sent 'Third message...'
java -cp $CP NewTask Fourth message....
# => [x] Sent 'Fourth message....'
java -cp $CP NewTask Fifth message.....
# => [x] Sent 'Fifth message.....'
```

我们一起看看什么传递给了我们的工作程序：

```bash
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

```bash
java -cp $CP Worker
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

默认情况下，RabbitMQ会将每条消息依次发送给下一个消费者。平均下来，每个消费者将获得相同数量的消息。这种分发消息的方式称为轮循。在三个或更多的工作程序上试试这个方法。

## 消息确认

完成一项任务可能需要几秒钟。你可能想知道，如果一个消费者开始了一项很长的任务，但只完成了一部分就死了，会发生什么。在我们目前的代码中，一旦RabbitMQ向消费者发送了一条消息，它就会立即标记删除。在这种情况下，如果您杀死一个工作程序，我们将丢失它正在处理的消息。我们还会丢失所有已分派给这个特定消费者但尚未处理的消息。

但我们不想失去任何任务。如果一个消费者死了，我们希望将任务交付给另一个消费者。

为了确保消息不会丢失，RabbitMQ支持[消息确认](https://www.rabbitmq.com/confirms.html)。一个确认信息被消费者发送回来，告诉RabbitMQ一个特定的消息已经被接收、处理，并且RabbitMQ可以删除它。

如果一个消费者在没有发送确认的情况下死亡(通道关闭，连接关闭，或者TCP连接丢失)，RabbitMQ会明白消息没有被完全处理，并将其重新排队。如果有其他消费者在同一时间在线，然后它将迅速重新交付给另一个消费者。这样你就可以确保没有信息丢失，即使消费者偶尔死亡。

在消费者交付确认时强制执行超时(默认为30分钟)。这有助于检测从不确认交付的有缺陷（卡住）的消费者。您可以按照[交付确认超时中的说明](https://www.rabbitmq.com/consumers.html#acknowledgement-timeout)增加该超时时间。

默认情况下，[手动消息确认](https://www.rabbitmq.com/confirms.html)是打开的。在前面的例子中，我们通过`autoAck=true`标志显式地关闭了它们。是时候将此标志设置为`false`，并在完成任务后，从消费者那里发送适当的确认。

```java
channel.basicQos(1); // accept only one unack-ed message at a time (see below)

DeliverCallback deliverCallback = (consumerTag, delivery) -> {
  String message = new String(delivery.getBody(), "UTF-8");

  System.out.println(" [x] Received '" + message + "'");
  try {
    doWork(message);
  } finally {
    System.out.println(" [x] Done");
    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
  }
};
boolean autoAck = false;
channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, consumerTag -> { });
```

使用这段代码，我们可以确保即使使用CTRL+C杀死正在处理消息的工作程序，也不会丢失任何东西。在消费者死亡后不久，所有未确认的消息将被重新发送。

确认必须在接收交付的同一通道上发送。尝试确认使用不同的通道将导致通道级协议异常。查看[确认操作指南](https://www.rabbitmq.com/confirms.html)以了解更多信息。

> ### 被遗忘的确认
>
> 丢失确认是一个常见的错误。这是一个很容易犯的错误，但后果是严重的。当你的客户端退出时，消息会被重新传递(看起来像是随机的重新传递)，但是RabbitMQ会消耗越来越多的内存，因为它无法释放任何未被释放的消息。
>
> 为了调试这种错误，你可以使用`rabbitmqctl`来打印`messages_unacknowledged`字段：
>
> ```bash
> sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
> ```
>
> 在Windows中，去掉sudo：
>
> ```bash
> rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
> ```

## 消息的持久化

我们已经学习了如何确保即使消费者死亡，任务也不会丢失。但是如果RabbitMQ服务器停止了，我们的任务仍然会丢失。

当RabbitMQ退出或崩溃时，它会丢失队列和消息，除非你告诉它不要这样做。要确保消息不会丢失，需要做两件事:我们需要将队列和消息都标记为持久性的。

首先，我们需要确保队列能够在RabbitMQ节点重启后存活。为了做到这一点，我们需要声明它是持久的：

```java
boolean durable = true;
channel.queueDeclare("hello", durable, false, false, null);
```

虽然这个命令本身是正确的，但它在我们当前的设置中不能工作。这是因为我们已经定义了一个名为`hello`的队列，它不是持久队列。RabbitMQ不允许你用不同的参数重新定义一个已经存在的队列，并且会给任何试图这样做的程序返回一个错误。但是有一个快速的解决方法——让我们用不同的名称声明一个队列，例如`task_queue`：

```java
boolean durable = true;
channel.queueDeclare("task_queue", durable, false, false, null);
```

这个`queueDeclare`更改需要同时应用于生产者和消费者代码。

此时，我们确信即使RabbitMQ重启，task_queue队列也不会丢失。现在，我们需要通过将`MessageProperties`（实现了`BasicProperties`）设置为值`PERSISTENT_TEXT_PLAIN`，将消息标记为持久化消息。

```java
import com.rabbitmq.client.MessageProperties;

channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

>### 关于消息持久性的注意事项
>
>将消息标记为“persistent”并不能完全保证消息不会丢失。虽然它告诉RabbitMQ将消息保存到磁盘，但是RabbitMQ接受消息后仍然有一个很短的时间窗口尚未保存。此外，RabbitMQ不会对每条消息执行`fsync(2)`操作——它可能只是被保存到缓存中，而不是真正写入磁盘。持久性保证并不强，但对于我们的简单任务队列来说已经足够了。如果你需要更有力的保证，你可以使用[发布确认](https://www.rabbitmq.com/confirms.html)。

## 公平分配

您可能已经注意到，调度仍然没有完全按照我们希望的那样工作。例如，在有两个工作程序的情况下，当所有奇数的消息都很大，而偶数消息都很小时，一个工作程序将一直很忙，而另一个几乎不做任何工作。好吧，RabbitMQ对此一无所知，仍然会均匀地发送消息。

这是因为RabbitMQ只在消息进入队列时发送一条消息。它不会查看消费者未确认消息的数量。它只是盲目地将第n条消息发送给第n个消费者。

![img](https://www.rabbitmq.com/img/tutorials/prefetch-count.png)

为了解决这个问题，我们可以使用带有prefetchCount = 1设置的basicQos方法。这就告诉RabbitMQ一次不能给一个工作程序发送多个消息。或者，换句话说，在工作程序处理并确认前一条消息之前，不要将新消息分派给它。相反，它将把它分派给下一个仍然不忙的工作程序。

```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

> ### 注意队列大小
>
> 如果所有的工作程序都很忙，你的队列会排满。你需要关注这一点，也许增加更多的工作程序，或者用一些其他的策略。

## 把它们放在一起

`NewTask.java`最终的样子：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.MessageProperties;

public class NewTask {

  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    try (Connection connection = factory.newConnection();
         Channel channel = connection.createChannel()) {
        channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);

        String message = String.join(" ", argv);

        channel.basicPublish("", TASK_QUEUE_NAME,
                MessageProperties.PERSISTENT_TEXT_PLAIN,
                message.getBytes("UTF-8"));
        System.out.println(" [x] Sent '" + message + "'");
    }
  }
}
```

[NewTask.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/NewTask.java)

`Worker.java`最终的样子：

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class Worker {

  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    final Connection connection = factory.newConnection();
    final Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    channel.basicQos(1);

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");

        System.out.println(" [x] Received '" + message + "'");
        try {
            doWork(message);
        } finally {
            System.out.println(" [x] Done");
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
    };
    channel.basicConsume(TASK_QUEUE_NAME, false, deliverCallback, consumerTag -> { });
  }

  private static void doWork(String task) {
    for (char ch : task.toCharArray()) {
        if (ch == '.') {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException _ignored) {
                Thread.currentThread().interrupt();
            }
        }
    }
  }
}
```

[Worker.java](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Worker.java)

使用消息确认和预取计数，您可以设置一个工作队列。持久性选项让任务在重启RabbitMQ时仍然存在。

有关通道方法和MessageProperties的更多信息，您可以在线浏览[JavaDocs](https://rabbitmq.github.io/rabbitmq-java-client/api/current/)。

现在我们可以继续[教程3](https://www.rabbitmq.com/tutorials/tutorial-three-java.html)，学习如何向许多用户传递相同的信息。