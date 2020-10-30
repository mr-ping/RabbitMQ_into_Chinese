>原文：[Hello_World](https://www.rabbitmq.com/tutorials/tutorial-one-go.html)  
>状态：待校对  
>翻译：[Bingjian-Zhu](https://bingjian-zhu.github.io/)  
>校对：

![CC-BY-SA](https://upload.wikimedia.org/wikipedia/commons/d/d0/CC-BY-SA_icon.svg)

## 前提条件
> 本教程假设RabbitMQ已经安装在你本机的 (5672)端口。如果你使用了不同的主机、端口或者凭证，连接设置就需要作出一些对应的调整。

<!--more-->

## 介绍
RabbitMQ是一个消息代理。它的工作就是接收和转发消息。你可以把它想像成一个邮局：你把信件放入邮箱，邮递员就会把信件投递到你的收件人处。在这个比喻中，RabbitMQ就扮演着邮箱、邮局以及邮递员的角色。

RabbitMQ和邮局的主要区别在于，它处理纸张，而是接收、存储和发送消息（message）这种二进制数据。

下面是RabbitMQ和消息所涉及到的一些术语。

* 生产(Producing)的意思就是发送。发送消息的程序就是一个生产者(producer)。我们一般用"P"来表示:
  ![img](http://www.rabbitmq.com/img/tutorials/producer.png)
* 队列(queue)就是存在于RabbitMQ中邮箱的名称。虽然消息的传输经过了RabbitMQ和你的应用程序，但是它只能被存储于队列当中。实质上队列就是个巨大的消息缓冲区，它的大小只受主机内存和硬盘限制。多个生产者（producers）可以把消息发送给同一个队列，同样，多个消费者（consumers）也能够从同一个队列（queue）中获取数据。队列可以绘制成这样（图上是队列的名称）：
  ![img](http://www.rabbitmq.com/img/tutorials/queue.png)
* 在这里，消费（Consuming）和接收(receiving)是同一个意思。一个消费者（consumer）就是一个等待获取消息的程序。我们把它绘制为"C"：
  ![img](http://www.rabbitmq.com/img/tutorials/consumer.png)


## Hello World!

 **（使用Go客户端）**

在本教程的这一部分中，我们将在Go中编写两个小程序：发送单个消息的生产者，以及接收消息并将其打印出来的消费者。我们将忽略Go RabbitMQ API中的一些细节，这里传递“Hello World”消息。
在下图中，“P”是生产者，“C”是消费者。中间的框是一个队列（保存消息的地方）。
![(P) -> [|||] -> (C)](http://www.rabbitmq.com/img/tutorials/python-one.png)


>**Go RabbitMQ客户端库**
> RabbitMQ使用的是AMQP 0.9.1协议。这是一个用于消息传递的开放、通用的协议。针对不同编程语言有大量的RabbitMQ客户端可用。在本教程中，我们将使用Go amqp客户端。

>首先，使用`go get`安装amqp：`go get github.com/streadway/amqp`

### 发送
![(P) -> [|||]](http://www.rabbitmq.com/img/tutorials/sending.png)

我们将编写消息生产者（发送者）`send.go`和我们的消息消费者（接收者）`receive.go`。
发布者将连接到RabbitMQ，发送单个消息，然后退出。
在`send.go`中，我们需要先导入包：
```go
package main

import (
  "log"

  "github.com/streadway/amqp"
)
```
我们还需要一个辅助函数来检查每个amqp调用的返回值
```go
func failOnError(err error, msg string) {
  if err != nil {
    log.Fatalf("%s: %s", msg, err)
  }
}
```
然后连接到RabbitMQ服务器
```go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()
```
配置连接套接字，它主要定义连接的协议和身份验证等。接下来，我们创建一个channel来传递消息：
```go
ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()
```
发送前，我们必须声明一个队列供我们发送，然后才能向队列发送消息：
```go
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")

body := "Hello World!"
err = ch.Publish(
  "",     // exchange
  q.Name, // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing {
    ContentType: "text/plain",
    Body:        []byte(body),
  })
failOnError(err, "Failed to publish a message")
```
声明队列是幂等的——只有在它不存在的情况下才会创建它。消息内容是一个字节数组，因此你可以编写任何内容。
![[|||] -> (C)](http://www.rabbitmq.com/img/tutorials/receiving.png)

### 接收
以上是消息生产者。我们的消费者需要监听来自RabbitMQ的消息，因此与生产者不同，它需要持续运行以监听消息并将其打印出来
![](http://pwzr2sh3s.bkt.clouddn.com/RabbitMQ/Hello%20World/6.png)
代码`receive.go`具有与send相同的导入包和辅助函数：
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
```
设置与生产者相同，首先打开一个连接和一个Channel，并声明我们要消费的队列。请注意，这与发送的队列相匹配
```go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()

ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()

q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when usused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")
```
注意，在这里也做了队列声明。因为消费者可能在生产者启动前就运行了，所以要确保使用消息之前队列已经存在。
我们即将告诉服务器从队列中传递消息。因为它会异步地向我们发送消息，所以我们将在goroutine中读取来自channel （由`amqp :: Consume`返回）的消息。
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
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```
[receive.go](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/go/receive.go)

### 代码整合
运行生产者：`go run send.go`
运行消费者：`go run receive.go`
消费者将通过RabbitMQ打印从生产者处获得的消息。消费者将继续运行，等待消息（使用Ctrl-C停止消息）。可以尝试从另一个终端再次运行生产者来发送消息。

我们已经学会如何发送消息到一个已知队列中并接收消息。是时候移步到第二部分了，我们将会建立一个简单的工作队列（work queue）。
