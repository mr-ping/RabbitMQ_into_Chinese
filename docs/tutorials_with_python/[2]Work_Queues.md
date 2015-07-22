>原文：[Work Queues](https://www.rabbitmq.com/tutorials/tutorial-two-python.html)  
>状态：待校对  
>翻译：[Adam](http://adamlu.net/dev/author/adam/)  
>校对：[Ping](http://weibo.com/370321376)

##工作队列

**（使用pika 0.9.5 Python客户端）**

![](http://www.rabbitmq.com/img/tutorials/python-two.png)  

在第一篇教程中，我们已经写了一个从已知队列中发送和获取消息的程序。在这篇教程中，我们将创建一个工作队列（Work Queue），它会发送一些耗时的任务给多个工作者（Worker）。

工作队列（又称：任务队列——Task Queues）是为了避免等待一些占用大量资源、时间的操作。当我们把任务（Task）当作消息发送到队列中，一个运行在后台的工作者（worker）进程就会取出任务然后处理。当你运行多个工作者（workers），任务就会在它们之间共享。

这个概念在网络应用中是非常有用的，它可以在短暂的HTTP请求中处理一些复杂的任务。

##准备

之前的教程中，我们发送了一个包含“Hello World!”的字符串消息。现在，我们将发送一些字符串，把这些字符串当作复杂的任务。我们没有真实的例子，例如图片缩放、pdf文件转换。所以使用time.sleep()函数来模拟这种情况。我们在字符串中加上点号（.）来表示任务的复杂程度，一个点（.）将会耗时1秒钟。比如"Hello..."就会耗时3秒钟。

我们对之前教程的send.py做些简单的调整，以便可以发送随意的消息。这个程序会按照计划发送任务到我们的工作队列中。我们把它命名为new_task.py：

```python
import sys

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body=message)
print " [x] Sent %r" % (message,)
```

我们的旧脚本（receive.py）同样需要做一些改动：它需要为消息体中每一个点号（.）模拟1秒钟的操作。它会从队列中获取消息并执行，我们把它命名为worker.py：

```python
import time

def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
```

##循环调度:

使用工作队列的一个好处就是它能够并行的处理队列。如果堆积了很多任务，我们只需要添加更多的工作者（workers）就可以了，扩展很简单。

首先，我们先同时运行两个worker.py脚本，它们都会从队列中获取消息，到底是不是这样呢？我们看看。

你需要打开三个终端，两个用来运行worker.py脚本，这两个终端就是我们的两个消费者（consumers）—— C1 和 C2。

```shell
shell1$ python worker.py
 [*] Waiting for messages. To exit press CTRL+C
```

```shell
shell2$ python worker.py
 [*] Waiting for messages. To exit press CTRL+C
```
第三个终端，我们用来发布新任务。你可以发送一些消息给消费者（consumers）：

```shell
shell3$ python new_task.py First message.
shell3$ python new_task.py Second message..
shell3$ python new_task.py Third message...
shell3$ python new_task.py Fourth message....
shell3$ python new_task.py Fifth message.....
```
看看到底发送了什么给我们的工作者（workers）：

```python
shell1$ python worker.py
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'First message.'
 [x] Received 'Third message...'
 [x] Received 'Fifth message.....'
```

```python
shell2$ python worker.py
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'Second message..'
 [x] Received 'Fourth message....'
```
默认来说，RabbitMQ会按顺序得把消息发送给每个消费者（consumer）。平均每个消费者都会收到同等数量得消息。这种发送消息得方式叫做——轮询（round-robin）。试着添加三个或更多得工作者（workers）。

##消息确认

当处理一个比较耗时得任务的时候，你也许想知道消费者（consumers）是否运行到一半就挂掉。当前的代码中，当消息被RabbitMQ发送给消费者（consumers）之后，马上就会在内存中移除。这种情况，你只要把一个工作者（worker）停止，正在处理的消息就会丢失。同时，所有发送到这个工作者的还没有处理的消息都会丢失。

我们不想丢失任何任务消息。如果一个工作者（worker）挂掉了，我们希望任务会重新发送给其他的工作者（worker）。

为了防止消息丢失，RabbitMQ提供了消息响应（acknowledgments）。消费者会通过一个ack（响应），告诉RabbitMQ已经收到并处理了某条消息，然后RabbitMQ就会释放并删除这条消息。

如果消费者（consumer）挂掉了，没有发送响应，RabbitMQ就会认为消息没有被完全处理，然后重新发送给其他消费者（consumer）。这样，及时工作者（workers）偶尔的挂掉，也不会丢失消息。

消息是没有超时这个概念的；当工作者与它断开连的时候，RabbitMQ会重新发送消息。这样在处理一个耗时非常长的消息任务的时候就不会出问题了。

消息响应默认是开启的。之前的例子中我们可以使用no_ack=True标识把它关闭。是时候移除这个标识了，当工作者（worker）完成了任务，就发送一个响应。

```python
def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_consume(callback,
                      queue='hello')
```
运行上面的代码，我们发现即使使用CTRL+C杀掉了一个工作者（worker）进程，消息也不会丢失。当工作者（worker）挂掉这后，所有没有响应的消息都会重新发送。

> #### 忘记确认

> 一个很容易犯的错误就是忘了basic_ack，后果很严重。消息在你的程序退出之后就会重新发送，如果它不能够释放没响应的消息，RabbitMQ就会占用越来越多的内存。

> 为了排除这种错误，你可以使用rabbitmqctl命令，输出messages_unacknowledged字段：

> ```
> $ sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
> Listing queues ...
> hello    0       0
> ...done.
> ```

##消息持久化

如果你没有特意告诉RabbitMQ，那么在它退出或者崩溃的时候，将会丢失所有队列和消息。为了确保信息不会丢失，有两个事情是需要注意的：我们必须把“队列”和“消息”设为持久化。

首先，为了不让队列消失，需要把队列声明为持久化（durable）：

    channel.queue_declare(queue='hello', durable=True)

尽管这行代码本身是正确的，但是仍然不会正确运行。因为我们已经定义过一个叫hello的非持久化队列。RabbitMq不允许你使用不同的参数重新定义一个队列，它会返回一个错误。但我们现在使用一个快捷的解决方法——用不同的名字，例如task_queue。

    channel.queue_declare(queue='task_queue', durable=True)

这个queue_declare必须在生产者（producer）和消费者（consumer）对应的代码中修改。

这时候，我们就可以确保在RabbitMq重启之后queue_declare队列不会丢失。另外，我们需要把我们的消息也要设为持久化——将delivery_mode的属性设为2。

```python
channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      ))
```

> #### 注意：消息持久化

> 将消息设为持久化并不能完全保证不会丢失。以上代码只是告诉了RabbitMq要把消息存到硬盘，但从RabbitMq收到消息到保存之间还是有一个很小的间隔时间。因为RabbitMq并不是所有的消息都使用fsync(2)——它有可能只是保存到缓存中，并不一定会写到硬盘中。并不能保证真正的持久化，但已经足够应付我们的简单工作队列。如果你一定要保证持久化，你需要改写你的代码来支持事务（transaction）。

##公平调度

你应该已经发现，它仍旧没有按照我们期望的那样进行分发。比如有两个工作者（workers），处理奇数消息的比较繁忙，处理偶数消息的比较轻松。然而RabbitMQ并不知道这些，它仍然一如既往的派发消息。

这时因为RabbitMQ只管分发进入队列的消息，不会关心有多少消费者（consumer）没有作出响应。它盲目的把第n-th条消息发给第n-th个消费者。

![](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)

我们可以使用basic.qos方法，并设置prefetch_count=1。这样是告诉RabbitMQ，再同一时刻，不要发送超过1条消息给一个工作者（worker），直到它已经处理了上一条消息并且作出了响应。这样，RabbitMQ就会把消息分发给下一个空闲的工作者（worker）。

    channel.basic_qos(prefetch_count=1)

> #### 关于队列大小

> 如果所有的工作者都处理繁忙状态，你的队列就会被填满。你需要留意这个问题，要么添加更多的工作者（workers），要么使用其他策略。

## 整合代码

new_task.py的完整代码：

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='task_queue',
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      ))
print " [x] Sent %r" % (message,)
connection.close()
```

([new_task.py](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/new_task.py)源码)

我们的worker：

```python
#!/usr/bin/env python
import pika
import time

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)
print ' [*] Waiting for messages. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(callback,
                      queue='task_queue')

channel.start_consuming()
```

([worker.py](http://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/worker.py) source)

使用消息响应和prefetch_count你就可以搭建起一个工作队列了。这些持久化的选项使得在RabbitMQ重启之后仍然能够恢复。

现在我们可以移步教程3学习如何发送相同的消息给多个消费者（consumers）。