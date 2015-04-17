>原文：[Publish/Subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-python.html)  
>状态：校对完毕  
>翻译：[Adam](http://adamlu.net/dev/author/adam/)  
>校对：[Ping](http://weibo.com/370321376)

## 发布／订阅

**（使用pika 0.9.5 Python客户端）**

在上篇教程中，我们搭建了一个工作队列，每个任务只分发给一个工作者（worker）。在本篇教程中，我们要做的跟之前完全不一样 —— 分发一个消息给多个消费者（consumers）。这种模式被称为“发布／订阅”。

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

有几个可供选择的交换机类型：直连交换机（direct）, 主题交换机（topic）, （头交换机）headers和 扇型交换机（fanout）。我们在这里主要说明最后一个 —— 扇型交换机（fanout）。先创建一个fanout类型的交换机，命名为logs：

```python
channel.exchange_declare(exchange='logs',
                         type='fanout')
```

扇型交换机（fanout）很简单，你可能从名字上就能猜测出来，它把消息发送给它所知道的所有队列。这正是我们的日志系统所需要的。

> #### 交换器列表

> rabbitmqctl能够列出服务器上所有的交换器：

>     $ sudo rabbitmqctl list_exchanges
>     Listing exchanges ...
>     logs      fanout
>     amq.direct      direct
>     amq.topic       topic
>     amq.fanout      fanout
>     amq.headers     headers
>     ...done.

> 这个列表中有一些叫做amq.*的交换器。这些都是默认创建的，不过这时候你还不需要使用他们。

> #### 匿名的交换器

> 前面的教程中我们对交换机一无所知，但仍然能够发送消息到队列中。因为我们使用了命名为空字符串("")默认的交换机。

> 回想我们之前是如何发布一则消息：

>     channel.basic_publish(exchange='',
>                           routing_key='hello',
>                           body=message)

> exchange参数就是交换机的名称。空字符串代表默认或者匿名交换机：消息将会根据指定的routing_key分发到指定的队列。

现在，我们就可以发送消息到一个具名交换机了：

```python
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
```

## 临时队列

你还记得之前我们使用的队列名吗（ hello和task_queue）？给一个队列命名是很重要的——我们需要把工作者（workers）指向正确的队列。如果你打算在发布者（producers）和消费者（consumers）之间共享同队列的话，给队列命名是十分重要的。

但是这并不适用于我们的日志系统。我们打算接收所有的日志消息，而不仅仅是一小部分。我们关心的是最新的消息而不是旧的。为了解决这个问题，我们需要做两件事情。

首先，当我们连接上RabbitMQ的时候，我们需要一个全新的、空的队列。我们可以手动创建一个随机的队列名，或者让服务器为我们选择一个随机的队列名（推荐）。我们只需要在调用queue_declare方法的时候，不提供queue参数就可以了：

    result = channel.queue_declare()

这时候我们可以通过result.method.queue获得已经生成的随机队列名。它可能是这样子的：amq.gen-U0srCoW8TsaXjNh73pnVAw==。

第二步，当与消费者（consumer）断开连接的时候，这个队列应当被立即删除。exclusive标识符即可达到此目的。

    result = channel.queue_declare(exclusive=True)

## 绑定（Bindings）

![](http://www.rabbitmq.com/img/tutorials/bindings.png)

我们已经创建了一个扇型交换机（fanout）和一个队列。现在我们需要告诉交换机如何发送消息给我们的队列。交换器和队列之间的联系我们称之为绑定（binding）。

```python
channel.queue_bind(exchange='logs',
                   queue=result.method.queue)
```

现在，logs交换机将会把消息添加到我们的队列中。

> #### 绑定（binding）列表

> 你可以使用`rabbitmqctl list_bindings` 列出所有现存的绑定。

## 代码整合

![](http://www.rabbitmq.com/img/tutorials/python-three-overall.png)

发布日志消息的程序看起来和之前的没有太大区别。最重要的改变就是我们把消息发送给logs交换机而不是匿名交换机。在发送的时候我们需要提供routing_key参数，但是它的值会被扇型交换机（fanout exchange）忽略。以下是emit_log.py脚本：

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs',
                         type='fanout')

message = ' '.join(sys.argv[1:]) or "info: Hello World!"
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
print " [x] Sent %r" % (message,)
connection.close()
```

([emit_log.py](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/emit_log.py) 源文件)

正如你看到的那样，在连接成功之后，我们声明了一个交换器，这一个是很重要的，因为不允许发布消息到不存在的交换器。

如果没有绑定队列到交换器，消息将会丢失。但这个没有所谓，如果没有消费者监听，那么消息就会被忽略。

receive_logs.py的代码：

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs',
                         type='fanout')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

channel.queue_bind(exchange='logs',
                   queue=queue_name)

print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] %r" % (body,)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
```

([receive_logs.py](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/receive_logs.py) source)

这样我们就完成了。如果你想把日志保存到文件中，只需要打开控制台输入：

    $ python receive_logs.py > logs_from_rabbit.log

如果你想在屏幕中查看日志，那么打开一个新的终端然后运行：

    $ python receive_logs.py

当然还要发送日志：

    $ python emit_log.py

使用`rabbitmqctl list_bindings`你可确认已经创建的队列绑定。你可以看到运行中的两个receive_logs.py程序：

    $ sudo rabbitmqctl list_bindings
    Listing bindings ...
     ...
    logs    amq.gen-TJWkez28YpImbWdRKMa8sg==                []
    logs    amq.gen-x0kymA4yPzAT6BoC/YP+zw==                []
    ...done.

显示结果很直观：logs交换器把数据发送给两个系统命名的队列。这就是我们所期望的。

如何监听消息的子集呢？让我们移步教程4