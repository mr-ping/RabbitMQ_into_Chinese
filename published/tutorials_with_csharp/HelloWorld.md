>原文：[Hello World](https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html)  
>翻译：[mr-ping](http://rabbitmq.mr-ping.com)  
![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

> ### 前置条件

> 本教程假设RabbitMQ已经[安装](http://www.rabbitmq.com/download.html)在你本机的 (5672)端口。如果你使用了不同的主机、端口或者凭证，连接设置就需要作出一些对应的调整。

> ### 如何获得帮助

> 如果你在使用本教程的过程中遇到了麻烦，你可以通过邮件列表来[联系我们](https://groups.google.com/forum/#!forum/rabbitmq-users)。


## 介绍

RabbitMQ 是一个消息代理：它用来接收消息，并将其进行转发。 你可以把它想象成一个邮局：当你把想要邮寄的邮件放到邮箱里后，邮递员就会把邮件最终送达到目的地。 在这个比喻中，RabbitMQ既代表了邮箱，也同时扮演着邮局和邮递员的角色.

RabbitMQ和邮局主要区别在于，RabbitMQ不处理纸质信件，取而代之，它接收、存储和转发的是被称为*消息*的二进制数据块。

下面介绍下通常情况下会用到的一些RabbitMQ和messaging术语：

- *生产*就是指的发送。一个用来发送消息的*生产者*程序：

  ![img](http://www.rabbitmq.com/img/tutorials/producer.png)

- *队列*指的是存在于RabbitMQ当中的邮箱。虽然消息是在RabbbitMQ和你的应用程序之间流转，但他们是存储在*队列*中的。*队列*只收到主机内存和磁盘的限制，它实质上是存在于主机内存和硬盘中的消息缓冲区。多个*生产者*可以发送消息到同一个*队列*中，多个*消费者*也可以从同一个*队列*中接收消息。我们这样来描述一个队列：

  ![img](http://www.rabbitmq.com/img/tutorials/queue.png)

- *消费*跟接收基本是一个意思。一个*消费者*基本上就是一个用来等待接收消息的程序：

  ![img](http://www.rabbitmq.com/img/tutorials/consumer.png)

需要注意的是，生产者、消费者和代理不需要存在于同一个主机上; 实际上，大多数应用中也确实如此。另外，一个应用程序也可以同时充当生产者和消费者两个角色。

## "Hello World"

### (使用 .NET/C# 客户端)

在教程的这个部分，我们会使用C#语言编写两个程序；一个生产者负责发送单条信息，一个消费者负责接收这条信息并将其打印出来。我们会有意忽略.NET 客户端接口的一些细节，专注于利用这个及其简单的例子来打开局面。这个例子就是传送"Hello World"消息。

下方的图例中，“P”是我们的生产者，“C”是我们的消费者。中间的盒子代表RabbitMQ中用来为消费者保持消息的缓冲区——队列。

![(P) -> [|||] -> (C)](http://www.rabbitmq.com/img/tutorials/python-one.png)

> #### .NET客户端库
> 
> RabbitMQ有多种协议可用。本教程用的是AMQP 0-9-1这个用于消息传输的，开放的通用协议。RabbitMQ有许多针对 [不同语言的客户端](http://rabbitmq.com/devtools.html)。这里我们使用RabbitMQ出品的.NET客户端。
>
> 此客户端支持 [.NET Core](https://www.microsoft.com/net/core) 以及 .NET Framework 4.5.1+。本教程会使用RabbitMQ .NET client 5.0 和 .NET Core，所以你需要确保已经安装了他们，并且配置到了PATH当中。
> 
> 当然你也可以使用.NET Framework来完成这个教程，但是安装配置过程会有所不同。
> 
> RabbitMQ .NET 客户端 5.0 是通过[nuget](https://www.nuget.org/packages/RabbitMQ.Client)来分发的。
> 
> 本教程假设你使用的是Windows中的powershell。在MacOS 和 Linux 中几乎所有的shell都可以正常完成我们的工作。

### 安装

首先，让我们验证一下你的`PATH`中是否有.NET Core工具连。

```powershell
dotnet --help
```

这时应该会生成一个帮助信息。

现在我们来生成两个项目，一个作为发布者（译者注：指的就是生产者），一个作为消费者。

```powershell
dotnet new console --name Send
mv Send/Program.cs Send/Send.cs
dotnet new console --name Receive
mv Receive/Program.cs Receive/Receive.cs
```

这样就会分别创建两个名为`Send`和`Receive`的目录。

然后我们来添加客户端依赖。

```powershell
cd Send
dotnet add package RabbitMQ.Client
dotnet restore
cd ../Receive
dotnet add package RabbitMQ.Client
dotnet restore
```

这样.NET项目就配置成功，我们可以着手写代码了。

### 发送

![(P) -> [|||]](http://www.rabbitmq.com/img/tutorials/sending.png)

我们将消息发布者（发送者）命名为Send.cs，将消息消费者（接收者）命名为Receive.cs。发布者将会连接到RabbitMQ，发送一条消息，然后退出。

在 [Send.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Send/Send.cs) 中, 我们需要使用一些命名空间：

```csharp
using System;
using RabbitMQ.Client;
using System.Text;
```

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

此连接为我们抽象了套接字的链接，协议版本的协商以及验证。此处我们连接的是本地机器的代理，所以写的是*localhost*。如果我们打算链接其他机器上的代理，这里得写上那台机器的机器名或者IP地址。

接下来我们来创建一个信道，大部分API的工作都在此信道的基础上完成。

为了发送消息，我们必须声明一个用于送达消息的队列，然后我们会将消息发布到此队列中：

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

队列的声明是幂等的，只有当这个队列不存在的时候才会被创建。消息的内容是一个比特数组，所以你可以按照自己的喜好对其进行编码。

当上方的代码运行完成之后，信道和连接会被销毁掉。以上就是我们的发布者。

[这是完整的Send.cs类代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Send/Send.cs).

> #### 没有发送成功!
>
> 如果这是你第一次使用RabbitMQ，而且没有成功看到"Sent"信息的输出，估计你会抓耳挠腮，不得其解。有可能只是因为你的硬盘没有足够的空间了（默认需要50MB空闲空间），所以才会把消息拒绝掉。你可以查看代理日志文件来进行确认，并且降低此限制条件。[配置文件文档](http://www.rabbitmq.com/configure.html#config-items) 会告诉你如何设置`disk_free_limit`.

### 接收

消费者会监听来自与RabbitMQ的消息。所以不同于发布者只发送一条消息，我们会让消费者保持持续运行来监听消息，并将消息打印出来。

![[|||] -> (C)](http://www.rabbitmq.com/img/tutorials/receiving.png)

此代码(在 [Receive.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Receive/Receive.cs)中) 跟`Send`代码非常相似：

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System;
using System.Text;
```

配置工作跟发布者是一样的；我们打开一个连接和一个信道，声明一个我们想要从其中获取消息的队列。注意这个队列需要与`Send`发布到信息的那个队列相匹配。

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

你可能注意到了，这里我们又把队列声明了一次。这是因为，我们可能会先把消费者启动起来，而不是发布者。我们希望确保用于消费的队列是确实存在的。

接下来我们通知服务器可以将消息从队列里发送过来了。由于服务器会异步地将消息推送给我们，所以我们这里提供一个回调方法。这就是`EventingBasicConsumer.Receivedevent`所做的工作。

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

[这是 Receive.cs 类的完整代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/Receive/Receive.cs).

### 将它们整合到一起

打开两个终端。

运行消费者：

```powershell
cd Receive
dotnet run
```

然后运行生产者：

```powershell
cd Send
dotnet run
```

消费者会接收到发布者通过RabbitMQ发送的消息，并将其打印出来。消费者会一直保持运行状态来等待接受消息（可以使用Ctrl-C 来将其停止），接下来我们可以试着在另一个终端里运行发布者代码来尝试发送消息了。
ß
接下来，我们可以移步[第二部分](http://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html) ，创建一个简单的*工作队列*。
