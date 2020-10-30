>原文：[Topics](https://www.rabbitmq.com/tutorials/tutorial-five-go.html)  
>状态：待校对  
>翻译：[Bingjian-Zhu](https://bingjian-zhu.github.io/)  
>校对：

![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

## 为什么需要topic交换机？

**（使用Go客户端）**

<!--more-->

[上一篇教程](https://bingjian-zhu.github.io/2019/09/01/RabbitMQ%E6%95%99%E7%A8%8B%EF%BC%88%E8%AF%91%EF%BC%89-Routing/)，我们改进了我们的日志系统。我们使用`direct`交换机替代了`fanout`交换机，从只能盲目的广播消息改进为有可能选择性的接收日志。

尽管`direct`交换机能够改善我们的系统，但是它也有它的限制 —— 没办法基于多个标准执行路由操作。

在我们的日志系统中，我们不只希望订阅基于严重程度的日志，同时还希望订阅基于发送来源的日志。Unix工具[syslog](http://en.wikipedia.org/wiki/Syslog)就是同时基于严重程度-severity (info/warn/crit...) 和 设备-facility (auth/cron/kern...)来路由日志的。

如果这样的话，将会给予我们非常大的灵活性，我们既可以监听来源于“cron”的严重程度为“critical errors”的日志，也可以监听来源于“kern”的所有日志。

为了实现这个目的，接下来我们学习如何使用另一种更复杂的交换机 —— topic交换机。

## topic交换机

发送到`topic`交换机的消息不可以携带随意`routing_key`，它的routing_key必须是一个由`.`分隔开的词语列表。这些单词随便是什么都可以，但是最好是跟携带它们的消息有关系的词汇。以下是几个推荐的例子："stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit"。词语的个数可以随意，但是不要超过255字节。

binding key也必须拥有同样的格式。`topic`交换机背后的逻辑跟`direct`交换机很相似 —— 一个携带着特定routing_key的消息会被topic交换机投递给绑定键与之想匹配的队列。但是它的binding key和routing_key有两个特殊应用方式：

 - `*` (星号) 用来表示一个单词.
 - `#` (井号) 用来表示任意数量（零个或多个）单词。

下边用图说明：  
![None](http://www.rabbitmq.com/img/tutorials/python-five.png)

这个例子里，我们发送的所有消息都是用来描述小动物的。发送的消息所携带的路由键是由三个单词所组成的，这三个单词被两个`.`分割开。路由键里的第一个单词描述的是动物的手脚的利索程度，第二个单词是动物的颜色，第三个是动物的种类。所以它看起来是这样的： `<celerity>.<colour>.<species>`。

我们创建了三个绑定：Q1的绑定键为 `*.orange.*`，Q2的绑定键为 `*.*.rabbit` 和 `lazy.#` 。

这三个绑定键被可以总结为：

 - Q1 对*所有的桔黄色动物*都感兴趣。
 - Q2 则是对*所有的兔子*和*所有懒惰的动物*感兴趣。

一个携带有 `quick.orange.rabbit` 的消息将会被分别投递给这两个队列。携带着 `lazy.orange.elephant` 的消息同样也会给两个队列都投递过去。另一方面携带有 `quick.orange.fox` 的消息会投递给第一个队列，携带有 `lazy.brown.fox` 的消息会投递给第二个队列。携带有 `lazy.pink.rabbit` 的消息只会被投递给第二个队列一次，即使它同时匹配第二个队列的两个绑定。携带着 `quick.brown.fox` 的消息不会投递给任何一个队列。

如果我们违反约定，发送了一个携带有一个单词或者四个单词（`"orange"` or `"quick.orange.male.rabbit"`）的消息时，发送的消息不会投递给任何一个队列，而且会丢失掉。

但是另一方面，即使 `"lazy.orange.male.rabbit"` 有四个单词，他还是会匹配最后一个绑定，并且被投递到第二个队列中。

>#### Topic交换机
>Topic交换机是很强大的，它可以表现出跟其他交换机类似的行为
>当一个队列的binding key为 "#"（井号） 的时候，这个队列将会无视消息的routing key，接收所有的消息。
>当 `*` (星号) 和 `#` (井号) 这两个特殊字符都未在binding key中出现的时候，此时Topic交换机就拥有的direct交换机的行为。

## 代码整合

接下来我们会将Topic交换机应用到我们的日志系统中。在开始工作前，我们假设日志的routing key由两个单词组成，routing key看起来是这样的：`<facility>.<severity>`

代码跟[上一篇教程](https://bingjian-zhu.github.io/2019/09/01/RabbitMQ%E6%95%99%E7%A8%8B%EF%BC%88%E8%AF%91%EF%BC%89-Routing/)差不多。

emit_log_topic.go的代码：

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
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs_topic",          // exchange
                severityFrom(os.Args), // routing key
                false, // mandatory
                false, // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 3) || os.Args[2] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[2:], " ")
        }
        return s
}

func severityFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "anonymous.info"
        } else {
                s = os.Args[1]
        }
        return s
}
```

receive_logs_topic.go的代码：

```go
package main

import (
        "log"
        "os"

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
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
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

        if len(os.Args) < 2 {
                log.Printf("Usage: %s [binding_key]...", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_topic", s)
                err = ch.QueueBind(
                        q.Name,       // queue name
                        s,            // routing key
                        "logs_topic", // exchange
                        false,
                        nil)
                failOnError(err, "Failed to bind a queue")
        }

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto ack
                false,  // exclusive
                false,  // no local
                false,  // no wait
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

执行下边命令 接收所有日志：  

	go run receive_logs_topic.go "#"

执行下边命令 接收来自”kern“设备的日志：  

	go run receive_logs_topic.go "kern.*"

执行下边命令 只接收严重程度为”critical“的日志：  

	go run receive_logs_topic.go "*.critical"

执行下边命令 建立多个绑定：  

	go run receive_logs_topic.go "kern.*" "*.critical"

执行下边命令 发送路由键为 "kern.critical" 的日志：  

	go run emit_log_topic.go "kern.critical" "A critical kernel error"

执行上边命令试试看效果吧。另外，上边代码不会对路由键和绑定键做任何假设，所以你可以在命令中使用超过两个路由键参数。

### 如果你现在还没被搞晕，想想下边问题:
 - 绑定键为 `*` 的队列会取到一个routing key为空的消息吗？  
 - 绑定键为 `#.*` 的队列会获取到一个名为`..`的路由键的消息吗？它会取到一个routing key为单个单词的消息吗？  
 - `a.*.#` 和 `a.#`的区别在哪儿？

（完整代码参见[emit_logs_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_topic.go) and [receive_logs_topic.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs_topic.go))
