>原文：[Routing](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html)  
>翻译：[mr-ping](http://rabbitmq.mr-ping.com)  
![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

> ### 前置条件

> 本教程假设RabbitMQ已经[安装](http://www.rabbitmq.com/download.html)在你本机的 (5672)端口。如果你使用了不同的主机、端口或者凭证，连接设置就需要作出一些对应的调整。

> ### 如何获得帮助

> 如果你在使用本教程的过程中遇到了麻烦，你可以通过邮件列表来[联系我们](https://groups.google.com/forum/#!forum/rabbitmq-users)。

## 路由

### （使用.NET客户端）

上个教程中，我们建立了一个简单的日志系统。我们可以将日志消息广播给多个接收者。

这个教程中我们会添加一个新功能——让它可以从日志消息中只订阅一个子集。例如，我们只将关键的错误消息定向到日志文件中（从而节省磁盘空间），同时仍旧可以将所有的日志消息打印到控制台上。

## 绑定

上个例子中，我们已经创建了绑定。代码如下：

```csharp
channel.QueueBind(queue: queueName,
                  exchange: "logs",
                  routingKey: "");
```

一个绑定就是交换机和队列之间的一个关系。可以解读为：目标队列对此交换机的消息感兴趣。

绑定可以使用一个格外的`routingKey`参数。为了避免跟`BasicPublish`参数混淆，我们称其为绑定键(`binding key`)。以下是如何创建一个绑定键：

```csharp
channel.QueueBind(queue: queueName,
                  exchange: "direct_logs",
                  routingKey: "black");
```

绑定键的实际意义依赖于交换机的类型。对于我们之前使用的扇形交换机来说，会简单的将其值忽略掉。

## 直连型交换机

上个教程中，我们的日志系统将所有的消息广播给所有的消费者。我们打算对其进行扩展以根据它们的严重性来进行过滤。例如，我们想要将日志写到磁盘上的脚本只接收关键性的错误，而不在警告信息和普通日志消息上浪费磁盘空间。

我们之前使用的扇形交换机不能提供足够的灵活性——它只能进行无意识的广播。

下面我们使用直连型交换机进行替代。直连型交换机背后的路由算法很简单——消息会传送给绑定键与消息的路由键完全匹配的那个队列。

为了说明这点，可以考虑如下设置：

![img](https://www.rabbitmq.com/img/tutorials/direct-exchange.png)

这种配置下，我们可以看到有两个队列绑定到了直连交换机`X`上。第一个队列用的是橘色（`orange`）绑定键，第二个有两个绑定键，其中一个绑定键是黑色（`black`），另一个绑定键是绿色（`green`）。

在此设置中，发布到交换机的带有橘色（`orange`）路由键的消息会被路由给队列`Q1`。带有黑色（`black`）或绿色（`green`）路由键的消息会被路由给`Q2`。其他的消息则会被丢弃。

## 多个绑定

![img](https://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

使用相同的绑定键来绑定多个队列是完全合法的。在我们的例子中，我们可以使用黑色（`black`）绑定键来绑定`X`和`Q1`。那种情况下，直连型交换机的行为就会跟扇形交换机类似，会将消息广播给所有匹配的队列。一个拥有黑色(`black`)路由键的消息会被头送给`Q1`和`Q2`两个队列。

使用相同的绑定键来绑定多个队列是完全合法的。在我们的例子中，我们可以使用黑色（`black`）绑定键来绑定`X`和`Q1`。那种情况下，直连型交换机（`direct`）的行为就会跟扇形交换机（`fanout`）类似，会将消息广播给所有匹配的队列。一个拥有黑色(`black`)路由键的消息会被头送给`Q1`和`Q2`两个队列。

## 发送日志

我们将会在我们日志系统中采用这种模式，将消息发送给直连交换机来替代扇形交换机。我们会提供日志的严重等级来作为路由键的值。通过这种方式脚本就可以选择其需要的严重等级来进行接收。首先让我们将关注点放到发送日志上：

像往常一样，首先我们需要创建一个交换机：

```csharp
channel.ExchangeDeclare(exchange: "direct_logs", type: "direct");
```

然后，做好发送消息的准备：

```csharp
var body = Encoding.UTF8.GetBytes(message);
channel.BasicPublish(exchange: "direct_logs",
                     routingKey: severity,
                     basicProperties: null,
                     body: body);
```

为了保持简洁，我们假设严重等级只可以是'info', 'warning', 'error'其中一种。

## 订阅

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

## 整合到一起

![img](https://www.rabbitmq.com/img/tutorials/python-four.png)

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

跟往常一样创建项目（参见  [教程一](https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html) ）

如果你只希望将'warning' 和 'error' (不包括 'info') 的日志信息保存到文件中，只需要打开一个控制台，输入：

```bash
cd ReceiveLogsDirect
dotnet run warning error > logs_from_rabbit.log
```

如果你希望将所有的日志信息显示在屏幕上，新开一个终端，做如下操作：

```bash
cd ReceiveLogsDirect
dotnet run info warning error
# => [*] Waiting for logs. To exit press CTRL+C
```

例如，如果你想发送一条`error`的日志信息，只需要输入：

```bash
cd EmitLogDirect
dotnet run error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
```

(完整的 [(EmitLogDirect.cs 源代码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/EmitLogDirect/EmitLogDirect.cs) 和 [(ReceiveLogsDirect.cs 源代码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/ReceiveLogsDirect/ReceiveLogsDirect.cs))

想要了解如何基于一种模式来监听消息，可以移步至 [教程 5](https://www.rabbitmq.com/tutorials/tutorial-five-dotnet.html) 。
