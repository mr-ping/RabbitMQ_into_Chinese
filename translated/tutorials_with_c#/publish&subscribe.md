> ### 前置条件
> 
> 本教程假设RabbitMQ已经[安装](http://www.rabbitmq.com/download.html)在你本机的 (5672)端口。如果你使用了不同的主机、端口或者凭证，连接设置就需要作出一些对应的调整。
> 
> ### 如何获得帮助
> 
> 如果你在使用本教程的过程中遇到了麻烦，你可以通过邮件列表来[联系我们](https://groups.google.com/forum/#!forum/rabbitmq-users)。



## Publish/Subscribe

## 发布/订阅

### (using the .NET Client)

### （使用 .NET 客户端）



In the [previous tutorial](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html) we created a work queue. The assumption behind a work queue is that each task is delivered to exactly one worker. In this part we'll do something completely different -- we'll deliver a message to multiple consumers. This pattern is known as "publish/subscribe".

在[上个教程](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html)中，我们创建了一个工作队列。工作队列假设每个任务只会被推送给一个工作者。这部分，我们会做一些完全不同的事情——我们会将消息投送给多个消费者。这种模式被称为“发布/订阅”。

To illustrate the pattern, we're going to build a simple logging system. It will consist of two programs -- the first will emit log messages and the second will receive and print them.

为了解释此种模式，我们将会建立一个简单的日志系统。它由两个程序组成——第一个会发送日志消息，第二个接收、并将其打印出来。

In our logging system every running copy of the receiver program will get the messages. That way we'll be able to run one receiver and direct the logs to disk; and at the same time we'll be able to run another receiver and see the logs on the screen.

在我们的日志系统中，每一个接收程序的拷贝都会获取到消息。通过这种方式，我们可以做到让其中一个接收者将日志直接存储到硬盘上，同时运行的另一个接收者将日志输出到屏幕上用于查看。

Essentially, published log messages are going to be broadcast to all the receivers.

实质上，发布的日志消息会广播给所有的接收者。

## Exchanges

## 交换机

In previous parts of the tutorial we sent and received messages to and from a queue. Now it's time to introduce the full messaging model in Rabbit.

教程的上个部分里，我们通过一个队列来发送和接收消息。现在，是时候把完整的Rabbit消息模型模型介绍一下了。

Let's quickly go over what we covered in the previous tutorials:

让我们快速过一下上个教程中所涉及的内容。

- A *producer* is a user application that sends messages.
- A *queue* is a buffer that stores messages.
- A *consumer* is a user application that receives messages.
- 一个“生产者”就是一个发送消息的用户应用程序。
- 一个“队列”就是存储消息的缓存。
- 一个“消费者”就是一个接收消息的用户应用程序。

The core idea in the messaging model in RabbitMQ is that the producer never sends any messages directly to a queue. Actually, quite often the producer doesn't even know if a message will be delivered to any queue at all.

RabbitMQ的消息模型中的核心思想就是生产者永远不会将任何消息直接发送给队列。实际上，通常情况下，生产者根本不知道它是否会将消息投送给任何一个队列。

Instead, the producer can only send messages to an *exchange*. An exchange is a very simple thing. On one side it receives messages from producers and the other side it pushes them to queues. The exchange must know exactly what to do with a message it receives. Should it be appended to a particular queue? Should it be appended to many queues? Or should it get discarded. The rules for that are defined by the *exchange type*.

真正的情况是，生产者只能将消息发送给一个*交换机*。交换机是个很简单概念。它做左手收生产者发送的消息，右手就将消息推送给队列。交换机必须明确的知道需要对接收到的消息做些什么。消息是需要追加到一个特定的队列中？是需要追加到多个队列中？还是需要被丢弃掉。*交换机类型(exchange type)*就是用来定义这种规则的。

![img](https://www.rabbitmq.com/img/tutorials/exchanges.png)

There are a few exchange types available: direct, topic, headers and fanout. We'll focus on the last one -- the fanout. Let's create an exchange of this type, and call it logs:

这里有几个可用的交换机类型：直连交换机(`direct`), 主题交换机(`topic`), 头交换机(`headers`) 和扇形交换机(`fanout`)。我们会把关注点放在最后一个上。让我们来创建一个此种类型的交换机，将其命名为`logs`：

```csharp
channel.ExchangeDeclare("logs", "fanout");
```

The fanout exchange is very simple. As you can probably guess from the name, it just broadcasts all the messages it receives to all the queues it knows. And that's exactly what we need for our logger.

扇形交换机非常简单。从名字就可猜出来，它只是负责将消息广播给所有它所知道的队列。这正是我们的日志系统所需要的。

> 
>
> #### Listing exchanges
>
> #### 交换机的监听
>
> To list the exchanges on the server you can run the ever useful rabbitmqctl:
>
> 想要列出服务器上的交换机，可以运行`rabbitmqctl`这个非常有用的程序：
>
> 
>
> ```bash
> sudo rabbitmqctl list_exchanges
> ```
>
> 
>
> In this list there will be some `amq.*` exchanges and the default (unnamed) exchange. These are created by default, but it is unlikely you'll need to use them at the moment.
>
> 在此列表中，会出现一些类似于`amq.*`的交换机以及默认（未命名）交换机。这些交换机是以默认方式创建的，但此刻并不需要用到它们。
>
> #### The default exchange
> #### 默认交换机
>
> In previous parts of the tutorial we knew nothing about exchanges, but still were able to send messages to queues. That was possible because we were using a default exchange, which we identify by the empty string ("").
>
> 教程的上一部分中，我们对交换机还一无所知，但是依然能将消息发送给队列。原因是我们使用了用空字符串(`""`)来标示的默认交换机。
>
> Recall how we published a message before:
>
> 回想一下之前我们是如何来发布消息的：
>
> 
>
> ```csharp
>  var message = GetMessage(args);
>  var body = Encoding.UTF8.GetBytes(message);
>  channel.BasicPublish(exchange: "",
>                       routingKey: "hello",
>                       basicProperties: null,
>                       body: body);
> ```
>
> 
>
> The first parameter is the name of the exchange. The empty string denotes the default or *nameless* exchange: messages are routed to the queue with the name specified by routingKey, if it exists.
>
> 第一个参数就是交换机的名字。空字符串用来表示默认或者*无名*交换机：如果队列存在的话，消息会依据路由键（`routingKey`）所指定的名称路由到队列中。

Now, we can publish to our named exchange instead:

现在我们可以对命名过的交换机执行发布操作了：

```csharp
var message = GetMessage(args);
var body = Encoding.UTF8.GetBytes(message);
channel.BasicPublish(exchange: "logs",
                     routingKey: "",
                     basicProperties: null,
                     body: body);
```

## Temporary queues
## 临时队列

As you may remember previously we were using queues that had specific names (remember hello and task_queue?). Being able to name a queue was crucial for us -- we needed to point the workers to the same queue. Giving a queue a name is important when you want to share the queue between producers and consumers.

你可能还记得我们上次使用的是命名过的队列（还记得`hello`和`task_queue`吗？）。可以对队列进行命名对我们来说是至关重要的——我们需要将工作者指向同一个队列。当你想在多个生产者和消费者之间共享一个队列时，给队列起个名字是很重要的。

But that's not the case for our logger. We want to hear about all log messages, not just a subset of them. We're also interested only in currently flowing messages not in the old ones. To solve that we need two things.

但是我们的日志系统不需要如此。我们希望了解所有的消息，而不是其中的一个子集。而且我们只对当前正在流动的消息感兴趣，而不是那些老的消息。所以我们需要做两件事情来解决这个问题。

Firstly, whenever we connect to Rabbit we need a fresh, empty queue. To do this we could create a queue with a random name, or, even better - let the server choose a random queue name for us.

首先，如论我们何时连接到Rabbit，我们需要的是一个新鲜的空队列。想要做到这点，我们可以创建一个随机命名的队列，或者更简单一点——让服务器为我们选择一个随机队列名称。

Secondly, once we disconnect the consumer the queue should be automatically deleted.

其次，一旦消费者断开连接，队列需要被自动删除。

In the .NET client, when we supply no parameters to QueueDeclare() we create a non-durable, exclusive, autodelete queue with a generated name:

在.NET客户端中，当我们不给`QueueDeclare()`提供参数的情况下，就可以创建一个非持久化、独享的、可自动删除的拥有生成名称的队列。

```csharp
var queueName = channel.QueueDeclare().QueueName;
```

You can learn more about the exclusive flag and other queue properties in the [guide on queues](https://www.rabbitmq.com/queues.html).

你可以在[guide on queues](https://www.rabbitmq.com/queues.html)中学习到更多关于独享（`exclusive`）标识以及其他队列属性的相关信息。

At that point queueName contains a random queue name. For example it may look like amq.gen-JzTY20BRgKO-HjmUJj0wLg.

此时，`queueName`包含的是一个随机的队列名称。看起来可能会类似于`amq.gen-JzTY20BRgKO-HjmUJj0wLg`这样。

## Bindings

## 绑定

![img](https://www.rabbitmq.com/img/tutorials/bindings.png)

We've already created a fanout exchange and a queue. Now we need to tell the exchange to send messages to our queue. That relationship between exchange and a queue is called a *binding*.

我们已经创建了一个扇形交换机和一个队列。现在我们需要通知交换机将消息发送给我们的队列。交换机和队列之间的这种关系称为*绑定(`binding`)*。

```csharp
channel.QueueBind(queue: queueName,
                  exchange: "logs",
                  routingKey: "");
```

From now on the logs exchange will append messages to our queue.

现在开始，`logs`交换机会将消息追加到我们的队列当中。

> #### Listing bindings
> #### 绑定的监听
>
> You can list existing bindings using, you guessed it,
>
> 你可以通过以下命令列出所有正在使用的绑定，
>
> ```bash
> rabbitmqctl list_bindings
> ```

## Putting it all together

## 组合到一起

![img](https://www.rabbitmq.com/img/tutorials/python-three-overall.png)

The producer program, which emits log messages, doesn't look much different from the previous tutorial. The most important change is that we now want to publish messages to our logs exchange instead of the nameless one. We need to supply a routingKey when sending, but its value is ignored for fanout exchanges. Here goes the code for EmitLog.cs file:

用来发送日志消息的生产者程序看起来跟上个教程中的没多大区别。最重大的改变是，现在我们希望将消息发布到`logs`交换机而不是未命名的那个。发送的时候我们需要提供一个`routingKey，`但是它的值会被扇形交换机忽略掉。下边是`EmitLog.cs`文件：

```csharp
using System;
using RabbitMQ.Client;
using System.Text;

class EmitLog
{
    public static void Main(string[] args)
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.ExchangeDeclare(exchange: "logs", type: "fanout");

            var message = GetMessage(args);
            var body = Encoding.UTF8.GetBytes(message);
            channel.BasicPublish(exchange: "logs",
                                 routingKey: "",
                                 basicProperties: null,
                                 body: body);
            Console.WriteLine(" [x] Sent {0}", message);
        }

        Console.WriteLine(" Press [enter] to exit.");
        Console.ReadLine();
    }

    private static string GetMessage(string[] args)
    {
        return ((args.Length > 0)
               ? string.Join(" ", args)
               : "info: Hello World!");
    }
}
```

[(EmitLog.cs source)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/EmitLog/EmitLog.cs)

[(EmitLog.cs 源文件)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/EmitLog/EmitLog.cs)

As you see, after establishing the connection we declared the exchange. This step is necessary as publishing to a non-existing exchange is forbidden.

如你所见，建立连接之后，我们对交换机进行了声明。这一步是必需的，因为不允许发布消息到一个不存在的交换机。

The messages will be lost if no queue is bound to the exchange yet, but that's okay for us; if no consumer is listening yet we can safely discard the message.

如果尚未有队列绑定到交换机，消息会丢失掉，但是对我们来说无所谓；如果还没有消费者进行监听，我们可以安全的将消息丢弃掉。

The code for ReceiveLogs.cs:

`ReceiveLogs.cs`的代码：

```csharp
using System;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

class ReceiveLogs
{
    public static void Main()
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.ExchangeDeclare(exchange: "logs", type: "fanout");

            var queueName = channel.QueueDeclare().QueueName;
            channel.QueueBind(queue: queueName,
                              exchange: "logs",
                              routingKey: "");

            Console.WriteLine(" [*] Waiting for logs.");

            var consumer = new EventingBasicConsumer(channel);
            consumer.Received += (model, ea) =>
            {
                var body = ea.Body;
                var message = Encoding.UTF8.GetString(body);
                Console.WriteLine(" [x] {0}", message);
            };
            channel.BasicConsume(queue: queueName,
                                 autoAck: true,
                                 consumer: consumer);

            Console.WriteLine(" Press [enter] to exit.");
            Console.ReadLine();
        }
    }
}
```

[(ReceiveLogs.cs source)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/ReceiveLogs/ReceiveLogs.cs)

[(ReceiveLogs.cs 源代码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/ReceiveLogs/ReceiveLogs.cs)

Follow the setup instructions from [tutorial one](https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html) to generate the EmitLogs and ReceiveLogsprojects.

根据 [教程一](https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html) 所介绍的步骤生成 `EmitLogs` and `ReceiveLogs`项目。

If you want to save logs to a file, just open a console and type:

如果你想要将日志保存到一个文件中，只需要在控制台中输入：

```bash
cd ReceiveLogs
dotnet run > logs_from_rabbit.log
```

If you wish to see the logs on your screen, spawn a new terminal and run:

如果你希望在屏幕上看到日志记录，打开一个新的终端并运行：

```bash
cd ReceiveLogs
dotnet run
```

And of course, to emit logs type:

当然，还需要通过以下方式发送日志：

```bash
cd EmitLog
dotnet run
```

Using rabbitmqctl list_bindings you can verify that the code actually creates bindings and queues as we want. With two ReceiveLogs.cs programs running you should see something like:

使用`rabbitmqctl list_bindings`命令可以验证绑定和队列是否按照我们期望的方式正确运行。当有两个`ReceiveLogs.cs`程序运行的时候，你应该可以看到类似于这样的信息：



```bash
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```

The interpretation of the result is straightforward: data from exchange logs goes to two queues with server-assigned names. And that's exactly what we intended.

简单地对结果进行下解释：跟我们期待的一样，数据从`logs`交换机传输到两个由服务器命名的队列当中。

To find out how to listen for a subset of messages, let's move on to [tutorial 4](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html)

接下来，我们可以移步[教程4](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html)来了解如何监听消息的子集。