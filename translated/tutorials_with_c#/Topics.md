## Topics

## 主题

### (using the .NET client)

### (使用.NET客户端)



In the [previous tutorial](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html) we improved our logging system. Instead of using a fanout exchange only capable of dummy broadcasting, we used a direct one, and gained a possibility of selectively receiving the logs.
[上个教程](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html)里，我们对日志系统进行了改进。我们用直连交换机取代了只会无脑广播的扇形交换机，并且有能力选择性地接收日志。

Although using the direct exchange improved our system, it still has limitations - it can't do routing based on multiple criteria.

尽管使用了直连交换机来改进了我们的系统，但是仍有一点缺陷——它仍然不能基于多个条件进行路由。

In our logging system we might want to subscribe to not only logs based on severity, but also based on the source which emitted the log. You might know this concept from the [syslog](http://en.wikipedia.org/wiki/Syslog) unix tool, which routes logs based on both severity (info/warn/crit...) and facility (auth/cron/kern...).

在我们的日志系统中，我们除了想要根据严重性来订阅消息外，还想根据发送日志的来源进行订阅。你之前有可能通过一个名叫 [`syslog`](http://en.wikipedia.org/wiki/Syslog)的Unix工具了解过这种情形，它是通过严重性(info/warn/crit...)和设备(auth/cron/kern...)来对日志进行路由的。

That would give us a lot of flexibility - we may want to listen to just critical errors coming from 'cron' but also all logs from 'kern'.

这会给我们带来很大的灵活性——可能我们想要监听的是来自'cron'的关键错误和来自 'kern'的所有日志。

To implement that in our logging system we need to learn about a more complex topicexchange.

想要在日志系统中实现以上的功能，我们需要学一下更复杂的主题交换机。

## Topic exchange

## 主题交换机

Messages sent to a topic exchange can't have an arbitrary routing_key - it must be a list of words, delimited by dots. The words can be anything, but usually they specify some features connected to the message. A few valid routing key examples: "stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit". There can be as many words in the routing key as you like, up to the limit of 255 bytes.

发送到主题交换机的消息所携带的`routing_key`不能任意命名——它必须是一个用点号分隔的词列表。当中的词可以是任何单词，不过一般都会指定一些跟消息有关的特征作为这些单词。列举几个有效的routing key例子："`stock.usd.nyse`", "`nyse.vmw`", "`quick.orange.rabbit`"。只要不超过255个字节，词的长度由你来定。

The binding key must also be in the same form. The logic behind the topic exchange is similar to a direct one - a message sent with a particular routing key will be delivered to all the queues that are bound with a matching binding key. However there are two important special cases for binding keys:

- \* (star) can substitute for exactly one word.
- \# (hash) can substitute for zero or more words.

It's easiest to explain this in an example:

![img](https://www.rabbitmq.com/img/tutorials/python-five.png)

In this example, we're going to send messages which all describe animals. The messages will be sent with a routing key that consists of three words (two dots). The first word in the routing key will describe speed, second a colour and third a species: "<speed>.<colour>.<species>".

We created three bindings: Q1 is bound with binding key "*.orange.*" and Q2 with "*.*.rabbit" and "lazy.#".

These bindings can be summarised as:

- Q1 is interested in all the orange animals.
- Q2 wants to hear everything about rabbits, and everything about lazy animals.

A message with a routing key set to "quick.orange.rabbit" will be delivered to both queues. Message "lazy.orange.elephant" also will go to both of them. On the other hand "quick.orange.fox" will only go to the first queue, and "lazy.brown.fox" only to the second. "lazy.pink.rabbit" will be delivered to the second queue only once, even though it matches two bindings. "quick.brown.fox" doesn't match any binding so it will be discarded.

What happens if we break our contract and send a message with one or four words, like "orange" or "quick.orange.male.rabbit"? Well, these messages won't match any bindings and will be lost.

On the other hand "lazy.orange.male.rabbit", even though it has four words, will match the last binding and will be delivered to the second queue.

> #### Topic exchange
>
> Topic exchange is powerful and can behave like other exchanges.
>
> When a queue is bound with "#" (hash) binding key - it will receive all the messages, regardless of the routing key - like in fanout exchange.
>
> When special characters "*" (star) and "#" (hash) aren't used in bindings, the topic exchange will behave just like a direct one.

## Putting it all together

We're going to use a topic exchange in our logging system. We'll start off with a working assumption that the routing keys of logs will have two words: "<facility>.<severity>".

The code is almost the same as in the [previous tutorial](https://www.rabbitmq.com/tutorials/tutorial-four-dotnet.html).

The code for EmitLogTopic.cs:

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

To receive all the logs:

```bash
cd ReceiveLogsTopic
dotnet run "#"
```

To receive all logs from the facility "kern":

```bash
cd ReceiveLogsTopic
dotnet run "kern.*"
```

Or if you want to hear only about "critical" logs:

```bash
cd ReceiveLogsTopic
dotnet run "*.critical"
```

You can create multiple bindings:

```bash
cd ReceiveLogsTopic
dotnet run "kern.*" "*.critical"
```

And to emit a log with a routing key "kern.critical" type:

```bash
cd EmitLogTopic
dotnet run "kern.critical" "A critical kernel error"
```

Have fun playing with these programs. Note that the code doesn't make any assumption about the routing or binding keys, you may want to play with more than two routing key parameters.

(Full source code for [EmitLogTopic.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/EmitLogTopic/EmitLogTopic.cs) and [ReceiveLogsTopic.cs](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/dotnet/ReceiveLogsTopic/ReceiveLogsTopic.cs))

Next, find out how to do a round trip message as a remote procedure call in [tutorial 6](https://www.rabbitmq.com/tutorials/tutorial-six-dotnet.html)