## Remote procedure call (RPC)

## 远程过程调用

### (using the .NET client)

### (使用 .NET 客户端)



### Prerequisites

This tutorial assumes RabbitMQ is [installed](https://www.rabbitmq.com/download.html) and running on localhost on standard port (5672). In case you use a different host, port or credentials, connections settings would require adjusting.

### Where to get help

If you're having trouble going through this tutorial you can [contact us](https://groups.google.com/forum/#!forum/rabbitmq-users) through the mailing list.



  In the [second tutorial](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html) we learned how to use *Work Queues* to distribute time-consuming tasks among multiple workers.

在[第二个教程](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html)中，我们学习了如何使用*工作队列* 在多个工作者之间分配耗时任务。

But what if we need to run a function on a remote computer and wait for the result? Well, that's a different story. This pattern is commonly known as *Remote Procedure Call* or *RPC*.

不过如果我们需要在一个远程电脑上运行函数并且等待结果的时候会怎样呢。这又是另一个故事了。这种模式通常被称为*远程过程调用*或者*RPC*。

In this tutorial we're going to use RabbitMQ to build an RPC system: a client and a scalable RPC server. As we don't have any time-consuming tasks that are worth distributing, we're going to create a dummy RPC service that returns Fibonacci numbers.

这个教程里，我们用RabbitMQ来建立一个RPC系统：一个客户端和一个可扩展的RPC服务器。因为没有任何耗时任务用于分发，我们会创建一个伪造的用来返回斐波那契数列的RPC服务。

### Client interface

### 客户端接口

To illustrate how an RPC service could be used we're going to create a simple client class. It's going to expose a method named Call which sends an RPC request and blocks until the answer is received:

为了说明如何使用一个RPC服务，我们创建一个简单的客户端类。它会暴露一个名为Call的方法，此方法发送一个RPC请求，然后阻塞到收到回答为止。

```csharp
var rpcClient = new RPCClient();

Console.WriteLine(" [x] Requesting fib(30)");
var response = rpcClient.Call("30");
Console.WriteLine(" [.] Got '{0}'", response);

rpcClient.Close();
```

> #### A note on RPC
> #### 有关RPC的说明
>
> Although RPC is a pretty common pattern in computing, it's often criticised. The problems arise when a programmer is not aware whether a function call is local or if it's a slow RPC. Confusions like that result in an unpredictable system and adds unnecessary complexity to debugging. Instead of simplifying software, misused RPC can result in unmaintainable spaghetti code.
> 虽然RPC在运算中是个非常常见的模式，但是也常常被批评。问题出在程序员察觉不到函数调用是发生在本地还是发生在一个很慢的RPC当中。这种困惑导致了系统的不可预测性并且为调试增加了不必要的复杂度。跟简单的软件相比，RPC会导致不可维护的面条式代码（译者注：[面条式代码](https://bkso.baidu.com/item/%E9%9D%A2%E6%9D%A1%E5%BC%8F%E4%BB%A3%E7%A0%81)在软件工程中是一种典型的反面模式）。
>
> Bearing that in mind, consider the following advice:
> 考虑到这点，请斟酌以下建议：
>
> - Make sure it's obvious which function call is local and which is remote.
> - 确保一眼就能看出来哪个方法是本地执行的，哪个方法是远程执行的。
> - Document your system. Make the dependencies between components clear.
> - 给你的系统写好文档。让组件之间的依赖清晰可查。
> - Handle error cases. How should the client react when the RPC server is down for a long time?
> - 处理错误用例。如果RPC宕掉的时间过长，客户端该如何反应。
>
> When in doubt avoid RPC. If you can, you should use an asynchronous pipeline - instead of RPC-like blocking, results are asynchronously pushed to a next computation stage.
> 当有疑虑的时候，请避免实用RPC。有可能的话，应该使用异步管道来替代RPC式的阻塞，从而将结果异步地推送给下一个计算阶段。

### Callback queue

### 回调队列

In general doing RPC over RabbitMQ is easy. A client sends a request message and a server replies with a response message. In order to receive a response we need to send a 'callback' queue address with the request:

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

> #### Message properties
> #### 消息属性
>
> The AMQP 0-9-1 protocol predefines a set of 14 properties that go with a message. Most of the properties are rarely used, with the exception of the following:
> AMQP 0-9-1 协议与定义了14个消息的属性。大部分属性很少会用到，不过以下几个例外：
>
> - Persistent: : Marks a message as persistent (with a value of 2) or transient (any other value). Take a look at [the second tutorial](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html).
> - Persistent: 标示此信息为持久的（用值为2来表示）还是暂时的（2之外的其他任何值）。具体可以去 [第二个教程](https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html) 一探究竟。
> - DeliveryMode: those familiar with the protocol may choose to use this property instead of Persistent. They control the same thing.
> - DeliveryMode：那些熟悉协议的人可能会选择使用这个属性，而不是Persistent。他们办的是一个事情。
> - ContentType: Used to describe the mime-type of the encoding. For example for the often used JSON encoding it is a good practice to set this property to: application/json.
> - ContentType：用来描述编码的mime-type。例如对于经常用到的JSON编码来说，将此属性设置为application/json是一个很好的做法。
> - ReplyTo: Commonly used to name a callback queue.
> - ReplyTo: 通常用来对一个回调队列进行命名。
> - CorrelationId: Useful to correlate RPC responses with requests.
> - CorrelationId: 用于将RPC响应和请求进行关联。

### Correlation Id

### 关联id

In the method presented above we suggest creating a callback queue for every RPC request. That's pretty inefficient, but fortunately there is a better way - let's create a single callback queue per client.

上面介绍的方法中，我们建议为每一个RPC请求创建一个回调队列。这样做效率很低，幸好我们有更好的解决办法，让我们为每个客户端创建一个单独的回调队列。

That raises a new issue, having received a response in that queue it's not clear to which request the response belongs. That's when the CorrelationId property is used. We're going to set it to a unique value for every request. Later, when we receive a message in the callback queue we'll look at this property, and based on that we'll be able to match a response with a request. If we see an unknown CorrelationId value, we may safely discard the message - it doesn't belong to our requests.

这样又有一个新问题，当我们从此队列里收到一个响应的时候并不清楚它是属于哪个请求的。这就是CorrelationId属性的用途所在了。我们为每一个请求将其设置为一个唯一值。稍后，当我们从回调队列里收到一条消息的时候，就可以通过这个属性来对响应和请求进行匹配。如果我们收到的消息的CorrelationId值是未知的，那就可以安心的把它丢弃掉，因为它并不属于我们的请求。

You may ask, why should we ignore unknown messages in the callback queue, rather than failing with an error? It's due to a possibility of a race condition on the server side. Although unlikely, it is possible that the RPC server will die just after sending us the answer, but before sending an acknowledgment message for the request. If that happens, the restarted RPC server will process the request again. That's why on the client we must handle the duplicate responses gracefully, and the RPC should ideally be idempotent.

或许你会问，我们为什么不把回调队列里的未知消息当成错误的失败来处理，而是要把它忽略掉？这是由于服务器端存在竞争条件的可能。虽然可能性不大，但是RPC服务器是有可能在发送给我们回应之后挂掉的，可此时它并没有完成为请求发回确认（acknowledgment）的动作。一旦这种情况发生，RPC服务器会再次将那条请求处理一遍。这就是为什么我们需要在客户端里优雅的处理重复的响应，并且在理想情况下，RPC应该是幂等的。

### Summary

### 总结

![img](https://www.rabbitmq.com/img/tutorials/python-six.png)

Our RPC will work like this:

我们的RPC看起来是这样的：

- When the Client starts up, it creates an anonymous exclusive callback queue.
- 当客户端启动时，会创建一个匿名的独占回调队列。
- For an RPC request, the Client sends a message with two properties: ReplyTo, which is set to the callback queue and CorrelationId, which is set to a unique value for every request.
- 客户端发送一条带有ReplyTo和CorrelationId两个属性的消息作为一个RPC请求，ReplyTo用于设置回调队列，CorrelationId用于为每一个请求设置一个独一无二的值。
- The request is sent to an rpc_queue queue.
- 请求被发送到一个rpc_queue队列。
- The RPC worker (aka: server) is waiting for requests on that queue. When a request appears, it does the job and sends a message with the result back to the Client, using the queue from the ReplyTo property.
- RPC工作者（也称为服务器）等待从那个队列中接受请求。当一个请求出现的时候，他会执行任务并且通过ReplyTo 属性所提及的队列来将带有执行结果的消息发回给客户端。
- The client waits for data on the callback queue. When a message appears, it checks the CorrelationId property. If it matches the value from the request it returns the response to the application.
- 客户端从回调队列那儿等待数据。当消息出现的时候，它会检查CorrelationId属性。如果属性值跟请求相匹配，就将响应返回给应用。

## Putting it all together

The Fibonacci task:

```csharp
private static int fib(int n)
{
    if (n == 0 || n == 1) return n;
    return fib(n - 1) + fib(n - 2);
}
```

We declare our fibonacci function. It assumes only valid positive integer input. (Don't expect this one to work for big numbers, and it's probably the slowest recursive implementation possible).

The code for our RPC server [RPCServer.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/RPCServer/RPCServer.cs) looks like this:

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

The server code is rather straightforward:

- As usual we start by establishing the connection, channel and declaring the queue.
- We might want to run more than one server process. In order to spread the load equally over multiple servers we need to set the prefetchCount setting in channel.BasicQos.
- We use BasicConsume to access the queue. Then we register a delivery handler in which we do the work and send the response back.

The code for our RPC client [RPCClient.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/RPCClient/RPCClient.cs):

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

The client code is slightly more involved:

- We establish a connection and channel and declare an exclusive 'callback' queue for replies.
- We subscribe to the 'callback' queue, so that we can receive RPC responses.
- Our Call method makes the actual RPC request.
- Here, we first generate a unique CorrelationId number and save it - the while loop will use this value to catch the appropriate response.
- Next, we publish the request message, with two properties: ReplyTo and CorrelationId.
- At this point we can sit back and wait until the proper response arrives.
- The while loop is doing a very simple job, for every response message it checks if the CorrelationId is the one we're looking for. If so, it saves the response.
- Finally we return the response back to the user.

Making the Client request:

```csharp
var rpcClient = new RPCClient();

Console.WriteLine(" [x] Requesting fib(30)");
var response = rpcClient.Call("30");
Console.WriteLine(" [.] Got '{0}'", response);

rpcClient.Close();
```

Now is a good time to take a look at our full example source code (which includes basic exception handling) for [RPCClient.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/RPCClient/RPCClient.cs) and [RPCServer.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/RPCServer/RPCServer.cs).

Set up as usual (see [tutorial one](https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html)):

Our RPC service is now ready. We can start the server:

```bash
cd RPCServer
dotnet run
# => [x] Awaiting RPC requests
```

To request a fibonacci number run the client:

```bash
cd RPCClient
dotnet run
# => [x] Requesting fib(30)
```

The design presented here is not the only possible implementation of a RPC service, but it has some important advantages:

- If the RPC server is too slow, you can scale up by just running another one. Try running a second RPCServer in a new console.
- On the client side, the RPC requires sending and receiving only one message. No synchronous calls like QueueDeclare are required. As a result the RPC client needs only one network round trip for a single RPC request.

Our code is still pretty simplistic and doesn't try to solve more complex (but important) problems, like:

- How should the client react if there are no servers running?
- Should a client have some kind of timeout for the RPC?
- If the server malfunctions and raises an exception, should it be forwarded to the client?
- Protecting against invalid incoming messages (eg checking bounds, type) before processing.

> If you want to experiment, you may find the [management UI](https://www.rabbitmq.com/management.html) useful for viewing the queues.