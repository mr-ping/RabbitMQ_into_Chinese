# 准备
本教程假设RabbitMQ已经安装并运行在本地主机的标准端口（5672）上。如果你是用不同的主机、端口或凭据，则需要调整连接设置。

# 去哪里寻求帮助

如果你在学习本教程时遇到了困难，你可以通过[邮件列表](https://groups.google.com/forum/#!forum/rabbitmq-users)或[RabbitMQ社区Slack](https://rabbitmq-slack.herokuapp.com/)联系我们。
# 介绍
RabbitMQ是一个消息中转器：它接收并转发消息。你可以把它想象成一个邮局：当你把一封信放进邮箱后，你可以肯定，邮局最终会将你的信送给收件人。在这个比喻中，RabbitMQ是邮箱、邮局以及邮递员。

RabbitMQ和邮局的主要区别在于它不处理纸张，而是接收、存储和转发二进制数据块，即消息。

一般来说，RabbitMQ和消息传递会用到一些术语。
- 生产者：发送消息的程序叫做生产者。
  
  ![生产者](https://www.rabbitmq.com/img/tutorials/producer.png)
- 队列：RabbitMQ中的邮箱就是队列，虽然消息会在程序之间传递，但它们都存储于队列中。队列只会收到主机内存和磁盘大小的限制，本质上它是一个较大的消息缓冲区。生产者们可以将消息发送进同一个队列，消费者们也可以从同一个队列接收消息。

  ![队列](https://www.rabbitmq.com/img/tutorials/queue.png)

- 消费者：和生产者类似，接收消息的程序叫做消费者。

  ![消费者](https://www.rabbitmq.com/img/tutorials/consumer.png)

**注意**：生产者、队列、消费者以及中转器并不需要在同一台主机上，事实上，在大多数的应用程序中也确实如此。一个应用程序既可以是生产者，也可以是消费者。
# Hello World
## (使用Java客户端)
在本部分中，我们将用Java编写两个程序，生产者发送单个消息，消费者接收消息并将其打印出来。我们将忽略Java API中的一些细节，从这个非常简单的事情开始。这个消息传递的Hello World。

在下图中，P是生产者，C是消费者，中间的盒子是一个队列——RabbitMQ为消费者保存消息的一个缓冲区。

![Hello World](https://www.rabbitmq.com/img/tutorials/python-one.png)
> ### Java客户端库
> RabbitMQ支持多种协议，本教程使用AMQP 0-9-1，这是一个用于消息传递的开源通用协议。RabbitMQ[有很多不同语言](https://rabbitmq.com/devtools.html)的客户端。我们将使用RabbitMQ提供的Java客户端。
> 下载客户端库及其依赖项（[SLF4J API](https://repo1.maven.org/maven2/org/slf4j/slf4j-api/1.7.26/slf4j-api-1.7.26.jar)和[SLF4J Simple](https://repo1.maven.org/maven2/org/slf4j/slf4j-simple/1.7.26/slf4j-simple-1.7.26.jar)）。复制这些文件到您的工作目录中。
> 请注意SLF4J对于本教程来说已经足够了，但是在实际的生产环境中您应该使用成熟的日志库，比如[Logback](https://logback.qos.ch/)。
> （RabbitMQ的Java客户端也在Maven的中心库中，groupId是`com.rabbitmq`，artifactId是`amqp-client`。

现在我们有了Java客户端及其依赖，我们可以编写一些代码。
## 发送
![发送](https://www.rabbitmq.com/img/tutorials/sending.png)

我们将把消息发布者（发送方）命名为Send，并将消息接受者命名为Recv。发布者会连接到RabbitMQ，发送一条消息，然后退出。

在[Send.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)中，我们需要导入一些类：
```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
```
设置类并命名队列：
```java
public class Send {
  private final static String QUEUE_NAME = "hello";
  public static void main(String[] argv) throws Exception {
      ...
  }
}
```
接着我们可以创建一个到服务的连接：
```java
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
try (Connection connection = factory.newConnection();
     Channel channel = connection.createChannel()) {
}
```
连接抽象了套接字连接，并为我们处理了版本协议、身份验证等等。这里我们连接到一个本地RabbitMQ节点，也就是localhost。如果我们想要连接到另一台机器上的节点，我们只需在这里指定它的主机名或IP地址。

接下来，我们创建一个通道，用于完成任务的大部分API都在这个通道中。注意，我们可以使用try-with-resources语句，因为Connection和Channel都实现了java.io.Closeable。这样我们就不需要在代码中显示的关闭它们。

至于发送，我们必须声明一个要发送的队列；然后我们可以向这个队列中发送一条消息，所有这些都在try-with-resources语句中：
```java
channel.queueDeclare(QUEUE_NAME, false, false, false, null);
String message = "Hello World!";
channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
```
声明队列是幂等的——只有当队列不存在的时候才会创建它。消息内容是一个字节数组，因此你可以在这里编码任何你喜欢的内容。

[这里是整个Send.java类](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)

> ### 发送不能工作！
> 如果这是你第一次使用RabbitMQ，而你没有看到“发送”的消息，那么你可能会抓耳挠腮，想知道哪里出了问题。可能中转器启动时没有足够的空闲磁盘空间（默认情况下，它至少需要200MB空闲空间），因此拒绝接收消息。检查中转器日志文件，确认并在必要时减少限制。配置文件文档将向您展示如何设置“disk_free_limit”。

## 接收
对于我们的发布者来说就是这样。我们的消费者监听来自RabbitMQ的消息，所以不像发布者只发布一条消息，我们会让消费者一直运行监听消息并把它们打印出来。

![消费者](https://www.rabbitmq.com/img/tutorials/receiving.png)

代码(在[Recv.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java))和`发送`几乎有相同的导入:
```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;
```

我们将使用额外的`DeliverCallback`接口来缓冲服务器推送给我们的消息。

设置与发行商是一样的；我们打开一个连接和一个通道，并声明将要使用的队列。注意，这与`发送`发布到的队列相匹配。
```java
public class Recv {

  private final static String QUEUE_NAME = "hello";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

  }
}
```
注意，我们在这里也声明了队列。因为我们可能会在发布者之前启动消费者，所以我们希望在尝试使用来自队列的消息之前确保队列存在。

为什么不使用try-with-resource语句来自动关闭通道和连接呢？通过这样做，我们只需让程序继续运行，关闭所有内容，然后退出！这可能会很尴尬，因为我们希望在消费者异步侦听要到达的消息时，流程保持活动状态。

我们将告诉服务器将队列中的消息传递给我们。因为它将异步地向我们推送消息，所以我们提供了一个对象形式的回调，该回调将缓冲消息，直到我们准备好使用它们。这就是`DeliverCallback`子类所做的。
```java
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
    String message = new String(delivery.getBody(), "UTF-8");
    System.out.println(" [x] Received '" + message + "'");
};
channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> { });
```
[这里是整个Recv.java类](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java)
## 把它们放在一起
你可以只在类路径中使用RabbitMQ的java客户端来编译这两个java文件：
```powershell
javac -cp amqp-client-5.7.1.jar Send.java Recv.java
```
要运行它们，你需要`rabbitmq-client.jar`及其类路径上的依赖项。在终端中，运行消费者（接收者）：
```powershell
java -cp .:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar Recv
```
接着运行发布者（发送者）：
```powershell
java -cp .:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar Send
```
在Windows上，使用分号而不是冒号来分隔路径中的项。

消费者将通过RabbitMQ打印从发布者那里得到的消息。消费者将继续运行，等待消息(使用Ctrl-C停止它)，因此请尝试从另一个终端运行发布者。

> ### 队列清单
> 你可能想看看RabbitMQ有哪些队列以及其中有多少条消息。你可以（作为特权用户）使用rabbitmqctl工具：
> ```powershell
> sudo rabbitmqctl list_queues
> ```
> 在Windows上忽略sudo：
> ```powershell
> rabbitmqctl.bat list_queues
> ```

接下来是[第2部分](https://www.rabbitmq.com/tutorials/tutorial-two-java.html)，构建一个简单的*工作队列*。

> ### 提示
> 为了节省输入，您可以为类路径设置一个环境变量，例如：
> ```powershell
> export CP=.:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar
> java -cp $CP Send
> ```
> 或者在Windows上：
> ```powershell
> set CP=.;amqp-client-5.7.1.jar;slf4j-api-1.7.26.jar;slf4j-simple-1.7.26.jar
> java -cp %CP% Send
> ```