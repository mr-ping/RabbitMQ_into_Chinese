>原文：[Remote procedure call RPC](https://www.rabbitmq.com/tutorials/tutorial-six-dotnet.html)  
>翻译：[mr-ping](http://rabbitmq.mr-ping.com)  
![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

> ### 前置条件

> 本教程假设RabbitMQ已经[安装](http://www.rabbitmq.com/download.html)在你本机的 (5672)端口。如果你使用了不同的主机、端口或者凭证，连接设置就需要作出一些对应的调整。

> ### 如何获得帮助

> 如果你在使用本教程的过程中遇到了麻烦，你可以通过邮件列表来[联系我们](https://groups.google.com/forum/#!forum/rabbitmq-users)。

## 远程过程调用(RPC)

### (使用 .NET 客户端)


在[第二个教程](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html)中，我们学习了如何使用*工作队列* 在多个工作者之间分配耗时任务。

不过如果我们需要在一个远程电脑上运行函数并且等待结果的时候会怎样呢。这又是另一个故事了。这种模式通常被称为*远程过程调用*或者*RPC*。

这个教程里，我们用RabbitMQ来建立一个RPC系统：一个客户端和一个可扩展的RPC服务器。因为没有任何耗时任务用于分发，我们会创建一个伪造的用来返回斐波那契数列的RPC服务。

### 客户端接口

为了说明如何使用一个RPC服务，我们创建一个简单的客户端类。它会暴露一个名为`Call`的方法，此方法发送一个RPC请求，然后阻塞到收到回答为止。

```csharp
var rpcClient = new RPCClient();

Console.WriteLine(" [x] Requesting fib(30)");
var response = rpcClient.Call("30");
Console.WriteLine(" [.] Got '{0}'", response);

rpcClient.Close();
```

> #### 有关RPC的说明
>
> 虽然RPC在运算中是个非常常见的模式，但是也常常被批评。问题出在程序员察觉不到函数调用是发生在本地还是发生在一个很慢的RPC当中。这种困惑导致了系统的不可预测性并且为调试增加了不必要的复杂度。跟简单的软件相比，RPC会导致不可维护的面条式代码（译者注：[面条式代码](https://bkso.baidu.com/item/%E9%9D%A2%E6%9D%A1%E5%BC%8F%E4%BB%A3%E7%A0%81)在软件工程中是一种典型的反面模式）。
>
> 考虑到这点，请斟酌以下建议：
>
> - 确保一眼就能看出来哪个方法是本地执行的，哪个方法是远程执行的。
> - 给你的系统写好文档。让组件之间的依赖清晰可查。
> - 处理错误用例。如果RPC宕掉的时间过长，客户端该如何反应。
>
> 当有疑虑的时候，请避免实用RPC。有可能的话，应该使用异步管道来替代RPC式的阻塞，从而将结果异步地推送给下一个计算阶段。

### 回调队列

通常情况下，通过RabbitMQ来使用RPC非常简单。一个客户端发送请求消息，一个服务器返回消息来进行响应。为了能够收到响应，我们需要发送一个带有回调队列地址的请求：

```csharp
var props = channel.CreateBasicProperties();
props.ReplyTo = replyQueueName;

var messageBytes = Encoding.UTF8.GetBytes(message);
channel.BasicPublish(exchange: "",
                     routingKey: "rpc_queue",
                     basicProperties: props,
                     body: messageBytes);

// ... then code to read a response message from the callback_queue ...
// ... 然后代码从回调队列中读取返回的消息 ...
```

> #### 消息属性
>
> AMQP 0-9-1 协议与定义了14个消息的属性。大部分属性很少会用到，不过以下几个例外：
>
> - `Persistent`: 标示此信息为持久的（用值为2来表示）还是暂时的（2之外的其他任何值）。具体可以去 [第二个教程](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html) 一探究竟。
> - `DeliveryMode`：那些熟悉协议的人可能会选择使用这个属性，而不是`Persistent`。他们办的是一个事情。
> - `ContentType`：用来描述编码的mime-type。例如对于经常用到的JSON编码来说，将此属性设置为application/json是一个很好的做法。
> - `ReplyTo`: 通常用来对一个回调队列进行命名。
> - `CorrelationId`: 用于将RPC响应和请求进行关联。

### 关联id

上面介绍的方法中，我们建议为每一个RPC请求创建一个回调队列。这样做效率很低，幸好我们有更好的解决办法，让我们为每个客户端创建一个单独的回调队列。

这样又有一个新问题，当我们从此队列里收到一个响应的时候并不清楚它是属于哪个请求的。这就是`CorrelationId`属性的用途所在了。我们为每一个请求将其设置为一个唯一值。稍后，当我们从回调队列里收到一条消息的时候，就可以通过这个属性来对响应和请求进行匹配。如果我们收到的消息的`CorrelationId`值是未知的，那就可以安心的把它丢弃掉，因为它并不属于我们的请求。

或许你会问，我们为什么不把回调队列里的未知消息当成错误的失败来处理，而是要把它忽略掉？这是由于服务器端存在竞争条件的可能。虽然可能性不大，但是RPC服务器是有可能在发送给我们回应之后挂掉的，可此时它并没有完成为请求发回确认（acknowledgment）的动作。一旦这种情况发生，RPC服务器会再次将那条请求处理一遍。这就是为什么我们需要在客户端里优雅的处理重复的响应，并且在理想情况下，RPC应该是幂等的。

### 总结

![img](https://www.rabbitmq.com/img/tutorials/python-six.png)

我们的RPC看起来是这样的：

- 当客户端启动时，会创建一个匿名的独占回调队列。
- 客户端发送一条带有`ReplyTo`和`CorrelationId`两个属性的消息作为一个RPC请求，`ReplyTo`用于设置回调队列，`CorrelationId`用于为每一个请求设置一个独一无二的值。
- 请求被发送到一个`rpc_queue`队列。
- RPC工作者（也称为服务器）等待从那个队列中接受请求。当一个请求出现的时候，他会执行任务并且通过`ReplyTo` 属性所提及的队列来将带有执行结果的消息发回给客户端。
- 客户端从回调队列那儿等待数据。当消息出现的时候，它会检查`CorrelationId`属性。如果属性值跟请求相匹配，就将响应返回给应用。

## 将代码整合到一起

斐波那契任务：

```csharp
private static int fib(int n)
{
    if (n == 0 || n == 1) return n;
    return fib(n - 1) + fib(n - 2);
}
```

我们定义了斐波那契函数。函数假设输入值是合法的正整数。（不要期望这个函数能作用于很大的数字，它可能是最慢的递归实现了）。

RPC服务器代码：[RPCServer.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/RPCServer/RPCServer.cs) 看起来像这样：

```csharp
using System;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

class RPCServer
{
    public static void Main()
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using (var connection = factory.CreateConnection())
        using (var channel = connection.CreateModel())
        {
            channel.QueueDeclare(queue: "rpc_queue", durable: false,
              exclusive: false, autoDelete: false, arguments: null);
            channel.BasicQos(0, 1, false);
            var consumer = new EventingBasicConsumer(channel);
            channel.BasicConsume(queue: "rpc_queue",
              autoAck: false, consumer: consumer);
            Console.WriteLine(" [x] Awaiting RPC requests");

            consumer.Received += (model, ea) =>
            {
                string response = null;

                var body = ea.Body;
                var props = ea.BasicProperties;
                var replyProps = channel.CreateBasicProperties();
                replyProps.CorrelationId = props.CorrelationId;

                try
                {
                    var message = Encoding.UTF8.GetString(body);
                    int n = int.Parse(message);
                    Console.WriteLine(" [.] fib({0})", message);
                    response = fib(n).ToString();
                }
                catch (Exception e)
                {
                    Console.WriteLine(" [.] " + e.Message);
                    response = "";
                }
                finally
                {
                    var responseBytes = Encoding.UTF8.GetBytes(response);
                    channel.BasicPublish(exchange: "", routingKey: props.ReplyTo,
                      basicProperties: replyProps, body: responseBytes);
                    channel.BasicAck(deliveryTag: ea.DeliveryTag,
                      multiple: false);
                }
            };

            Console.WriteLine(" Press [enter] to exit.");
            Console.ReadLine();
        }
    }

    /// 
    /// Assumes only valid positive integer input.
    /// Don't expect this one to work for big numbers, and it's
    /// probably the slowest recursive implementation possible.
    /// 
    /// 假设输入值只能是合法的正整数。
    /// 不要期望它能服务于很大的数字，它可能是最慢的递归实现了。
    ///
    private static int fib(int n)
    {
        if (n == 0 || n == 1)
        {
            return n;
        }

        return fib(n - 1) + fib(n - 2);
    }
}
```

服务器代码很简单：

- 跟之前一样，一开始我们建立连接、信道，并且声明队列。
- 可能我们会希望运行多个服务器进程。为了在多个服务器间均分负载，我们需要在`channel.BasicQos`中设置`prefetchCount`。
- 我们使用`BasicConsume`去访问队列。然后我们注册一个投递处理程序，我们在这个处理程序中完成工作并发回响应。

RPC客户端代码 [RPCClient.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/RPCClient/RPCClient.cs)：

```csharp
using System;
using System.Collections.Concurrent;
using System.Text;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

public class RpcClient
{
    private readonly IConnection connection;
    private readonly IModel channel;
    private readonly string replyQueueName;
    private readonly EventingBasicConsumer consumer;
    private readonly BlockingCollection<string> respQueue = new BlockingCollection<string>();
    private readonly IBasicProperties props;

public RpcClient()
{
        var factory = new ConnectionFactory() { HostName = "localhost" };

        connection = factory.CreateConnection();
        channel = connection.CreateModel();
        replyQueueName = channel.QueueDeclare().QueueName;
        consumer = new EventingBasicConsumer(channel);

        props = channel.CreateBasicProperties();
        var correlationId = Guid.NewGuid().ToString();
        props.CorrelationId = correlationId;
        props.ReplyTo = replyQueueName;

        consumer.Received += (model, ea) =>
        {
            var body = ea.Body;
            var response = Encoding.UTF8.GetString(body);
            if (ea.BasicProperties.CorrelationId == correlationId)
            {
                respQueue.Add(response);
            }
        };
    }

    public string Call(string message)
    {
        var messageBytes = Encoding.UTF8.GetBytes(message);
        channel.BasicPublish(
            exchange: "",
            routingKey: "rpc_queue",
            basicProperties: props,
            body: messageBytes);

        channel.BasicConsume(
            consumer: consumer,
            queue: replyQueueName,
            autoAck: true);

        return respQueue.Take(); ;
    }

    public void Close()
    {
        connection.Close();
    }
}

public class Rpc
{
    public static void Main()
    {
        var rpcClient = new RpcClient();

        Console.WriteLine(" [x] Requesting fib(30)");
        var response = rpcClient.Call("30");

        Console.WriteLine(" [.] Got '{0}'", response);
        rpcClient.Close();
    }
}
```

客户端代码稍显复杂：

- 我们创建一个连接和信道，并且声明一个独享的`callback`队列用于回复。
- 我们订阅`callback`队列，这样就可以接收到RPC的响应。
- 我们的`Call`方法生成实际的RPC请求。
- 这里，我们首先生成一个唯一的`CorrelationId`数字并且保存起来——整个循环都会使用这个值来获取相对应的响应。
- 接下来，我们发送带有`ReplyTo` 和 `CorrelationId`属性的请求信息。
- 此刻，我们可以坐等匹配的回应到来。
- 整个循环所做的工作很简单，就是检查每个响应消息，看它们是不是我们需要的那个。如果是的话就将响应保存起来。
- 最后我们将响应返回给用户。

生成客户端请求：

```csharp
var rpcClient = new RPCClient();

Console.WriteLine(" [x] Requesting fib(30)");
var response = rpcClient.Call("30");
Console.WriteLine(" [.] Got '{0}'", response);

rpcClient.Close();
```

现在我们可以看看  [RPCClient.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/RPCClient/RPCClient.cs) 和 [RPCServer.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/RPCServer/RPCServer.cs) 的完整的样例代码了（代码里包含了简单的异常处理）。

像往常一样进行设置（见 [tutorial one](https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html)）：

RPC服务已经就绪，让我们启动它：

```bash
cd RPCServer
dotnet run
# => [x] Awaiting RPC requests
```

运行客户端来请求一个斐波那契数：

```bash
cd RPCClient
dotnet run
# => [x] Requesting fib(30)
```

这里所呈现的设计方式并不是实现RPC服务的唯一方法，但是它具备一些重要的优势：

- 如果RPC服务器过慢，你可以通过再运行一个服务器来进行扩展。试试在一个新的控制台中运行第二个`RPCServer`吧。
- 在客户端这边，RPC只需要发送和接收一条消息。不需要类似`QueueDeclare`这样的同步调用。因此RPC客户端在一次RPC请求中，只需要进行一次网络的往返。

我们的代码依旧很简洁，并且只去尝试解决重要的而不是更加复杂的问题，比如：

- 如果没有服务器运行，客户端是不是要做出反应？
- 客户端是不是需要有针对RPC的某种超时设置？
- 如果服务器发生故障，抛出异常，是不是需要转发给客户端？
- 在进行处理之前防止无效的消息传入（比如检查边界、类型）。

> 如果你想进行一下实验。会发现[管理界面](https://www.rabbitmq.com/management.html) 对于浏览队列来说用处很大。
