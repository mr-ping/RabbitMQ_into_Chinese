> ### 前置条件

> 本教程假设RabbitMQ已经[安装](http://www.rabbitmq.com/download.html)在你本机的 (5672)端口。如果你使用了不同的主机、端口或者凭证，连接设置就需要作出一些对应的调整。

> ### 如何获得帮助

> 如果你在使用本教程的过程中遇到了麻烦，你可以通过邮件列表来[联系我们](https://groups.google.com/forum/#!forum/rabbitmq-users)。

## Introduction

## 介绍

RabbitMQ is a message broker: it accepts and forwards messages. You can think about it as a post office: when you put the mail that you want posting in a post box, you can be sure that Mr. or Ms. Mailperson will eventually deliver the mail to your recipient. In this analogy, RabbitMQ is a post box, a post office and a postman.

RabbitMQ 是一个消息代理：它用来接收消息，并将其进行转发。 你可以把它想象成一个邮局：当你把想要邮寄的邮件放到邮箱里后，邮递员就会把邮件最终送到目的地。 在这个比喻中，RabbitMQ既代表了邮箱，也同时扮演着邮局和邮递员的角色.

The major difference between RabbitMQ and the post office is that it doesn't deal with paper, instead it accepts, stores and forwards binary blobs of data ‒ messages.

RabbitMQ和邮局主要区别在于，RabbitMQ不处理纸质信件，取而代之，它接收、存储和转发的是被称为*消息*的二进制数据块。

RabbitMQ, and messaging in general, uses some jargon.

下面介绍下通常情况下会用到的一些RabbitMQ和messaging术语：

- *Producing* means nothing more than sending. A program that sends messages is a *producer* :
- *生产者*就是值得发送端。一个用来发送消息的*生产*程序。

  ![img](http://www.rabbitmq.com/img/tutorials/producer.png)

  

- *A queue* is the name for a post box which lives inside RabbitMQ. Although messages flow through RabbitMQ and your applications, they can only be stored inside a *queue*. A *queue*is only bound by the host's memory & disk limits, it's essentially a large message buffer. Many *producers* can send messages that go to one queue, and many *consumers* can try to receive data from one *queue*. This is how we represent a queue:
- *队列*指的是存在与RabbitMQ中的邮箱。虽然消息是在RabbbitMQ和你的应用程序之间流转，但是他们是存储在*队列*中的。*队列*实质上是存在于主机内存和硬盘中的消息缓冲区。多个*生产者*可以发送消息到同一个*队列*中，多个*消费者*也可以从同一个*队列*中接收消息。我们这样来描述一个队列：

  ![img](http://www.rabbitmq.com/img/tutorials/queue.png)

  

- *Consuming* has a similar meaning to receiving. A *consumer* is a program that mostly waits to receive messages:
- *消费者*跟接收端是同一个意思。一个*消费者*就是一个主要用来等待接收消息的程序：

  ![img](http://www.rabbitmq.com/img/tutorials/consumer.png)

  

Note that the producer, consumer, and broker do not have to reside on the same host; indeed in most applications they don't. An application can be both a producer and consumer, too.

需要注意的是，生产者、消费者和代理不需要存在于同一个主机上; 实际上，大多数应用中也确实如此。另外，一个应用程序也可以同时充当生产者和消费者两个角色。



## "Hello World"

### (using the .NET/C# Client)

### (使用 .NET/C# 客户端)

In this part of the tutorial we'll write two programs in C#; a producer that sends a single message, and a consumer that receives messages and prints them out. We'll gloss over some of the detail in the .NET client API, concentrating on this very simple thing just to get started. It's a "Hello World" of messaging.

在教程的这个部分，我们将使用C#语言编写两个程序；一个生产者负责发送单条信息，一个消费者负责接收这条信息并将其打印出来。我们会有意忽略.NET 客户端接口的一些细节，专注于利用这个及其简单的例子来打开局面。这个例子就是传送"Hello World"消息。

In the diagram below, "P" is our producer and "C" is our consumer. The box in the middle is a queue - a message buffer that RabbitMQ keeps on behalf of the consumer.

下方的图例中，“P”是我们的生产者，“C”是我们的消费者。中间的盒子代表RabbitMQ中用来为消费者保持消息的缓冲区——队列。

![(P) -> [|||] -> (C)](http://www.rabbitmq.com/img/tutorials/python-one.png)

> #### The .NET client library
> 
> #### .NET客户端库
>
> RabbitMQ speaks multiple protocols. This tutorial uses AMQP 0-9-1这个, which is an open, general-purpose protocol for messaging. There are a number of clients for RabbitMQ in [many different languages](http://rabbitmq.com/devtools.html). We'll use the .NET client provided by RabbitMQ.
> 
> RabbitMQ有多种协议可用。次教程用的是AMQP 0-9-1这个用于消息传输的开放的通用协议。RabbitMQ有许多针对 [不同语言的客户端](http://rabbitmq.com/devtools.html)。这里我们使用RabbitMQ出品的.NET客户端。
>
> The client supports [.NET Core](https://www.microsoft.com/net/core) as well as .NET Framework 4.5.1+. This tutorial will use RabbitMQ .NET client 5.0 and .NET Core so you will ensure you have it [installed](https://www.microsoft.com/net/core) and in your PATH.
> 
> 此客户端支持 [.NET Core](https://www.microsoft.com/net/core) 以及 .NET Framework 4.5.1+。此教程会使用RabbitMQ .NET client 5.0 和 .NET Core，所以你需要确保已经安装了他们，并且配置到了PATH中。
>
> You can also use the .NET Framework to complete this tutorial however the setup steps will be different.
> 
> 当然你也可以使用.NET Framework来完成这个教程，但是安装过程会有所不同。
>
> RabbitMQ .NET client 5.0 and later versions are distributed via [nuget](https://www.nuget.org/packages/RabbitMQ.Client).
> 
> RabbitMQ .NET 客户端 5.0 是通过[nuget](https://www.nuget.org/packages/RabbitMQ.Client)来分发的。
>
> This tutorial assumes you are using powershell on Windows. On MacOS and Linux nearly any shell will work.
> 
> 本教程假设你使用的是Windows中的powershell。在MacOS 和 Linux 中几乎所有的shell都可以正常完成我们的工作。

### Setup

### 安装

First lets verify that you have .NET Core toolchain in PATH:

首先，让我们验证一下你PATH中的.NET Core工具连。

```powershell
dotnet --help
```

should produce a help message.

这里应该会生成一个帮助信息。

Now let's generate two projects, one for the publisher and one for the consumer:

现在我们来生成两个项目，一个作为发布者（译者注：指的就是生产者），一个作为消费者。

```powershell
dotnet new console --name Send
mv Send/Program.cs Send/Send.cs
dotnet new console --name Receive
mv Receive/Program.cs Receive/Receive.cs
```

This will create two new directories named Send and Receive.

这样就会分别创建两个名为Send和Receive的目录。

Then we add the client dependency.

然后我们来添加客户端依赖。

```powershell
cd Send
dotnet add package RabbitMQ.Client
dotnet restore
cd ../Receive
dotnet add package RabbitMQ.Client
dotnet restore
```

Now we have the .NET project set up we can write some code.

这样.NET项目就配置成功，我们可以着手写代码了。

### Sending

### 发送

![(P) -> [|||]](http://www.rabbitmq.com/img/tutorials/sending.png)

We'll call our message publisher (sender) Send.cs and our message consumer (receiver) Receive.cs。 The publisher will connect to RabbitMQ, send a single message, then exit.

我们将消息发布者（发送者）命名为Send.cs，将消息消费者（接收者）命名为Receive.cs。发布者将会链接到RabbitMQ，发送一条消息，然后退出。

In [Send.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Send/Send.cs), we need to use some namespaces:

在 [Send.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Send/Send.cs) 中, 我们需要使用一些命名空间：

```csharp
using System;
using RabbitMQ.Client;
using System.Text;
```

Set up the class:

配置类：

```csharp
class Send
{
    public static void Main()
    {
        ...
    }
}
```

then we can create a connection to the server:

然后创建到服务器的连接：

```csharp
class Send
{
    public static void Main()
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using (var connection = factory.CreateConnection())
        {
            using (var channel = connection.CreateModel())
            {
                ...
            }
        }
    }
}
```

The connection abstracts the socket connection, and takes care of protocol version negotiation and authentication and so on for us. Here we connect to a broker on the local machine - hence the *localhost*. If we wanted to connect to a broker on a different machine we'd simply specify its name or IP address here.

此连接为我们抽象了套接字链接，协议版本的协商以及验证。此处我们连接的是本地机器的代理，所以写的是*localhost*。如果我们打算链接其他机器上的代理，这里得写上那台机器的机器名或者IP地址。

Next we create a channel, which is where most of the API for getting things done resides.

接下来我们来创建一个信道，大部分API的工作都在此信道的基础上完成。

To send, we must declare a queue for us to send to; then we can publish a message to the queue:

为了发送小，我们必须声明一个用于送达消息的队列，然后我们会将消息发布到此队列中：

```csharp
using System;
using RabbitMQ.Client;
using System.Text;

class Send
{
    public static void Main()
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.QueueDeclare(queue: "hello",
                                 durable: false,
                                 exclusive: false,
                                 autoDelete: false,
                                 arguments: null);

            string message = "Hello World!";
            var body = Encoding.UTF8.GetBytes(message);

            channel.BasicPublish(exchange: "",
                                 routingKey: "hello",
                                 basicProperties: null,
                                 body: body);
            Console.WriteLine(" [x] Sent {0}", message);
        }

        Console.WriteLine(" Press [enter] to exit.");
        Console.ReadLine();
    }
}
```

Declaring a queue is idempotent - it will only be created if it doesn't exist already. The message content is a byte array, so you can encode whatever you like there.

队列的声明是幂等的，只有当这个队列不存在的时候才会被创建。消息的内容是一个比特数组，所以你可以按照自己的喜好对其编码。


When the code above finishes running, the channel and the connection will be disposed. That's it for our publisher.

当上方的代码运行完之后，信道和连接会被小毁掉。以上就是我们的发布者。

[Here's the whole Send.cs class](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Send/Send.cs).

[这是完整的Send.cs类代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Send/Send.cs).

> #### Sending doesn't work!
> 
> #### 没有发送成功!
>
> If this is your first time using RabbitMQ and you don't see the "Sent" message then you may be left scratching your head wondering what could be wrong. Maybe the broker was started without enough free disk space (by default it needs at least 50 MB free) and is therefore refusing to accept messages. Check the broker logfile to confirm and reduce the limit if necessary. The [configuration file documentation](http://www.rabbitmq.com/configure.html#config-items) will show you how to set disk_free_limit.
> 
> 如果这是你第一次使用RabbitMQ，而且没有成功看到"Sent"信息的输出，估计你会抓耳挠腮，不得其解。有可能只是因为你的硬盘没有足够的空间了（默认需要50MB空闲空间），所以才会把消息拒绝掉。你可以查看代理日志文件来进行确认，并且降低此限制条件。[配置文件文档](http://www.rabbitmq.com/configure.html#config-items) 会告诉你怎么点各地一个涅磐空闲空间限制。

### Receiving

### 接收

As for the consumer, it listening for messages from RabbitMQ. So unlike the publisher which publishes a single message, we'll keep the consumer running continuously to listen for messages and print them out.

作为消费者，它会监听来自与RabbitMQ的消息。所以不同与发布者只发送一条消息的做法，我们会让消费者持续运行来监听消息，并将消息打印出来。

![[|||] -> (C)](http://www.rabbitmq.com/img/tutorials/receiving.png)

The code (in [Receive.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Receive/Receive.cs)) has almost the same using statements as Send:

此处代码(在 [Receive.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Receive/Receive.cs)中) 跟发送代码非常相似：

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System;
using System.Text;
```

Setting up is the same as the publisher; we open a connection and a channel, and declare the queue from which we're going to consume. Note this matches up with the queue that Sendpublishes to.

配置跟发布者是一样的；我们会打开一个连接和一个信道，声明一个我们想要从其中获取消息的队列。注意这个队列需要与发布到信息的那个队列相匹配。

```csharp
class Receive
{
    public static void Main()
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using (var connection = factory.CreateConnection())
        {
            using (var channel = connection.CreateModel())
            {
                channel.QueueDeclare(queue: "hello",
                                     durable: false,
                                     exclusive: false,
                                     autoDelete: false,
                                     arguments: null);
                ...
            }
        }
    }
}
```

Note that we declare the queue here as well. Because we might start the consumer before the publisher, we want to make sure the queue exists before we try to consume messages from it.

你可能注意到了，这里我们又把队列声明了一次。这是因为，我们可能会先把消费者启动起来，而不是发布者。我们希望确保用于消费的队列是存在的（译者注：队列生命行为是幂等的，不会重复作用）。

We're about to tell the server to deliver us the messages from the queue. Since it will push us messages asynchronously, we provide a callback. That is what EventingBasicConsumer.Receivedevent handler does.

接下来我们通知服务器可以将消息从队列里发送过来了。由于服务器会异步的将消息推送给我们，所以我们这里提供一个回调方法。这就是EventingBasicConsumer.Receivedevent所做的工作。

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System;
using System.Text;

class Receive
{
    public static void Main()
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.QueueDeclare(queue: "hello",
                                 durable: false,
                                 exclusive: false,
                                 autoDelete: false,
                                 arguments: null);

            var consumer = new EventingBasicConsumer(channel);
            consumer.Received += (model, ea) =>
            {
                var body = ea.Body;
                var message = Encoding.UTF8.GetString(body);
                Console.WriteLine(" [x] Received {0}", message);
            };
            channel.BasicConsume(queue: "hello",
                                 autoAck: true,
                                 consumer: consumer);

            Console.WriteLine(" Press [enter] to exit.");
            Console.ReadLine();
        }
    }
}
```

[Here's the whole Receive.cs class](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Receive/Receive.cs).

[这是 Receive.cs 类的完整代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Receive/Receive.cs).

### Putting It All Together
### 将它们整合到一起

Open two terminals.

打开两个终端。

Run the consumer:

运行消费者：

```powershell
cd Receive
dotnet run
```

Then run the producer:

然后运行生产者：

```powershell
cd Send
dotnet run
```

The consumer will print the message it gets from the publisher via RabbitMQ. The consumer will keep running, waiting for messages (Use Ctrl-C to stop it), so try running the publisher from another terminal.

消费者会将从发布者通过RabbitMQ发送的接收到，并且打印出来。消费者会一直保持运行状态来等待接受消息（可以使用Ctrl-C 来将其停止），接下来我们可以试着在另一个终端里运行发布者代码来尝试发送消息。

Time to move on to [part 2](http://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html) and build a simple *work queue*.

接下来，我们可以移步[第二部分](http://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html) ，创建一个简单的*工作队列*。