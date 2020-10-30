>原文：[RPC](https://www.rabbitmq.com/tutorials/tutorial-six-go.html)  
>状态：待校对   
>翻译：[Bingjian-Zhu](https://bingjian-zhu.github.io/)   
>校对：

![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

## 远程过程调用（RPC）

**（使用Go客户端）**

<!--more-->

在[第二篇教程](https://bingjian-zhu.github.io/2019/09/01/RabbitMQ%E6%95%99%E7%A8%8B%EF%BC%88%E8%AF%91%EF%BC%89-Work-queues/)中我们介绍了如何使用工作队列（work queue）在多个工作者（woker）中间分发耗时的任务。

可是如果我们需要将一个函数运行在远程计算机上并且等待从那儿获取结果时，该怎么办呢？这就是另外的故事了。这种模式通常被称为远程过程调用（Remote Procedure Call）或者RPC。

这篇教程中，我们会使用RabbitMQ来构建一个RPC系统：包含一个客户端和一个RPC服务器。现在的情况是，我们没有一个值得被分发的足够耗时的任务，所以接下来，我们会创建一个模拟RPC服务来返回斐波那契数列。


> #### 关于RPC的注意事项：
> 尽管RPC在计算领域是一个常用模式，但它也经常被诟病。当一个问题被抛出的时候，程序员往往意识不到这到底是由本地调用还是由较慢的RPC调用引起的。同样的困惑还来自于系统的不可预测性和给调试工作带来的不必要的复杂性。跟软件精简不同的是，滥用RPC会导致不可维护的.
> * 考虑到这一点，牢记以下建议：
>* 确保能够明确的搞清楚哪个函数是本地调用的，哪个函数是远程调用的。给你的系统编写文档。保持各个组件间的依赖明确。处理错误案例。明了客户端该如何处理RPC服务器的宕机和长时间无响应情况。
> * 当对避免使用RPC有疑问的时候。如果可以的话，你应该尽量使用异步管道来代替RPC类的阻塞。结果被异步地推送到下一个计算场景。

### 回调队列

一般来说通过RabbitMQ来实现RPC是很容易的。一个客户端发送请求信息，服务器端将其应用到一个回复信息中。为了接收到回复信息，客户端需要在发送请求的时候同时发送一个回调队列`callback queue`的地址。我们试试看：

```go
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when usused
  true,  // exclusive
  false, // noWait
  nil,   // arguments
)

err = ch.Publish(
  "",          // exchange
  "rpc_queue", // routing key
  false,       // mandatory
  false,       // immediate
  amqp.Publishing{
    ContentType:   "text/plain",
    CorrelationId: corrId,
    ReplyTo:       q.Name,
    Body:          []byte(strconv.Itoa(n)),
})
```

> #### 消息属性
> AMQP协议给消息预定义了一系列的14个属性。大多数属性很少会用到，除了以下几个：
> * `persistent`（持久性）：将消息标记为持久性（值为true）或瞬态（false）。第二篇教程中有说明这个属性。
>* `content_type`（内容类型）:用来描述编码的mime-type。例如在实际使用中常常使用`application/json`来描述JOSN编码类型。
>* `reply_to`（回复目标）：通常用来命名回调队列。
>* `correlation_id`（关联标识）：用来将RPC的响应和请求关联起来。

### 关联标识

上边介绍的方法中，我们建议给每一个RPC请求新建一个回调队列。这不是一个高效的做法，幸好这儿有一个更好的办法 —— 我们可以为每个客户端只建立一个独立的回调队列。

这就带来一个新问题，当此队列接收到一个响应的时候它无法辨别出这个响应是属于哪个请求的。`correlation\_id` 就是为了解决这个问题而来的。我们给每个请求设置一个独一无二的值。稍后，当我们从回调队列中接收到一个消息的时候，我们就可以查看这条属性从而将响应和请求匹配起来。如果我们接手到的消息的`correlation\_id`是未知的，那就直接销毁掉它，因为它不属于我们的任何一条请求。
 
你也许会问，为什么我们接收到未知消息的时候不抛出一个错误，而是要将它忽略掉？这是为了解决服务器端有可能发生的竞争情况。尽管可能性不大，但RPC服务器还是有可能在已将应答发送给我们但还未将确认消息发送给请求的情况下死掉。如果这种情况发生，RPC在重启后会重新处理请求。这就是为什么我们必须在客户端优雅的处理重复响应，同时RPC也需要尽可能保持幂等性。

### 总结

![](http://www.rabbitmq.com/img/tutorials/python-six.png)

我们的RPC如此工作:

* 当客户端启动的时候，它创建一个匿名独享的回调队列。
* 在RPC中，客户端发送带有两个属性的消息：一个是设置回调队列的 *reply\_to* 属性，另一个是设置唯一值的 *correlation\_id* 属性。
* 将请求发送到一个 *rpc\_queue* 队列中。
* RPC工作者（又名：服务器）等待请求发送到这个队列中来。当请求出现的时候，它执行他的工作并且将带有执行结果的消息发送给*reply\_to*字段指定的队列。
* 客户端等待回调队列里的数据。当有消息出现的时候，它会检查*correlation_id*属性。如果此属性的值与请求匹配，将它返回给应用。

## 代码整合

斐波那列函数：
``` go
func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
        }
}
```
声明斐波那契函数，假定只有有效的正整数输入。 （不要指望这个函数能适用于大数字，它可能是最慢的递归实现）。

`rpc_server.go`代码：

```go
package main

import (
        "log"
        "strconv"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
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
                "rpc_queue", // name
                false,       // durable
                false,       // delete when usused
                false,       // exclusive
                false,       // no-wait
                nil,         // arguments
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
                        n, err := strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")

                        log.Printf(" [.] fib(%d)", n)
                        response := fib(n)

                        err = ch.Publish(
                                "",        // exchange
                                d.ReplyTo, // routing key
                                false,     // mandatory
                                false,     // immediate
                                amqp.Publishing{
                                        ContentType:   "text/plain",
                                        CorrelationId: d.CorrelationId,
                                        Body:          []byte(strconv.Itoa(response)),
                                })
                        failOnError(err, "Failed to publish a message")

                        d.Ack(false)
                }
        }()

        log.Printf(" [*] Awaiting RPC requests")
        <-forever
}
```

服务器端代码相当简单：

* 像往常一样，我们建立连接，声明队列
* 为了在多个服务器上平均分配负载，我们希望运行多个服务器进程，需要在通道上设置`prefetch`
* 我们为 basic_consume 声明了一个回调函数，这是RPC服务器端的核心。它执行实际的操作并且作出响应。
* 我们使用`Channel.Consume`来获取接收消息的队列的go频道。然后使用goroutine来处理并作出回应

`rpc_client.go` 代码:

```go
package main

import (
        "log"
        "math/rand"
        "os"
        "strconv"
        "strings"
        "time"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
        }
}

func randomString(l int) string {
        bytes := make([]byte, l)
        for i := 0; i < l; i++ {
                bytes[i] = byte(randInt(65, 90))
        }
        return string(bytes)
}

func randInt(min int, max int) int {
        return min + rand.Intn(max-min)
}

func fibonacciRPC(n int) (res int, err error) {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when usused
                true,  // exclusive
                false, // noWait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

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

        corrId := randomString(32)

        err = ch.Publish(
                "",          // exchange
                "rpc_queue", // routing key
                false,       // mandatory
                false,       // immediate
                amqp.Publishing{
                        ContentType:   "text/plain",
                        CorrelationId: corrId,
                        ReplyTo:       q.Name,
                        Body:          []byte(strconv.Itoa(n)),
                })
        failOnError(err, "Failed to publish a message")

        for d := range msgs {
                if corrId == d.CorrelationId {
                        res, err = strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")
                        break
                }
        }

        return
}

func main() {
        rand.Seed(time.Now().UTC().UnixNano())

        n := bodyFrom(os.Args)

        log.Printf(" [x] Requesting fib(%d)", n)
        res, err := fibonacciRPC(n)
        failOnError(err, "Failed to handle RPC request")

        log.Printf(" [.] Got %d", res)
}

func bodyFrom(args []string) int {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "30"
        } else {
                s = strings.Join(args[1:], " ")
        }
        n, err := strconv.Atoi(s)
        failOnError(err, "Failed to convert arg to integer")
        return n
}
```
（完整的[rpc_client.go][1] 和 [rpc_server.go][2]代码)

[1]:https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_client.go
[2]:https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/rpc_server.go

我们的RPC服务已经准备就绪了，现在启动服务器端：

```go
go run rpc_server.go
# => [x] Awaiting RPC requests
```

运行客户端，请求一个fibonacci队列。

```go
go run rpc_client.go 30
# => [x] Requesting fib(30)
```

此处呈现的设计并不是实现RPC服务的唯一方式，但是他有一些重要的优势：

* 如果RPC服务器运行的过慢的时候，你可以通过运行另外一个服务器端轻松扩展它。试试在控制台中运行第二个 `rpc_server.go` 
* 在客户端，RPC请求只发送或接收一条消息。不需要像 queue_declare 这样的异步调用。所以RPC客户端的单个请求只需要一个网络往返。

我们的代码依旧非常简单，而且没有试图去解决一些复杂（但是重要）的问题，如：

* 当没有服务器运行时，客户端如何作出反映。
* 客户端是否需要实现类似RPC超时的东西。
* 如果服务器发生故障，并且抛出异常，应该被转发到客户端吗？
* 在处理前，防止混入无效的信息（例如检查边界）

如果你想做一些实验，你会发现[management UI ](https://www.rabbitmq.com/management.html)在观测队列方面是很有用处的

