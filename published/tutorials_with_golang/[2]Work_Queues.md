>原文：[Work Queues](https://www.rabbitmq.com/tutorials/tutorial-two-go.html)  
>状态：待校对  
>翻译：[Bingjian-Zhu](https://bingjian-zhu.github.io/)  
>校对：

![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

## 工作队列

**（使用Go客户端）**

<!--more-->

![](http://www.rabbitmq.com/img/tutorials/python-two.png)  

在[第一篇教程](https://bingjian-zhu.github.io/2019/09/01/RabbitMQ%E6%95%99%E7%A8%8B%EF%BC%88%E8%AF%91%EF%BC%89-Hello-World/)中，我们已经写了一个从已知队列中发送和获取消息的程序。在这篇教程中，我们将创建一个工作队列（Work Queue），它会发送一些耗时的任务给多个工作者（Worker）。

工作队列（又称：任务队列——Task Queues）是为了避免等待一些占用大量资源、时间的操作。当我们把任务（Task）当作消息发送到队列中，一个运行在后台的工作者（worker）进程就会取出任务然后处理。当你运行多个工作者（workers），任务就会在它们之间共享。

这个概念在网络应用中是非常有用的，它可以在短暂的HTTP请求中处理一些复杂的任务。

## 准备

之前的教程中，我们发送了一个包含“Hello World!”的字符串消息。现在，我们将发送一些字符串，把这些字符串当作复杂的任务。我们没有真实的例子，例如图片缩放、pdf文件转换。所以使用time.sleep()函数来模拟这种情况。我们在字符串中加上点号（.）来表示任务的复杂程度，一个点（.）将会耗时1秒钟。比如"Hello..."就会耗时3秒钟。

我们将稍微修改前面示例中的`send.go`代码，以允许从命令行发送任意消息。该程序将发送任务到我们的工作队列，所以我们将其命名为`new_task.go`：

```go
body := bodyFrom(os.Args)
err = ch.Publish(
  "",           // exchange
  q.Name,       // routing key
  false,        // mandatory
  false,
  amqp.Publishing {
    DeliveryMode: amqp.Persistent,
    ContentType:  "text/plain",
    Body:         []byte(body),
  })
failOnError(err, "Failed to publish a message")
log.Printf(" [x] Sent %s", body)
```

我们旧的`receive.go`也需要进行一些更改：它需要为消息体中每一个点号（.）模拟1秒钟的操作。它会从队列中获取消息并执行，我们把它命名为`worker.go`:

```go
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
    log.Printf("Received a message: %s", d.Body)
    dot_count := bytes.Count(d.Body, []byte("."))
    t := time.Duration(dot_count)
    time.Sleep(t * time.Second)
    log.Printf("Done")
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```
请注意，我们的假任务模拟执行时间。
像在教程一中那样运行它们
``` shell 
# shell 1
go run worker.go
```
``` shell 
# shell 2
go run new_task.go
```

## 循环调度:

使用工作队列的一个好处就是它能够并行的处理队列。如果堆积了很多任务，我们只需要添加更多的工作者（workers）就可以了，扩展很简单。

首先，我们先同时运行两个`worker.go`，它们都会从队列中获取消息，到底是不是这样呢？我们看看。

你需要打开三个终端，两个用来运行`worker.go`，这两个终端就是我们的两个消费者（consumers）—— C1 和 C2。

```shell
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```

```shell
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
```
第三个终端，我们用来发布新任务。你可以发送一些消息给消费者（consumers）：

```shell
# shell 3
go run new_task.go First message.
go run new_task.go Second message..
go run new_task.go Third message...
go run new_task.go Fourth message....
go run new_task.go Fifth message.....
```
看看到底发送了什么给我们的工作者（workers）：

```go
# shell 1
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

```go
# shell 2
go run worker.go
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```
默认来说，RabbitMQ会按顺序得把消息发送给每个消费者（consumer）。平均每个消费者都会收到同等数量得消息。这种发送消息得方式叫做——轮询（round-robin）。试着添加三个或更多得工作者（workers）。

## 消息确认

当处理一个比较耗时得任务的时候，你也许想知道消费者（consumers）是否运行到一半就挂掉。当前的代码中，当消息被RabbitMQ发送给消费者（consumers）之后，马上就会在内存中移除。这种情况，你只要把一个工作者（worker）停止，正在处理的消息就会丢失。同时，所有发送到这个工作者的还没有处理的消息都会丢失。

我们不想丢失任何任务消息。如果一个工作者（worker）挂掉了，我们希望任务会重新发送给其他的工作者（worker）。

为了防止消息丢失，RabbitMQ提供了消息响应（acknowledgments）。消费者会通过一个ack（响应），告诉RabbitMQ已经收到并处理了某条消息，然后RabbitMQ就会释放并删除这条消息。

如果消费者（consumer）挂掉了，没有发送响应，RabbitMQ就会认为消息没有被完全处理，然后重新发送给其他消费者（consumer）。这样，即使工作者（workers）偶尔的挂掉，也不会丢失消息。

消息是没有超时这个概念的；当工作者与它断开连的时候，RabbitMQ会重新发送消息。这样在处理一个耗时非常长的消息任务的时候就不会出问题了。

在本教程中，我们将使用手动消息确认，通过为`auto-ack`参数传递false，一旦有任务完成，使用`d.Ack（false）`向RabbitMQ服务器发送消费完成的确认（这个确认消息是单次传递的）。

```go
msgs, err := ch.Consume(
  q.Name, // queue
  "",     // consumer
  false,  // auto-ack
  false,  // exclusive
  false,  // no-local
  false,  // no-wait
  nil,    // args
)
failOnError(err, "Failed to register a consumer")

forever := make(chan bool)

go func() {
  for d := range msgs {
    log.Printf("Received a message: %s", d.Body)
    dot_count := bytes.Count(d.Body, []byte("."))
    t := time.Duration(dot_count)
    time.Sleep(t * time.Second)
    log.Printf("Done")
    d.Ack(false)
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```
运行上面的代码，我们发现即使使用CTRL+C杀掉了一个工作者（worker）进程，消息也不会丢失。当工作者（worker）挂掉这后，所有没有响应的消息都会重新发送。

> #### 忘记确认
> 忘记ack是一个常见的错误。这是一个简单的错误，但后果是严重的。当客户端退出时，消息将被重新传递（这可能看起来像随机重新传递），但是RabbitMQ将会占用越来越多的内存，因为它无法释放任何未经消息的消息
> 为了排除这种错误，你可以使用`rabbitmqctl`命令，输出`messages_unacknowledged`字段：

>       sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged

Windows上执行：

>       rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged


## 消息持久化

如果你没有特意告诉RabbitMQ，那么在它退出或者崩溃的时候，将会丢失所有队列和消息。为了确保信息不会丢失，有两个事情是需要注意的：我们必须把“队列”和“消息”设为持久化。

首先，为了不让队列消失，需要把队列声明为持久化（durable）：
```go
q, err := ch.QueueDeclare(
  "hello",      // name
  true,         // durable
  false,        // delete when unused
  false,        // exclusive
  false,        // no-wait
  nil,          // arguments
)
failOnError(err, "Failed to declare a queue")
```

尽管这行代码本身是正确的，但是达不到我们预期的结果。因为我们已经定义过一个叫`hello`的非持久化队列。RabbitMq不允许你使用不同的参数重新定义一个队列，它会返回一个错误。但我们现在使用一个快捷的解决方法——用不同的名字，例如`task_queue`。

```go
q, err := ch.QueueDeclare(
  "task_queue", // name
  true,         // durable
  false,        // delete when unused
  false,        // exclusive
  false,        // no-wait
  nil,          // arguments
)
failOnError(err, "Failed to declare a queue")
```

这个`durable`必须在生产者（producer）和消费者（consumer）对应的代码中修改。

此时，已经确保即使RabbitMQ重新启动，`task_queue`队列也不会丢失。现在我们需要将消息标记为持久性 - 通过设置`amqp.Publishing`的`amqp.Persistent`属性完成。

```go
err = ch.Publish(
  "",           // exchange
  q.Name,       // routing key
  false,        // mandatory
  false,
  amqp.Publishing {
    DeliveryMode: amqp.Persistent,
    ContentType:  "text/plain",
    Body:         []byte(body),
})
```

> #### 注意：消息持久化
> 将消息设为持久化并不能完全保证不会丢失。以上代码只是告诉了RabbitMq要把消息存到硬盘，但从RabbitMq收到消息到保存之间还是有一个很小的间隔时间。因为RabbitMq并不是所有的消息都使用fsync(2)——它有可能只是保存到缓存中，并不一定会写到硬盘中。并不能保证真正的持久化，但已经足够应付我们的简单工作队列。如果您需要更强的保证，那么您可以使用[publisher confirms.](https://www.rabbitmq.com/confirms.html)。

## 公平调度

你应该已经发现，它仍旧没有按照我们期望的那样进行分发。比如有两个工作者（workers），处理奇数消息的比较繁忙，处理偶数消息的比较轻松。然而RabbitMQ并不知道这些，它仍然一如既往的派发消息。

这时因为RabbitMQ只管分发进入队列的消息，不会关心有多少消费者（consumer）没有作出响应。它盲目的把第n-th条消息发给第n-th个消费者。

![](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)

我们可以设置预取计数值为1。告诉RabbitMQ一次只向一个worker发送一条消息。换句话说，在处理并确认前一个消息之前，不要向工作人员发送新消息。
```go
err = ch.Qos(
  1,     // prefetch count
  0,     // prefetch size
  false, // global
)
failOnError(err, "Failed to set QoS")
```

> #### 关于队列大小
> 如果所有的工作者都处理繁忙状态，你的队列就会被填满。你需要留意这个问题，要么添加更多的工作者（workers），要么使用其他策略。

## 整合代码

`new_task.go`的完整代码：

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

        q, err := ch.QueueDeclare(
                "task_queue", // name
                true,         // durable
                false,        // delete when unused
                false,        // exclusive
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare a queue")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "",           // exchange
                q.Name,       // routing key
                false,        // mandatory
                false,
                amqp.Publishing{
                        DeliveryMode: amqp.Persistent,
                        ContentType:  "text/plain",
                        Body:         []byte(body),
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

([new_task.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/new_task.go)源码)

`worker.go`：

```go
package main

import (
        "bytes"
        "github.com/streadway/amqp"
        "log"
        "time"
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

        q, err := ch.QueueDeclare(
                "task_queue", // name
                true,         // durable
                false,        // delete when unused
                false,        // exclusive
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare a queue")

        err = ch.Qos(
                1,     // prefetch count
                0,     // prefetch size
                false, // global
        )
        failOnError(err, "Failed to set QoS")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                false,  // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        forever := make(chan bool)

        go func() {
                for d := range msgs {
                        log.Printf("Received a message: %s", d.Body)
                        dot_count := bytes.Count(d.Body, []byte("."))
                        t := time.Duration(dot_count)
                        time.Sleep(t * time.Second)
                        log.Printf("Done")
                        d.Ack(false)
                }
        }()

        log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
        <-forever
}
```

([worker.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/worker.go) 源码)

使用消息响应和prefetch_count你就可以搭建起一个工作队列了。这些持久化的选项使得在RabbitMQ重启之后仍然能够恢复。

现在我们可以移步教程3学习如何发送相同的消息给多个消费者（consumers）。
