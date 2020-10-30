>原文：[Publish/Subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-go.html)  
>状态：待校对  
>翻译：[Bingjian-Zhu](https://bingjian-zhu.github.io/)  
>校对：

![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

## 发布／订阅

**（使用Go客户端）**

<!--more-->

在[上篇教程](https://bingjian-zhu.github.io/2019/09/01/RabbitMQ%E6%95%99%E7%A8%8B%EF%BC%88%E8%AF%91%EF%BC%89-Work-queues/)中，我们搭建了一个工作队列，每个任务只分发给一个工作者（worker）。在本篇教程中，我们要做的跟之前完全不一样 —— 分发一个消息给多个消费者（consumers）。这种模式被称为“发布／订阅”。

为了描述这种模式，我们将会构建一个简单的日志系统。它包括两个程序——第一个程序负责发送日志消息，第二个程序负责获取消息并输出内容。

在我们的这个日志系统中，所有正在运行的接收方程序都会接受消息。我们用其中一个接收者（receiver）把日志写入硬盘中，另外一个接受者（receiver）把日志输出到屏幕上。

最终，日志消息被广播给所有的接受者（receivers）。

## 交换机（Exchanges）

前面的教程中，我们发送消息到队列并从中取出消息。现在是时候介绍RabbitMQ中完整的消息模型了。

让我们简单的概括一下之前的教程：

* 发布者（producer）是发布消息的应用程序。
* 队列（queue）用于消息存储的缓冲。
* 消费者（consumer）是接收消息的应用程序。

RabbitMQ消息模型的核心理念是：发布者（producer）不会直接发送任何消息给队列。事实上，发布者（producer）甚至不知道消息是否已经被投递到队列。

发布者（producer）只需要把消息发送给一个交换机（exchange）。交换机非常简单，它一边从发布者方接收消息，一边把消息推送到队列。交换机必须知道如何处理它接收到的消息，是应该推送到指定的队列还是是多个队列，或者是直接忽略消息。这些规则是通过交换机类型（exchange type）来定义的。

![](http://www.rabbitmq.com/img/tutorials/exchanges.png)

有几个可供选择的交换机类型：`direct`, `topic`, `headers`和`fanout`。我们在这里主要说明最后一个 —— `fanout`。先创建一个`fanout`类型的交换机，命名为logs：

```go
err = ch.ExchangeDeclare(
  "logs",   // name
  "fanout", // type
  true,     // durable
  false,    // auto-deleted
  false,    // internal
  false,    // no-wait
  nil,      // arguments
)
```

`fanout`交换机很简单，你可能从名字上就能猜测出来，它把消息发送给它所知道的所有队列。这正是我们的日志系统所需要的。

> #### 交换器列表
> `rabbitmqctl`能够列出服务器上所有的交换器：`sudo rabbitmqctl list_exchanges`
> 这个列表中有一些叫做`amq.*`的匿名交换器。这些都是默认创建的，不过这时候你还不需要使用他们。

> #### 匿名的交换器
> 前面的教程中我们对交换机一无所知，但仍然能够发送消息到队列中。因为我们使用了命名为空字符串("")的匿名交换机。
> 回想我们之前是如何发布一则消息：
>  
	 err = ch.Publish(
      "",     // exchange
      q.Name, // routing key
	  false,  // mandatory
	  false,  // immediate
	  amqp.Publishing{
	    ContentType: "text/plain",
	    Body:        []byte(body),
	})
> exchange参数就是交换机的名称。空字符串代表默认或者匿名交换机，消息将会根据指定的`routing_key`分发到指定的队列。

现在，我们就可以发送消息到一个具名交换机了：

```go
err = ch.ExchangeDeclare(
  "logs",   // name
  "fanout", // type
  true,     // durable
  false,    // auto-deleted
  false,    // internal
  false,    // no-wait
  nil,      // arguments
)
failOnError(err, "Failed to declare an exchange")

body := bodyFrom(os.Args)
err = ch.Publish(
  "logs", // exchange
  "",     // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing{
          ContentType: "text/plain",
          Body:        []byte(body),
  })
```

## 临时队列

你还记得之前我们使用的队列名吗（ hello和task_queue）？给一个队列命名是很重要的——我们需要把工作者（workers）指向正确的队列。如果你打算在发布者（producers）和消费者（consumers）之间共享同队列的话，给队列命名是十分重要的。

但是这并不适用于我们的日志系统。我们打算接收所有的日志消息，而不仅仅是一小部分。我们关心的是最新的消息而不是旧的。为了解决这个问题，我们需要做两件事情。

首先，当我们连接上RabbitMQ的时候，我们需要一个全新的、空的队列。我们可以手动创建一个随机的队列名，或者让服务器为我们选择一个随机的队列名（推荐）。

其次，当与消费者（consumer）断开连接的时候，这个队列应当被立即删除。
在`amqp`客户端中，当我们将队列名称作为空字符串提供时，我们创建一个具有生成名称的非持久队列：

```go
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when usused
  true,  // exclusive
  false, // no-wait
  nil,   // arguments
)
```
该方法返回时，队列实例包含RabbitMQ生成的随机队列名称。例如，它可能看起来像`amq.gen-JzTY20BRgKO-HjmUJj0wLg`。
当声明它的连接关闭时，队列将被删除，因为它被声明为`exclusive`。
您可以在[guide on queues](https://www.rabbitmq.com/queues.html)了解有关`exclusive`和其他队列属性的更多信息

## 绑定（Bindings）

![](http://www.rabbitmq.com/img/tutorials/bindings.png)

我们已经创建了一个`fanout`交换机和一个队列。现在我们需要告诉交换机如何发送消息给我们的队列。交换器和队列之间的联系我们称之为绑定（binding）。

```go
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil,
)
```

现在，logs交换机将会把消息添加到我们的队列中。

> #### 绑定（binding）列表
> 你可以使用`rabbitmqctl list_bindings` 列出所有现存的绑定。

## 代码整合

![](http://www.rabbitmq.com/img/tutorials/python-three-overall.png)

发布日志消息的程序看起来和之前的没有太大区别。最重要的改变就是我们把消息发送给logs交换机而不是匿名交换机。在发送的时候我们需要提供`routing_key`参数，但可以忽略它的值。以下是`emit_log.go`：

```go
package main

import (
        "log"
        "os"
        "strings"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs",   // name
                "fanout", // type
                true,     // durable
                false,    // auto-deleted
                false,    // internal
                false,    // no-wait
                nil,      // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs", // exchange
                "",     // routing key
                false,  // mandatory
                false,  // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[1:], " ")
        }
        return s
}
```

([emit_log.go](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log.go) 源码)

正如你看到的那样，在连接成功之后，我们声明了一个交换器，这一个是很重要的，因为不允许发布消息到不存在的交换器。

如果没有绑定队列到交换器，消息将会丢失。但这个没有所谓，如果没有消费者监听，那么消息就会被忽略。

`receive_logs.go`:的代码：

```go
package main

import (
        "log"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs",   // name
                "fanout", // type
                true,     // durable
                false,    // auto-deleted
                false,    // internal
                false,    // no-wait
                nil,      // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when usused
                true,  // exclusive
                false, // no-wait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        err = ch.QueueBind(
                q.Name, // queue name
                "",     // routing key
                "logs", // exchange
                false,
                nil,
        )
        failOnError(err, "Failed to bind a queue")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        forever := make(chan bool)

        go func() {
                for d := range msgs {
                        log.Printf(" [x] %s", d.Body)
                }
        }()

        log.Printf(" [*] Waiting for logs. To exit press CTRL+C")
        <-forever
}
```

([receive_logs.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs.go) 源码)

这样我们就完成了。如果你想把日志保存到文件中，只需要打开控制台输入：

    $ go run receive_logs.go > logs_from_rabbit.log
    
如果你想在屏幕中查看日志，那么打开一个新的终端然后运行：

    $ go run receive_logs.go

当然还要发送日志：

    $ go run emit_log.go

使用`rabbitmqctl list_bindings`你可确认已经创建的队列绑定。你可以看到运行中的两个`receive_logs.go`程序：

	 sudo rabbitmqctl list_bindings
	# => Listing bindings ...
	# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
	# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
	# => ...done.

显示结果很直观：logs交换器把数据发送给两个系统命名的队列。这就是我们所期望的。

如何监听消息的子集呢？让我们移步教程4
