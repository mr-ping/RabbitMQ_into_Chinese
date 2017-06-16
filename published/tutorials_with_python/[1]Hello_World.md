##介绍

RabbitMQ是一个消息代理。它的工作就是接收和转发消息。你可以把它想像成一个邮局：你把信件放入邮箱，邮递员就会把信件投递到你的收件人处。在这个比喻中，RabbitMQ就扮演着邮箱、邮局以及邮递员的角色。

RabbitMQ和邮局的主要区别在于，它处理纸张，而是接收、存储和发送消息（message）这种二进制数据。

下面是RabbitMQ和消息所涉及到的一些术语。

 - 生产(Producing)的意思就是发送。发送消息的程序就是一个生产者(producer)。我们一般用"P"来表示:  
![](http://www.rabbitmq.com/img/tutorials/producer.png)

 - 队列(queue)就是存在于RabbitMQ中邮箱的名称。虽然消息的传输经过了RabbitMQ和你的应用程序，但是它只能被存储于队列当中。实质上队列就是个巨大的消息缓冲区，它的大小只受主机内存和硬盘限制。多个生产者（producers）可以把消息发送给同一个队列，同样，多个消费者（consumers）也能够从同一个队列（queue）中获取数据。队列可以绘制成这样（图上是队列的名称）：  
![](http://www.rabbitmq.com/img/tutorials/queue.png)

 - 在这里，消费（Consuming）和接收(receiving)是同一个意思。一个消费者（consumer）就是一个等待获取消息的程序。我们把它绘制为"C"：  
![](http://www.rabbitmq.com/img/tutorials/consumer.png)

需要指出的是生产者、消费者、代理需不要待在同一个设备上；事实上大多数应用也确实不在会将他们放在一台机器上。

##Hello World!

**（使用pika 0.10.0 Python客户端）**

接下来我们用Python写两个小程序。一个发送单条消息的生产者（producer）和一个接收消息并将其输出的消费者（consumer）。传递的消息是"Hello World"。

下图中，“P”代表生产者，“C”代表消费者，中间的盒子代表为消费者保留的消息缓冲区，也就是我们的队列。

![](http://www.rabbitmq.com/img/tutorials/python-one-overall.png)

生产者（producer）把消息发送到一个名为“hello”的队列中。消费者（consumer）从这个队列中获取消息。

>####RabbitMQ库

>RabbitMQ使用的是AMQP 0.9.1协议。这是一个用于消息传递的开放、通用的协议。针对[不同编程语言](https://www.rabbitmq.com/devtools.html)有大量的RabbitMQ客户端可用。在这个系列教程中，RabbitMQ团队推荐使用[Pika](https://pika.readthedocs.org/en/0.10.0/#)这个Python客户端。大家可以通过[pip](https://pip.pypa.io/en/stable/quickstart/)这个包管理工具进行安装：

###发送

![](http://www.rabbitmq.com/img/tutorials/sending.png)

我们第一个程序`send.py`会发送一个消息到队列中。首先要做的事情就是建立一个到RabbitMQ服务器的连接。

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
```

现在我们已经跟本地机器的代理建立了连接。如果你想连接到其他机器的代理上，需要把代表本地的`localhost`改为指定的名字或IP地址。

接下来，在发送消息之前，我们需要确认服务于消费者的队列已经存在。如果将消息发送给一个不存在的队列，RabbitMQ会将消息丢弃掉。下面我们创建一个名为"hello"的队列用来将消息投递进去。

```python
channel.queue_declare(queue='hello')
```

这时候我们就可以发送消息了，我们第一条消息只包含了Hello World!字符串，我们打算把它发送到hello队列。

在RabbitMQ中，消息是不能直接发送到队列中的，这个过程需要通过交换机（exchange）来进行。但是为了不让细节拖累我们的进度，这里我们只需要知道如何使用由空字符串表示的默认交换机即可。如果你想要详细了解交换机，可以查看我们[教程的第三部分](https://www.rabbitmq.com/tutorials/tutorial-three-python.html)来获取更多细节。默认交换机比较特别，它允许我们指定消息究竟需要投递到哪个具体的队列中，队列名字需要在`routing_key`参数中指定。

```python
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
```

在退出程序之前，我们需要确认网络缓冲已经被刷写、消息已经投递到RabbitMQ。通过安全关闭连接可以做到这一点。

```python
connection.close()
```

> 发送不成功！

> 如果这是你第一次使用RabbitMQ，并且没有看到“Sent”消息出现在屏幕上，你可能会抓耳挠腮不知所以。这也许是因为没有足够的磁盘空间给代理使用所造成的（代理默认需要200MB的空闲空间），所以它才会拒绝接收消息。查看一下代理的日志文件进行确认，如果需要的话也可以减少限制。[配置文件文档](http://www.rabbitmq.com/configure.html#config-items)会告诉你如何更改磁盘空间限制（disk_free_limit）。

###接收

![](https://www.rabbitmq.com/img/tutorials/receiving.png)

我们的第二个程序`receive.py`，将会从队列中获取消息并将其打印到屏幕上。

这次我们还是需要要先连接到RabbitMQ服务器。连接服务器的代码和之前是一样的。

下一步也和之前一样，我们需要确认队列是存在的。我们可以多次使用`queue_declare`命令来创建同一个队列，但是只有一个队列会被真正的创建。

```python
channel.queue_declare(queue='hello')
```

你也许要问: 为什么要重复声明队列呢 —— 我们已经在前面的代码中声明过它了。如果我们确定了队列是已经存在的，那么我们可以不这么做，比如此前预先运行了send.py程序。可是我们并不确定哪个程序会首先运行。这种情况下，在程序中重复将队列重复声明一下是种值得推荐的做法。

>####列出所有队列

>你也许希望查看RabbitMQ中有哪些队列、有多少消息在队列中。此时你可以使用rabbitmqctl工具（使用有权限的用户）：
>```bash
>sudo rabbitmqctl list_queues
>```
>（在Windows中不需要sudo命令）
>```bash
>rabbitmqctl list_queues
>```

从队列中获取消息相对来说稍显复杂。需要为队列定义一个回调（callback）函数。当我们获取到消息的时候，Pika库就会调用此回调函数。这个回调函数会将接收到的消息内容输出到屏幕上。

```python
def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
```

下一步，我们需要告诉RabbitMQ这个回调函数将会从名为"hello"的队列中接收消息：

```python
channel.basic_consume(callback,
                      queue='hello',
                      no_ack=True)
```

要成功运行这些命令，我们必须保证队列是存在的，我们的确可以确保它的存在——因为我们之前已经使用`queue_declare`将其声明过了。

`no_ack`参数[稍后](https://www.rabbitmq.com/tutorials/tutorial-two-python.html)会进行介绍。

最后，我们运行一个用来等待消息数据并且在需要的时候运行回调函数的无限循环。

```python
print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

###将代码整合到一起

**send.py的完整代码：**

```python
#!/usr/bin/env python
import pika

connection =
pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello')

channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
```

([send.py源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/send.py))

**receive.py的完整代码：**

```python
#!/usr/bin/env python
import pika

connection =
pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello')

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)

channel.basic_consume(callback,
                      queue='hello',
                      no_ack=True)

print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

([receive.py源码](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/receive.py))

现在我们可以在终端中尝试一下我们的程序了。  
首先我们启动一个消费者，它会持续的运行来等待投递到达。

```bash
python receive.py
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Hello World!'
```

然后启动生产者，生产者程序每次执行后都会停止运行。

```bash
python send.py
# => [x] Sent 'Hello World!'
```

**成功了！**我们已经通过RabbitMQ发送第一条消息。你也许已经注意到了，receive.py程序并没有退出。它一直在准备获取消息，你可以通过Ctrl-C来中止它。

试下在新的终端中再次运行`send.py`。

我们已经学会如何发送消息到一个已知队列中并接收消息。是时候移步到第二部分了，我们将会建立一个简单的工作队列（work queue）。

>原文：[Hello World](http://www.rabbitmq.com/tutorials/tutorial-one-python.html)  
>Updated at 2017-06-16
