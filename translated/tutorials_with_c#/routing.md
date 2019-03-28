> ### 前置条件

> 本教程假设RabbitMQ已经[安装](http://www.rabbitmq.com/download.html)在你本机的 (5672)端口。如果你使用了不同的主机、端口或者凭证，连接设置就需要作出一些对应的调整。

> ### 如何获得帮助

> 如果你在使用本教程的过程中遇到了麻烦，你可以通过邮件列表来[联系我们](https://groups.google.com/forum/#!forum/rabbitmq-users)。

## Routing

## 路由

### (using the .NET client)

### （使用.NET客户端）

In the [previous tutorial](https://www.rabbitmq.com/tutorials/tutorial-three-dotnet.html) we built a simple logging system. We were able to broadcast log messages to many receivers.

上个教程中，我们建立了一个简单的日志系统。我们可以将日志消息广播给多个接收者。

In this tutorial we're going to add a feature to it - we're going to make it possible to subscribe only to a subset of the messages. For example, we will be able to direct only critical error messages to the log file (to save disk space), while still being able to print all of the log messages on the console.

这个教程中我们会添加一个新功能——让它可以从日志消息中只订阅一个子集。例如，我们只将关键的错误消息定向到日志文件中（从而节省磁盘空间），同时仍旧可以将所有的日志消息打印到控制台上。

## Bindings

## 绑定

In previous examples we were already creating bindings. You may recall code like:

上个例子中，我们已经创建了绑定。代码如下：

```csharp
channel.QueueBind(queue: queueName,
                  exchange: "logs",
                  routingKey: "");
```

A binding is a relationship between an exchange and a queue. This can be simply read as: the queue is interested in messages from this exchange.

一个绑定就是交换机和队列之间的一个关系。可以解读为：目标队列对此交换机的消息感兴趣。

Bindings can take an extra routingKey parameter. To avoid the confusion with a BasicPublishparameter we're going to call it a binding key. This is how we could create a binding with a key:

绑定可以使用一个格外的`routingKey`参数。为了避免跟`BasicPublish`参数混淆，我们称其为绑定键(`binding key`)。以下是如何创建一个绑定键：

```csharp
channel.QueueBind(queue: queueName,
                  exchange: "direct_logs",
                  routingKey: "black");
```

The meaning of a binding key depends on the exchange type. The fanout exchanges, which we used previously, simply ignored its value.

绑定键的实际意义依赖于交换机的类型。对于我们之前使用的扇形交换机来说，会简单的将其值忽略掉。

## Direct exchange

## 直连型交换机

Our logging system from the previous tutorial broadcasts all messages to all consumers. We want to extend that to allow filtering messages based on their severity. For example we may want the script which is writing log messages to the disk to only receive critical errors, and not waste disk space on warning or info log messages.

上个教程中，我们的日志系统将所有的消息广播给所有的消费者。我们打算对其进行扩展以根据它们的严重性来进行过滤。例如，我们想要将日志写到磁盘上的脚本只接收关键性的错误，而不在警告信息和普通日志消息上浪费磁盘空间。

We were using a fanout exchange, which doesn't give us much flexibility - it's only capable of mindless broadcasting.

我们之前使用的扇形交换机不能提供足够的灵活性——它只能进行无意识的广播。

We will use a direct exchange instead. The routing algorithm behind a direct exchange is simple - a message goes to the queues whose binding key exactly matches the routing keyof the message.

下面我们使用直连型交换机进行替代。直连型交换机背后的路由算法很简单——消息会传送给绑定键与消息的路由键完全匹配的那个队列。

To illustrate that, consider the following setup:

为了说明这点，可以考虑如下设置：

![img](https://www.rabbitmq.com/img/tutorials/direct-exchange.png)

In this setup, we can see the direct exchange X with two queues bound to it. The first queue is bound with binding key orange, and the second has two bindings, one with binding key black and the other one with green.

这种配置下，我们可以看到有两个队列绑定到了直连交换机`X`上。第一个队列用的是橘色（`orange`）绑定键，第二个有两个绑定键，其中一个绑定键是黑色（`black`），另一个绑定键是绿色（`green`）。

In such a setup a message published to the exchange with a routing key orange will be routed to queue Q1. Messages with a routing key of black or green will go to Q2. All other messages will be discarded.

在此设置中，发布到交换机的带有橘色（`orange`）路由键的消息会被路由给队列`Q1`。带有黑色（`black`）或绿色（`green`）路由键的消息会被路由给`Q2`。其他的消息则会被丢弃。

## Multiple bindings

## 多个绑定

![img](https://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

使用相同的绑定键来绑定多个队列是完全合法的。在我们的例子中，我们可以使用黑色（`black`）绑定键来绑定`X`和`Q1`。那种情况下，直连型交换机的行为就会跟扇形交换机类似，会将消息广播给所有匹配的队列。一个拥有黑色(`black`)路由键的消息会被头送给`Q1`和`Q2`两个队列。

It is perfectly legal to bind multiple queues with the same binding key. In our example we could add a binding between X and Q1 with binding key black. In that case, the direct exchange will behave like fanout and will broadcast the message to all the matching queues. A message with routing key black will be delivered to both Q1 and Q2.

使用相同的绑定键来绑定多个队列是完全合法的。在我们的例子中，我们可以使用黑色（`black`）绑定键来绑定`X`和`Q1`。那种情况下，直连型交换机（`direct`）的行为就会跟扇形交换机（`fanout`）类似，会将消息广播给所有匹配的队列。一个拥有黑色(`black`)路由键的消息会被头送给`Q1`和`Q2`两个队列。

## Emitting logs

## 发送日志

We'll use this model for our logging system. Instead of fanout we'll send messages to a direct exchange. We will supply the log severity as a routing key. That way the receiving script will be able to select the severity it wants to receive. Let's focus on emitting logs first.

我们将会在我们日志系统中采用这种模式，将消息发送给直连交换机来替代扇形交换机。我们会提供日志的严重等级来作为路由键的值。通过这种方式脚本就可以选择其需要的严重等级来进行接收。首先让我们将关注点放到发送日志上：

As always, we need to create an exchange first:

像往常一样，首先我们需要创建一个交换机：

```csharp
channel.ExchangeDeclare(exchange: "direct_logs", type: "direct");
```

And we're ready to send a message:

然后，做好发送消息的准备：

```csharp
var body = Encoding.UTF8.GetBytes(message);
channel.BasicPublish(exchange: "direct_logs",
                     routingKey: severity,
                     basicProperties: null,
                     body: body);
```

To simplify things we will assume that 'severity' can be one of 'info', 'warning', 'error'.

为了保持简洁，我们假设严重等级只可以是'info', 'warning', 'error'其中一种。

## Subscribing

## 订阅

Receiving messages will work just like in the previous tutorial, with one exception - we're going to create a new binding for each severity we're interested in.

除了我们会为每个我们感兴趣的严重等级创建一个新的绑定键之外，接收消息的工作方式跟前一个教程中几乎一样。

```csharp
var queueName = channel.QueueDeclare().QueueName;

foreach(var severity in args)
{
    channel.QueueBind(queue: queueName,
                      exchange: "direct_logs",
                      routingKey: severity);
}
```

## Putting it all together
## 整合到一起

![img](https://www.rabbitmq.com/img/tutorials/python-four.png)

The code for `EmitLogDirect.cs` class:类的代码：
`EmitLogDirect.cs`类的代码：


```csharp
using System;
using System.Linq;
using RabbitMQ.Client;
using System.Text;

class EmitLogDirect
{
    public static void Main(string[] args)
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.ExchangeDeclare(exchange: "direct_logs",
                                    type: "direct");

            var severity = (args.Length > 0) ? args[0] : "info";
            var message = (args.Length > 1)
                          ? string.Join(" ", args.Skip( 1 ).ToArray())
                          : "Hello World!";
            var body = Encoding.UTF8.GetBytes(message);
            channel.BasicPublish(exchange: "direct_logs",
                                 routingKey: severity,
                                 basicProperties: null,
                                 body: body);
            Console.WriteLine(" [x] Sent '{0}':'{1}'", severity, message);
        }

        Console.WriteLine(" Press [enter] to exit.");
        Console.ReadLine();
    }
}
```

The code for ReceiveLogsDirect.cs:

`ReceiveLogsDirect.cs`的代码：

```csharp
using System;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

class ReceiveLogsDirect
{
    public static void Main(string[] args)
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.ExchangeDeclare(exchange: "direct_logs",
                                    type: "direct");
            var queueName = channel.QueueDeclare().QueueName;

            if(args.Length < 1)
            {
                Console.Error.WriteLine("Usage: {0} [info] [warning] [error]",
                                        Environment.GetCommandLineArgs()[0]);
                Console.WriteLine(" Press [enter] to exit.");
                Console.ReadLine();
                Environment.ExitCode = 1;
                return;
            }

            foreach(var severity in args)
            {
                channel.QueueBind(queue: queueName,
                                  exchange: "direct_logs",
                                  routingKey: severity);
            }

            Console.WriteLine(" [*] Waiting for messages.");

            var consumer = new EventingBasicConsumer(channel);
            consumer.Received += (model, ea) =>
            {
                var body = ea.Body;
                var message = Encoding.UTF8.GetString(body);
                var routingKey = ea.RoutingKey;
                Console.WriteLine(" [x] Received '{0}':'{1}'",
                                  routingKey, message);
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

Create projects as usual (see [tutorial one](https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html) for advice).

跟往常一样创建项目（参见  [教程一](https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html) ）

If you want to save only 'warning' and 'error' (and not 'info') log messages to a file, just open a console and type:

如果你只希望将'warning' 和 'error' (不包括 'info') 的日志信息保存到文件中，只需要打开一个控制台，输入：

```bash
cd ReceiveLogsDirect
dotnet run warning error > logs_from_rabbit.log
```

If you'd like to see all the log messages on your screen, open a new terminal and do:

如果你希望将所有的日志信息显示在屏幕上，新开一个终端，做如下操作：

```bash
cd ReceiveLogsDirect
dotnet run info warning error
# => [*] Waiting for logs. To exit press CTRL+C
```

And, for example, to emit an error log message just type:

例如，如果你想发送一条`error`的日志信息，只需要输入：

```bash
cd EmitLogDirect
dotnet run error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
```

(Full source code for [(EmitLogDirect.cs source)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/EmitLogDirect/EmitLogDirect.cs) and [(ReceiveLogsDirect.cs source)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/ReceiveLogsDirect/ReceiveLogsDirect.cs))

(完整的 [(EmitLogDirect.cs 源代码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/EmitLogDirect/EmitLogDirect.cs) 和 [(ReceiveLogsDirect.cs 源代码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/ReceiveLogsDirect/ReceiveLogsDirect.cs))

Move on to [tutorial 5](https://www.rabbitmq.com/tutorials/tutorial-five-dotnet.html) to find out how to listen for messages based on a pattern.

想要了解如何基于一种模式来监听消息，可以移步至 [教程 5](https://www.rabbitmq.com/tutorials/tutorial-five-dotnet.html) 。