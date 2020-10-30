>原文：[Routing](https://www.rabbitmq.com/tutorials/tutorial-four-go.html)  
>状态：待校对  
>翻译：[Bingjian-Zhu](https://bingjian-zhu.github.io/)  
>校对：

![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

## 路由(Routing)

**（使用Go客户端）**

<!--more-->

在前面的教程中，我们实现了一个简单的日志系统。可以把日志消息广播给多个接收者。

本篇教程中我们打算新增一个功能 —— 使得它能够只订阅消息的一个字集。例如，我们只需要把严重的错误日志信息写入日志文件（存储到磁盘），但同时仍然把所有的日志信息输出到控制台中

## 绑定（Bindings）

前面的例子，我们已经创建过绑定（bindings），代码如下：

```go
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil)
```

绑定（binding）是指交换机（exchange）和队列（queue）的关系。可以简单理解为：这个队列（queue）对这个交换机（exchange）的消息感兴趣。

绑定的时候可以带上一个额外的`routing_key`参数。为了避免与`Channel.Publish`的参数混淆，我们把它叫做绑定键`binding key`。以下是如何创建一个带绑定键的绑定。

```go
err = ch.QueueBind(
  q.Name,    // queue name
  "black",   // routing key
  "logs",    // exchange
  false,
  nil)
```

绑定键的意义取决于交换机（exchange）的类型。我们之前使用过`fanout` 交换机会忽略这个值。

## 直连交换机（Direct exchange）

我们的日志系统广播所有的消息给所有的消费者（consumers）。我们打算扩展它，使其基于日志的严重程度进行消息过滤。例如我们也许只是希望将比较严重的错误（error）日志写入磁盘，以免在警告（warning）或者信息（info）日志上浪费磁盘空间。

我们使用的`fanout` 交换机没有足够的灵活性 —— 它能做的仅仅是广播。

我们将会使用`direct` 交换机来代替。路由的算法很简单 —— 交换机将会对`binding key`和`routing key`进行精确匹配，从而确定消息该分发到哪个队列。

下图能够很好的描述这个场景：

![](http://www.rabbitmq.com/img/tutorials/direct-exchange.png)

在这个场景中，我们可以看到`direct`交换机 X和两个队列进行了绑定。第一个队列使用`orange`作为binding key，第二个队列有两个绑定，一个使用`black`作为binding key，另外一个使用`green`。

这样以来，当消息发布到routing key为`orange`的交换机时，就会被路由到队列Q1。routing key为`black`或者`green`的消息就会路由到Q2。其他的所有消息都将会被丢弃。

## 多个绑定（Multiple bindings）

![](http://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

多个队列使用相同的binding key是合法的。这个例子中，我们可以添加一个X和Q1之间的绑定，使用`black`为binding key。这样一来，`direct`交换机就和`fanout`交换机的行为一样，会将消息广播到所有匹配的队列。带有routing key为`black`的消息会同时发送到Q1和Q2。

## 发送日志

我们将会发送消息到一个`fanout`，把日志等级作为`routing key`。这样接收日志就可以根据日志级别来选择它想要处理的日志。我们先看看发送日志。

我们需要创建一个交换机（exchange）：

```go
err = ch.ExchangeDeclare(
  "logs_direct", // name
  "direct",      // type
  true,          // durable
  false,         // auto-deleted
  false,         // internal
  false,         // no-wait
  nil,           // arguments
)
```

然后我们发送一则消息：

```go
err = ch.ExchangeDeclare(
  "logs_direct", // name
  "direct",      // type
  true,          // durable
  false,         // auto-deleted
  false,         // internal
  false,         // no-wait
  nil,           // arguments
)
failOnError(err, "Failed to declare an exchange")

body := bodyFrom(os.Args)
err = ch.Publish(
  "logs_direct",         // exchange
  severityFrom(os.Args), // routing key
  false, // mandatory
  false, // immediate
  amqp.Publishing{
    ContentType: "text/plain",
    Body:        []byte(body),
})
```

我们假设日志等级的值是info、warning、error中的一个。

## 订阅

处理接收消息的方式和之前差不多，只有一个例外，我们将会为我们感兴趣的每个严重级别分别创建一个新的绑定。

```go
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
  log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
  os.Exit(0)
}
for _, s := range os.Args[1:] {
  log.Printf("Binding queue %s to exchange %s with routing key %s",
     q.Name, "logs_direct", s)
  err = ch.QueueBind(
    q.Name,        // queue name
    s,             // routing key
    "logs_direct", // exchange
    false,
    nil)
  failOnError(err, "Failed to bind a queue")
}
```

## 代码整合

![](http://www.rabbitmq.com/img/tutorials/python-four.png)

emit_log_direct.go的代码：

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
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs_direct",         // exchange
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
                s = "info"
        } else {
                s = os.Args[1]
        }
        return s
}
```

receive_logs_direct.go的代码：

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
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
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
                log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_direct", s)
                err = ch.QueueBind(
                        q.Name,        // queue name
                        s,             // routing key
                        "logs_direct", // exchange
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

如果你希望只是保存warning和error级别的日志到磁盘，只需要打开控制台并输入：

    $ go run receive_logs_direct.go warning error > logs_from_rabbit.log

如果你希望所有的日志信息都输出到屏幕中，打开一个新的终端，然后输入：

    go run receive_logs_direct.go info warning error
	# => [*] Waiting for logs. To exit press CTRL+C

如果要触发一个error级别的日志，只需要输入：

	go run emit_log_direct.go error "Run. Run. Or it will explode."
	# => [x] Sent 'error':'Run. Run. Or it will explode.'
	
这里是完整的代码：([emit_log_direct.go][1]和[receive_logs_direct.go][2])


[1]:https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/emit_log_direct.go
[2]:https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive_logs_direct.go
