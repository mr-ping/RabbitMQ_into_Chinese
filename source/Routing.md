>原文：[Routing](https://www.rabbitmq.com/tutorials/tutorial-four-python.html)  
>状态：待校对  
>翻译：[Adam](http://adamlu.net/dev/author/adam/)  
>校对：

## 路由(Routing)

**（使用pika 0.9.5 Python客户端）**

在前面的教程中，我们实现了一个简单的日志系统。可以把日志消息广播给多个接收者。

本篇教程中我们打算新增一个功能——使得它能够只订阅消息的一个字集。例如，我们只需要把严重的错误日志信息写入日志文件（存储到磁盘），但同时仍然把所有的日志信息输出到控制台中

## 绑定（Bindings）

前面的例子，我们已经创建过绑定（bindings），代码如下：

```python
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name)
```

绑定（binding）是指交换器（exchange）和队列（queue）的关系。可以简单理解为：这个队列（queue）对这个交换器（exchange）的消息感兴趣。

绑定的时候可以带上一个额外的routing_key参数。为了避免与basic_publish的参数混淆，我们把它叫做binding key。以下是如何创建一个带binding key的绑定。

```python
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name,
                   routing_key='black')
```

binding key的含义取决于交换器（exchange）的类型。我们之前使用过的fanout类型会忽略这个值。

## Direct类型的交换器（exchange）

我们的日志系统广播所有的消息给所有的消费者（consumers）。我们打算扩展它，使其可以能够精确的过滤消息。例如我们也许值是希望当接收到一个严重的错误的时候才把消息写入磁盘，以免浪费磁盘空间。

我们使用的fanout类型的交换器（exchange）扩展性不够——它能做的仅仅是广播。

我们将会使用direct类型的交换器（exchange）来代替。路由的算法很简单——交换器将会对binding key和routing key进行精确匹配，从而确定消息该分发到哪个队列。

下图能够很好的描述这个场景：

![](http://www.rabbitmq.com/img/tutorials/direct-exchange.png)

在这个场景中，我们可以看到direct exchange X和两个队列绑定了。第一个队列使用orange作为binding key，第二个队列有两个绑定，一个使用black作为binding key，另外一个是green。

这样以来，当routing key为orange的消息发布到交换器（exchange），就会被路由到队列Q1。routing key为black或者green的消息就会路由到Q2。其他的所有消息都将会被丢弃。

## 多个绑定（Multiple bindings）

![](http://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

多个队列使用相同的binding key是合法的。我们的这个例子，我们可以添加一个X和Q1之间的绑定，使用blackbinding key。这样一来，direct交换器就和fanout交换器的行为一样，将会广播消息到所有匹配的队列。带有routing key为black的消息都会发送到Q1和Q2。

## 发送日志

我们将会发送消息到一个direct exchange，把日志级别作为routing key。这样子负责处理接收的脚本就可以选择它要处理的日志级别。我们先看看触发日志。

我们需要创建一个交换器（exchange）：

```python
channel.exchange_declare(exchange='direct_logs',
                         type='direct')
```

然后我们发送一则消息：

```python
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
```

我们先假设“severity”的值是info、warning、error中的一个。

## 订阅（Subscribing）

处理接收消息的方式和之前差不多，但是我们为每一个日志级别创建了一个新的绑定。

```python
result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)
```

整合

![](http://www.rabbitmq.com/img/tutorials/python-four.png)

emit_log_direct.py的代码：

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs',
                         type='direct')

severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
print " [x] Sent %r:%r" % (severity, message)
connection.close()
```

receive_logs_direct.py的代码：

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs',
                         type='direct')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

severities = sys.argv[1:]
if not severities:
    print >> sys.stderr, "Usage: %s [info] [warning] [error]" % \
                         (sys.argv[0],)
    sys.exit(1)

for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)

print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] %r:%r" % (method.routing_key, body,)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
```

如果你希望只是保存warning和error级别的日志到磁盘，只需要打开控制台并输入：

    $ python receive_logs_direct.py warning error > logs_from_rabbit.log

如果你希望所有的日志信息都输出到屏幕中，打开一个新的终端，然后输入：

    $ python receive_logs_direct.py info warning error
     [*] Waiting for logs. To exit press CTRL+C

如果要触发一个error级别的日志，只需要输入：

    $ python emit_log_direct.py error "Run. Run. Or it will explode."
     [x] Sent 'error':'Run. Run. Or it will explode.'

这里是完整的代码：(emit_log_direct.py和receive_logs_direct.py)

教程5展示了如果通过一个模式来监听消息。