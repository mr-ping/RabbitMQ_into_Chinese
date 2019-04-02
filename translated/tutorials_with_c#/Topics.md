> ### 前置条件

> 本教程假设RabbitMQ已经[安装](http://www.rabbitmq.com/download.html)在你本机的 (5672)端口。如果你使用了不同的主机、端口或者凭证，连接设置就需要作出一些对应的调整。

> ### 如何获得帮助

> 如果你在使用本教程的过程中遇到了麻烦，你可以通过邮件列表来[联系我们](https://groups.google.com/forum/#!forum/rabbitmq-users)。

## Topics

## 主题

### (using the .NET client)

### (使用.NET客户端)



In the [previous tutorial](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html) we improved our logging system. Instead of using a fanout exchange only capable of dummy broadcasting, we used a direct one, and gained a possibility of selectively receiving the logs.

[上个教程](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html)里，我们对日志系统进行了改进。我们用直连交换机取代了只会无脑广播的扇形交换机，并且具备了选择性接收日志的能力。

Although using the direct exchange improved our system, it still has limitations - it can't do routing based on multiple criteria.

尽管使用直连交换机来改进了我们的系统，但是仍有一点缺陷——仍然不能基于多个条件进行路由。

In our logging system we might want to subscribe to not only logs based on severity, but also based on the source which emitted the log. You might know this concept from the [syslog](http://en.wikipedia.org/wiki/Syslog) unix tool, which routes logs based on both severity (info/warn/crit...) and facility (auth/cron/kern...).

在我们的日志系统中，我们除了想要根据严重性来订阅消息外，还想根据发送日志的来源进行订阅。你之前有可能通过一个名叫 [`syslog`](http://en.wikipedia.org/wiki/Syslog)的Unix工具了解过这种情形，它是通过严重性(info/warn/crit...)和设备(auth/cron/kern...)来对日志进行路由的。

That would give us a lot of flexibility - we may want to listen to just critical errors coming from 'cron' but also all logs from 'kern'.

通过多种条件进行路由会给我们带来很大的灵活性——比如可能我们想要监听的是来自'cron'的关键(`critical`)错误和来自 'kern'的所有日志。

To implement that in our logging system we need to learn about a more complex topicexchange.

想要在日志系统中实现以上的功能，我们需要学一下更复杂的主题交换机。

## Topic exchange

## 主题交换机

Messages sent to a topic exchange can't have an arbitrary routing_key - it must be a list of words, delimited by dots. The words can be anything, but usually they specify some features connected to the message. A few valid routing key examples: "stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit". There can be as many words in the routing key as you like, up to the limit of 255 bytes.

发送到主题交换机的消息所携带的路由键（`routing_key`）不能随意命名——它必须是一个用点号分隔的词列表。当中的词可以是任何单词，不过一般都会指定一些跟消息有关的特征作为这些单词。列举几个有效的路由键的例子："`stock.usd.nyse`", "`nyse.vmw`", "`quick.orange.rabbit`"。只要不超过255个字节，词的长度由你来定。

The binding key must also be in the same form. The logic behind the topic exchange is similar to a direct one - a message sent with a particular routing key will be delivered to all the queues that are bound with a matching binding key. However there are two important special cases for binding keys:

绑定键（`binding key`）也得使用相同的格式。主题交换机背后的逻辑跟直连交换机比较相似——一条携带特定路由键（`routing key`）的消息会被投送给所有绑定键（`binding key`）与之相匹配的队列。尽管如此，仍然有两条与绑定键相关的特殊情况：

- \`*` (star) can substitute for exactly one word.
- \`#` (hash) can substitute for zero or more words.

- \`*` (星号) 能够替代一个单词。
- \`#` (井号) 能够替代零个或多个单词。

It's easiest to explain this in an example:

用一个例子可以很容易地解释：

![img](https://www.rabbitmq.com/img/tutorials/python-five.png)

In this example, we're going to send messages which all describe animals. The messages will be sent with a routing key that consists of three words (two dots). The first word in the routing key will describe speed, second a colour and third a species: "<speed>.<colour>.<species>".

此例中，我们将会发送用来描述动物的多条消息。发送的消息包含带有三个单词（两个点号）的路由键（`routing key`）。路由键中第一个单词描述速度，第二个单词是颜色，第三个是品种： "`<速度>.<颜色>.<品种>`"。

We created three bindings: Q1 is bound with binding key "`*.orange.*`" and Q2 with "`*.*.rabbit`" and "`lazy.#`".

我们创建三个绑定：Q1通过"`*.orange.*`"绑定键进行绑定，Q2使用"`*.*.rabbit`" 和 "`lazy.#`"。

These bindings can be summarised as:

这些绑定可以总结为：

- Q1 is interested in all the orange animals.
- Q1针对所有的橘色`orange`动物。
- Q2 wants to hear everything about rabbits, and everything about lazy animals.
- Q2针对每一个有关兔子`rabbits`和慵懒`lazy`的动物的消息。

A message with a routing key set to "quick.orange.rabbit" will be delivered to both queues. Message "lazy.orange.elephant" also will go to both of them. On the other hand "quick.orange.fox" will only go to the first queue, and "lazy.brown.fox" only to the second. "lazy.pink.rabbit" will be delivered to the second queue only once, even though it matches two bindings. "quick.brown.fox" doesn't match any binding so it will be discarded.

一个带有"`quick.orange.rabbit`"绑定键的消息会给两个队列都进行投送。消息"`lazy.orange.elephant`"也会投送给这两个队列。另外一方面，"`quick.orange.fox`" 只会给第一个队列。"`lazy.pink.rabbit`"虽然与两个绑定键都匹配，但只会给第二个队列投送一遍。"`quick.brown.fox`" 没有匹配到任何绑定，因此会被丢弃掉。

What happens if we break our contract and send a message with one or four words, like "orange" or "quick.orange.male.rabbit"? Well, these messages won't match any bindings and will be lost.

如果我们破坏规则，发送的消息只带有一个或者四个单词，例如 "`orange`" 或者 "`quick.orange.male.rabbit`"会发生什么呢？结果是这些消息不会匹配到任何绑定，将会被丢弃。

On the other hand "lazy.orange.male.rabbit", even though it has four words, will match the last binding and will be delivered to the second queue.

另一方面，“`lazy.orange.male.rabbit`”即使有四个单词，也会与最后一个绑定匹配，并 被投送到第二个队列。

> #### Topic exchange
> #### 主题交换机
>
> Topic exchange is powerful and can behave like other exchanges.
> 主题交换机非常强大，并且可以表现的跟其他交换机相似。
>
> When a queue is bound with "#" (hash) binding key - it will receive all the messages, regardless of the routing key - like in fanout exchange.
> 当一个队列使用"`#`"（井号）绑定键进行绑定。它会表现的像扇形交换机一样，不理会路由键，接收所有消息。
>
> When special characters "*" (star) and "#" (hash) aren't used in bindings, the topic exchange will behave just like a direct one.
> 当绑定当中不包含任何一个 "*" (星号) 和 "#" (井号)特殊字符的时候，主题交换机会表现的跟直连交换机一毛一样。

## Putting it all together

## 整合到一起

We're going to use a topic exchange in our logging system. We'll start off with a working assumption that the routing keys of logs will have two words: "<facility>.<severity>".

我们将在日志系统中使用主题交换机。我们现从一个可行的假设开始，假设日志的路由键包含两个单词： "`<设施>.<严重性>`".

The code is almost the same as in the [previous tutorial](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html).

代码跟[上个教程](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html)非常相似：

The code for EmitLogTopic.cs:

`EmitLogTopic.cs`的代码：

```csharp
using System;
using System.Linq;
using RabbitMQ.Client;
using System.Text;

class EmitLogTopic
{
    public static void Main(string[] args)
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.ExchangeDeclare(exchange: "topic_logs",
                                    type: "topic");

            var routingKey = (args.Length > 0) ? args[0] : "anonymous.info";
            var message = (args.Length > 1)
                          ? string.Join(" ", args.Skip( 1 ).ToArray())
                          : "Hello World!";
            var body = Encoding.UTF8.GetBytes(message);
            channel.BasicPublish(exchange: "topic_logs",
                                 routingKey: routingKey,
                                 basicProperties: null,
                                 body: body);
            Console.WriteLine(" [x] Sent '{0}':'{1}'", routingKey, message);
        }
    }
}
```

The code for ReceiveLogsTopic.cs:

`ReceiveLogsTopic.cs`的代码：

```csharp
using System;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

class ReceiveLogsTopic
{
    public static void Main(string[] args)
    {
        var factory = new ConnectionFactory() { HostName = "localhost" };
        using(var connection = factory.CreateConnection())
        using(var channel = connection.CreateModel())
        {
            channel.ExchangeDeclare(exchange: "topic_logs", type: "topic");
            var queueName = channel.QueueDeclare().QueueName;

            if(args.Length < 1)
            {
                Console.Error.WriteLine("Usage: {0} [binding_key...]",
                                        Environment.GetCommandLineArgs()[0]);
                Console.WriteLine(" Press [enter] to exit.");
                Console.ReadLine();
                Environment.ExitCode = 1;
                return;
            }

            foreach(var bindingKey in args)
            {
                channel.QueueBind(queue: queueName,
                                  exchange: "topic_logs",
                                  routingKey: bindingKey);
            }

            Console.WriteLine(" [*] Waiting for messages. To exit press CTRL+C");

            var consumer = new EventingBasicConsumer(channel);
            consumer.Received += (model, ea) =>
            {
                var body = ea.Body;
                var message = Encoding.UTF8.GetString(body);
                var routingKey = ea.RoutingKey;
                Console.WriteLine(" [x] Received '{0}':'{1}'",
                                  routingKey,
                                  message);
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

Run the following examples:

运行一下例子：

To receive all the logs:

接收所有日志：

```bash
cd ReceiveLogsTopic
dotnet run "#"
```

To receive all logs from the facility "kern":

接收来自于"`kern`"设施的所有日志：

```bash
cd ReceiveLogsTopic
dotnet run "kern.*"
```

Or if you want to hear only about "critical" logs:

或者如果你只想接收跟”严重“（”`critical`“）程度有关的日志：

```bash
cd ReceiveLogsTopic
dotnet run "*.critical"
```

You can create multiple bindings:

你可以创建多个绑定：

```bash
cd ReceiveLogsTopic
dotnet run "kern.*" "*.critical"
```

And to emit a log with a routing key "kern.critical" type:

然后发送一个路由键为"`kern.critical`"的日志：

```bash
cd EmitLogTopic
dotnet run "kern.critical" "A critical kernel error"
```

Have fun playing with these programs. Note that the code doesn't make any assumption about the routing or binding keys, you may want to play with more than two routing key parameters.

有意思吧？需要注意的是代码并没有对路由键或者绑定键做任何假定，你仍然可以用多于两个路由参数。

(Full source code for [EmitLogTopic.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/EmitLogTopic/EmitLogTopic.cs) and [ReceiveLogsTopic.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/ReceiveLogsTopic/ReceiveLogsTopic.cs))

([EmitLogTopic.cs的完整代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/EmitLogTopic/EmitLogTopic.cs) 和 [ReceiveLogsTopic.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/ReceiveLogsTopic/ReceiveLogsTopic.cs))

Next, find out how to do a round trip message as a remote procedure call in [tutorial 6](https://www.rabbitmq.com/tutorials/tutorial-six-dotnet.html)、

接下来，可以[教程 6](https://www.rabbitmq.com/tutorials/tutorial-six-dotnet.html)会介绍到如何像远程过程调用一样操作一个往返的消息。