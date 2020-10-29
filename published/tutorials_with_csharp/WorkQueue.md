>原文：[Work Queues](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html)  
>翻译：[mr-ping](http://rabbitmq.mr-ping.com)  
![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

> ### 前置条件

> 本教程假设RabbitMQ已经[安装](http://www.rabbitmq.com/download.html)在你本机的 (5672)端口。如果你使用了不同的主机、端口或者凭证，连接设置就需要作出一些对应的调整。

> ### 如何获得帮助

> 如果你在使用本教程的过程中遇到了麻烦，你可以通过邮件列表来[联系我们](https://groups.google.com/forum/#!forum/rabbitmq-users)。
>
> 

## 工作队列

### (使用 .NET 客户端)


![img](http://www.rabbitmq.com/img/tutorials/python-two.png)

在[第一个教程](http://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html)中，我们写了一个用于向命名过的队列发送消息并且从其中进行接收的程序。本教程中，我们会创建一个用于在多个工作者（worker）当中分发耗时任务的*工作队列（work queue）*。

工作队列 (亦称 *任务队列*) 的主要目的在于避免资源密集型任务被立即执行，执行者需要一直等到它完成为止。相反，我们会安排任务稍后完成。我们将*任务*封装成一条消息，并将其发送给队列。运行于后台的工作者程序会丢弃掉这条消息（译者注：首先从工作队列中接收到消息），并最终将消息所描述的任务执行完。当你有多个工作者的情况下，任务会在多个工作者中共享。

由于web应用无法在一个短暂的HTTP请求过程中执行比较复杂的任务，所以这种概念在web应用当中特别有用。

## 准备

上个教程中，我们发送了一条"Hellow World!"消息。这次，我们会发送一个用于表示复杂任务的字符串。由于我们当下没有一个类似于图片缩放，渲染pdf文件这种真实的任务需要执行，因此我们会使用`Thread.Sleep()`函数来伪造繁忙的状态（要使用有关线程的接口，需要在文件顶部添加`using System.Threading;`的引用）。我们会在字符串中使用英文句号来表示任务的复杂程度。每一个点代表一秒钟的工作者执行的耗时。举个例子，如果一个伪造的任务用`Hello...`来表示，那就意味着它会耗时三秒钟。

我们会稍微改造下上个教程中的*Send*程序，以便于可以通过命令行发送任意一条消息。这个程序会用来为我们的工作队列规划任务，让我们称它为`NewTask`：

跟 [教程一](http://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html) 一样，我们需要生成两个项目：

```powershell
dotnet new console --name NewTask
mv NewTask/Program.cs NewTask/NewTask.cs
dotnet new console --name Worker
mv Worker/Program.cs Worker/Worker.cs
cd NewTask
dotnet add package RabbitMQ.Client
dotnet restore
cd ../Worker
dotnet add package RabbitMQ.Client
dotnet restore
var message = GetMessage(args);
var body = Encoding.UTF8.GetBytes(message);

var properties = channel.CreateBasicProperties();
properties.Persistent = true;

channel.BasicPublish(exchange: "",
                     routingKey: "task_queue",
                     basicProperties: properties,
                     body: body);
```

为从命令行参数获取消息内容提供一些帮助：

```csharp
private static string GetMessage(string[] args)
{
    return ((args.Length > 0) ? string.Join(" ", args) : "Hello World!");
}
```

上一版 *Receive.cs* 脚本也需要做一些修改：它需要为消息体中的每一个英文句号伪造一秒钟的工作耗时，所以我们将它拷贝到`worker`项目中，并做如下修改：

```csharp
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (model, ea) =>
{
    var body = ea.Body;
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine(" [x] Received {0}", message);

    int dots = message.Split('.').Length - 1;
    Thread.Sleep(dots * 1000);

    Console.WriteLine(" [x] Done");
};
channel.BasicConsume(queue: "task_queue", autoAck: true, consumer: consumer);
```

伪造任务来模拟执行时间：

```csharp
int dots = message.Split('.').Length - 1;
Thread.Sleep(dots * 1000);
```

## 循环调度

使用任务队列所带来的一个高级特性是可以用简单的方式让工作具有并行执行的能力。如果我们建立的是一个阻塞的任务，那只需要通过添加更多的工作者就可以简便的进行扩展。

首先，让我们同事运行两个`Worker`实例。它们都可以从队列中获取消息。下面我们看看它们是如何工作的。

你需要打开三个控制台。两个用于运行`Worker`程序。它们分别代表两个消费者，命名为C1和C2。

```bash
# shell 1
cd Worker
dotnet run
# => [*] Waiting for messages. To exit press CTRL+C
# shell 2
cd Worker
dotnet run
# => [*] Waiting for messages. To exit press CTRL+C
```

第三个控制台用来发布新任务。开启了消费者之后，你就可以发布新消息了：

```bash
# shell 3
cd NewTask
dotnet run "First message."
dotnet run "Second message.."
dotnet run "Third message..."
dotnet run "Fourth message...."
dotnet run "Fifth message....."
```

让我们看看投送给工作者的是什么内容：

```bash
# shell 1
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
# shell 2
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

默认情况下，RabbitMQ会将消息依次发送给下一个消费者。平均每个消费者会获得同样数量的消息。这种消息分发方法被称为循环法（round-robin）。你可以发布三条以上的消息来尝试一下。

## 消息确认

执行一个任务可能会耗时好几秒钟。你也许会好奇如果一个消费者在执行一个耗时任务时只完成了部分工作就挂掉的情况下会发生什么。在我们当前代码下，一旦RabbitMQ将消息投送给消费者后，它会立即将消息标示为删除状态。这个案例中，如果你将工作者杀掉的话，我们会丢失它正在处理的消息。如果有其他已经调度给这个工作者的消息没有完成，也会一起丢失。

但是我们不想丢失任何任务。如果一个工作者挂掉了，我们希望任务会投送给其他的工作者。

为了确保消息永不丢失，RabbitMQ支持 [消息 *确认*](http://www.rabbitmq.com/confirms.html)。消费者回送一个确认信号——ack(nowledgement)给RabbitMQ，告诉它一条指定的消息已经接收到并且处理完毕，可以选择将消息删除掉了。

如果一个消费者在没有回送确认信号的情况下挂掉了（消费者的信道关闭，连接关闭或者TCP连接已经丢失），RabbitMQ会理解为此条消息没有被处理完成，并且重新将其放入队列。如果恰时有其他消费者在线，这条消息会立即被投送给其他的消费者。通过这种方式，你可以确定即使有工作者由于事故挂掉，也不会发生消息丢失的情况。

RabbitMQ不会有任何消息超时的机制，消费者挂掉之后RabbitMQ才会将此消息投送给其他消费者。所以即使消息处理需要话费超长的是时间也没有问题。

[手动进行消息确认](http://www.rabbitmq.com/confirms.html) 默认为开启状态。上个例子中，我们明确地通过将autoAck ("自动确认模式")设置为true将其关闭掉了。这次我们移除掉这个标志，一旦任务完成，手动从工作者当中发送合适的确认标志。

```csharp
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (model, ea) =>
{
    var body = ea.Body;
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine(" [x] Received {0}", message);

    int dots = message.Split('.').Length - 1;
    Thread.Sleep(dots * 1000);

    Console.WriteLine(" [x] Done");

    channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
};
channel.BasicConsume(queue: "task_queue", autoAck: false, consumer: consumer);
```

使用本代码，即使你在它运行时使用 CTRL+C杀掉工作者，也不会有任何东西丢失。稍后，挂掉的工作者当中未进行确认的消息会被重新投送。

确认信号必须在收到投送的同一个信道上发送。尝试在不同的信道上发送确认信号会引发信道级别的协议异常。 [确认行为的文档指南](http://www.rabbitmq.com/confirms.html) 里有更多介绍。

> #### 忘记进行确认
>
> 忘记使用`BasicAck`是一个常见的错误。虽然是个简单的错误，但是后果严重。消息会在客户端退出后重新投送（就像是随机进行的重新投送），但是由于RabbitMQ无法释放任何未经确认的消息，内存占用会越来越严重。
>
> 想要对这种错误进行调试，你可以使用`rabbitmqctl`将“未经确认的消息”（`messages_unacknowledged`）字段打印出来
>
> 
>
> ```bash
> sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
> ```
>
> 
>
> On Windows, drop the sudo:
> 
> 在Windows中，不需要sudo:
>
> ```bash
> rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
> ```

## 消息持久化

我们已经学过了如何让任务在消费者即使挂掉的情况也不会丢失。但我们的任务仍有可能在RabbitMQ服务器停机的时候丢失掉。

当RabbitMQ退出或崩溃的时候会忘记掉所有的队列，除非你告诉他不要这么做。如果想要确保消息不会丢失，我们需要做两件事，将队列和消息都标示成持久化。

首先，我们需要确保RabbitMQ永远不会将队列丢失。为了达到此目的，我们需要使用*durable*来将其持久化。

```csharp
channel.QueueDeclare(queue: "hello",
                     durable: true,
                     exclusive: false,
                     autoDelete: false,
                     arguments: null);
```

虽然这个命令本身是正确的，但是它在我们当前的配置中并不起作用。原因是我们已经定义了一个名为`hello`的非持久化队列。RabbitMQ不允许用不同的参数去重新定义一个已经存在的队列，如果有程序尝试这样做的话，会收到一个错误的返回值。 但是有一个快捷的解决方案——我们可以定义一个不重名的队列，例如`task_queue`:


```csharp
channel.QueueDeclare(queue: "task_queue",
                     durable: true,
                     exclusive: false,
                     autoDelete: false,
                     arguments: null);
```

此`QueueDeclare`的改变需要应用到生产者和消费者两份代码当中。

当下，我们可以确认即使RabbitMQ重启，我们的`task_queue`队列也不会丢失。接下来，我们需要将`IBasicProperties.SetPersistent`设置为`true`，用来将我们的消息标示成持久化的。


```csharp
var properties = channel.CreateBasicProperties();
properties.Persistent = true;
```

> #### 消息持久化的注释
>
> 将消息标示为持久化并不能完全保证消息不会丢失。尽管它会告诉RabbitMQ将消息存储到硬盘上，但是在RabbitMQ接收到消息并将其进行存储两个行为之间仍旧会有一个窗口期。同样的，RabbitMQ也不会对每一条消息执行`fsync(2)`，所以消息获取只是存到了缓存之中，而不是硬盘上。虽然持久化的保证不强，但是应对我们简单的任务队列已经足够了。如果你需要更强的保证，可以使用[publisher confirms](https://www.rabbitmq.com/confirms.html).

## 公平调度

你可能注意到了，调度依照我们希望的方式运行。例如在有两个工作者的情况下，当所有的奇数任务都很繁重而所有的偶数任务都很轻松的时候，其中一个工作者会一直处于忙碌之中而另一个几乎无事可做。RabbitMQ并不会对此有任何察觉，仍旧会平均分配消息。

这种情况发生的原因是由于当有消息进入队列时，RabbitMQ只负责将消息调度的工作，而不会检查某个消费者有多少未经确认的消息。它只是盲目的将第n个消息发送给第n个消费者而已。

![img](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)

要改变这种行为的话，我们可以在`BasicQos`方法中设置`prefetchCount = 1`。这样会告诉RabbitMQ一次不要给同一个worker提供多于一条的信息。话句话说，在一个工作者还没有处理完消息，并且返回确认标志之前，不要再给它调度新的消息。取而代之，它会将消息调度给下一个不再繁忙的工作者。

```csharp
channel.BasicQos(0, 1, false);
```

> If all the workers are busy, your queue can fill up. You will want to keep an eye on that, and maybe add more workers, or have some other strategy.

## 整合到一起

最终的`NewTask.cs` 类代码：

```csharp
using System;
using RabbitMQ.Client;
using System.Text;

class NewTask
{
    public static void Main(string[] args)
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.QueueDeclare(queue: "task_queue",
                                 durable: true,
                                 exclusive: false,
                                 autoDelete: false,
                                 arguments: null);

            var message = GetMessage(args);
            var body = Encoding.UTF8.GetBytes(message);

            var properties = channel.CreateBasicProperties();
            properties.Persistent = true;

            channel.BasicPublish(exchange: "",
                                 routingKey: "task_queue",
                                 basicProperties: properties,
                                 body: body);
            Console.WriteLine(" [x] Sent {0}", message);
        }

        Console.WriteLine(" Press [enter] to exit.");
        Console.ReadLine();
    }

    private static string GetMessage(string[] args)
    {
        return ((args.Length > 0) ? string.Join(" ", args) : "Hello World!");
    }
}
```

[(NewTask.cs 源代码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/NewTask/NewTask.cs)

我们的 `Worker.cs`：

```csharp
using System;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;
using System.Threading;

class Worker
{
    public static void Main()
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.QueueDeclare(queue: "task_queue",
                                 durable: true,
                                 exclusive: false,
                                 autoDelete: false,
                                 arguments: null);

            channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);

            Console.WriteLine(" [*] Waiting for messages.");

            var consumer = new EventingBasicConsumer(channel);
            consumer.Received += (model, ea) =>
            {
                var body = ea.Body;
                var message = Encoding.UTF8.GetString(body);
                Console.WriteLine(" [x] Received {0}", message);

                int dots = message.Split('.').Length - 1;
                Thread.Sleep(dots * 1000);

                Console.WriteLine(" [x] Done");

                channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
            };
            channel.BasicConsume(queue: "task_queue",
                                 autoAck: false,
                                 consumer: consumer);

            Console.WriteLine(" Press [enter] to exit.");
            Console.ReadLine();
        }
    }
}
```

[(Worker.cs 源代码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Worker/Worker.cs)

使用消息确认和`BasicQos`，你可以设置一个工作队列。持久化选项可以使得任务即使在RabbitMQ重启后也不会丢失。

有关`IModel`方法和`IBasicProperties`的更多信息，请浏览[RabbitMQ .NET client API reference online](http://www.rabbitmq.com/releases/rabbitmq-dotnet-client/v3.6.10/rabbitmq-dotnet-client-3.6.10-client-htmldoc/html/index.html).

现在我们可以异步[教程 3](http://www.rabbitmq.com/tutorials/tutorial-three-dotnet.html)，学习将消息投送给多个消费者。
