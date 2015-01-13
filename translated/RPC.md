>原文：[Remote procedure call (RPC)](https://www.rabbitmq.com/tutorials/tutorial-six-python.html)  
>状态：翻译完成
>翻译：Ping
>校对：

## Remote procedure call (RPC)
## 远程过程调用（RPC）
#### (using the pika 0.9.8 Python client)
#### （Python客户端 —— 使用 pika 0.9.8）

In the [second tutorial](http://www.rabbitmq.com/tutorials/tutorial-two-python.html) we learned how to use Work Queues to distribute time-consuming tasks among multiple workers.  
在[第二篇教程][4]中我们介绍了如何使用工作队列（work queue）在多个工作者（woker）中分发耗时的任务。

But what if we need to run a function on a remote computer and wait for the result? Well, that's a different story. This pattern is commonly known as Remote Procedure Call or RPC.  
可是如果我们需要将一个功能运行在远程计算机上并且等待从那儿获取结果时，该怎么办呢？这种模式通常被成为远程过程调用（Remote Procedure Call）或者RPC。

In this tutorial we're going to use RabbitMQ to build an RPC system: a client and a scalable RPC server. As we don't have any time-consuming tasks that are worth distributing, we're going to create a dummy RPC service that returns Fibonacci numbers.    
这篇教程中，我们会使用RabbitMQ来构建一个RPC系统：包含一个客户端和一个RPC服务器。现在的情况是，我们没有一个值得被分发的足够耗时的任务，所以接下来，我们会创建一个模拟RPC服务来返回斐波那契数列。

### Client interface
### 客户端接口

To illustrate how an RPC service could be used we're going to create a simple client class. It's going to expose a method named call which sends an RPC request and blocks until the answer is received:  
为了展示RPC服务如何使用，我们创建了一个简单的客户端类。它会暴露出一个名为“call”的方法用来发送一个RPC请求，并且在收到回应前保持阻塞。

```python
fibonacci_rpc = FibonacciRpcClient()
result = fibonacci_rpc.call(4)
print "fib(4) is %r" % (result,)
```

> A note on RPC
> RPC的注意事项：

> Although RPC is a pretty common pattern in computing, it's often criticised. The problems arise when a programmer is not aware whether a function call is local or if it's a slow RPC. Confusions like that result in an unpredictable system and adds unnecessary complexity to debugging. Instead of simplifying software, misused RPC can result in unmaintainable spaghetti code.  
> 尽管RPC在计算上是一个常用模式，但它也经常被诟病。当一个问题被抛出的时候，程序员往往意识不到这到底是由本地调用还是由较慢的RPC调用引起的。同样的困惑还来自于系统的不可预测性和给调试工作带来的不必要的复杂性。跟软件精简不同的是，滥用RPC会导致不可维护的[面条代码][5].

> Bearing that in mind, consider the following advice:  
> 考虑到这一点，牢记以下建议：

> Make sure it's obvious which function call is local and which is remote.
Document your system. Make the dependencies between components clear.
Handle error cases. How should the client react when the RPC server is down for a long time?  
确保能够明确的搞清楚哪个函数是本地调用的，哪个函数是远程调用的。给你的系统编写文档。保持各个组件间的依赖明确。处理错误案例。明了客户端改如何处理RPC服务器的宕机和长时间无响应情况。

> When in doubt avoid RPC. If you can, you should use an asynchronous pipeline - instead of RPC-like blocking, results are asynchronously pushed to a next computation stage.  
> 当对避免RPC有疑问的时候。如果可以，你应该使用异步管道来代替RPC类的阻塞。结果被异步地推送到下一个计算场景。

### Callback queue
### 回调队列

In general doing RPC over RabbitMQ is easy. A client sends a request message and a server replies with a response message. In order to receive a response the client needs to send a 'callback' queue address with the request. Let's try it:  
一般来说通过RabbitMQ来实现RPC是很容易的。一个客户端发送请求信息，服务器端将其应用到一个回复信息中。为了接收到回复信息，客户端需要在发送请求的时候同时发送一个回调队列（callback queue）的地址。我们试试看：

```python
result = channel.queue_declare(exclusive=True)
callback_queue = result.method.queue

channel.basic_publish(exchange='',
                      routing_key='rpc_queue',
                      properties=pika.BasicProperties(
                            reply_to = callback_queue,
                            ),
                      body=request)

# ... and some code to read a response message from the callback_queue ...
```

> Message properties  
> 消息属性

> The AMQP protocol predefines a set of 14 properties that go with a message. Most of the properties are rarely used, with the exception of the following:  
> AMQP协议给消息预定义了一系列的14个属性。大多数属性很少会用到，除了以下几个：

> * delivery_mode: Marks a message as persistent (with a value of 2) or transient (any other value). You may remember this property from the second tutorial.
* delivery_mode（投递模式）：将消息标记为持久的（值为2）或暂存的（除了2之外的其他任何值）。第二篇教程里接触过这个属性，记得吧？
* content_type（内容类型）: Used to describe the mime-type of the encoding. For example for the often used JSON encoding it is a good practice to set this property to: application/json.
* content_type（内容类型）:用来描述编码的mime-type。例如在实际使用中常常使用application/json来描述JOSN编码类型。
* reply_to: Commonly used to name a callback queue.
* reply_to（回复）：通常用来命名回调队列。
* correlation_id: Useful to correlate RPC responses with requests.
* correlation_id（关联标识）：用来将RPC的响应和请求关联起来。

### Correlation id
### 关联标识

In the method presented above we suggest creating a callback queue for every RPC request. That's pretty inefficient, but fortunately there is a better way - let's create a single callback queue per client.  
上边介绍的方法中，我们建议给每一个RPC请求新建一个回调队列。这不是一个高效的做法，幸好这儿有一个更好的办法 —— 我们可以为每个客户端只建立一个独立的回调队列。

That raises a new issue, having received a response in that queue it's not clear to which request the response belongs. That's when the correlation_id property is used. We're going to set it to a unique value for every request. Later, when we receive a message in the callback queue we'll look at this property, and based on that we'll be able to match a response with a request. If we see an unknown correlation_id value, we may safely discard the message - it doesn't belong to our requests.  
这就带来一个新问题，当队列接收到一个响应的时候它无法辨别出这个响应是属于哪个请求的。correlation_id 就是为了解决这个问题而来的。我们给每个请求设置一个独一无二的值。稍后，当我们从回调队列中接收到一个消息的时候，我们就可以查看这条属性从而将响应和请求匹配起来。如果我们接手到的消息的correlation_id是未知的，那就直接销毁掉它，因为它不属于我们的任何一条请求。

You may ask, why should we ignore unknown messages in the callback queue, rather than failing with an error? It's due to a possibility of a race condition on the server side. Although unlikely, it is possible that the RPC server will die just after sending us the answer, but before sending an acknowledgment message for the request. If that happens, the restarted RPC server will process the request again. That's why on the client we must handle the duplicate responses gracefully, and the RPC should ideally be idempotent.  
你也许会问，为什么我们接收到未知消息的时候不抛出一个错误，而是要将它忽略掉？那是为了解决服务器端有可能发生的竞争情况。尽管可能性不大，但RPC服务器还是有可能在已将应答发送给我们但还未将确认消息发送给请求的情况下死掉。如果这种情况发生，RPC在重启后会重新处理请求。这就是为什么我们必须在客户端优雅的处理重复响应，同时RPC也需要尽可能保持幂等性。

### Summary
### 总结

![](http://www.rabbitmq.com/img/tutorials/python-six.png)

Our RPC will work like this:  
我们的RPC如此工作:

* When the Client starts up, it creates an anonymous exclusive callback queue.
* 当客户端启动的时候，它创建一个匿名独享的回调队列。
* For an RPC request, the Client sends a message with two properties: reply_to, which is set to the callback queue and correlation_id, which is set to a unique value for every request.
* 在RPC请求中，客户端发送带有两个属性的消息：一个是设置回调队列的 *reply_to* 属性，另一个是设置唯一值的 *correlation_id* 属性。
* The request is sent to an rpc_queue queue.
* 将请求发送到一个 *rpc_queue* 队列中。
* The RPC worker (aka: server) is waiting for requests on that queue. When a request appears, it does the job and sends a message with the result back to the Client, using the queue from the reply_to field.
* RPC工作者（又名：服务器）等待请求发送到这个队列中来。当请求出现的时候，它执行他的工作并且将带有执行结果的消息发送给reply_to字段指定的队列。
* The client waits for data on the callback queue. When a message appears, it checks the correlation_id property. If it matches the value from the request it returns the response to the application.
* 客户端等待回调队列里的数据。当有消息出现的时候，它会检查correlation_id属性。如果此属性的值与请求匹配，将它返回给应用。

## Putting it all together
## 整合到一起

The code for rpc_server.py:  
rpc_server.py代码：

```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))

channel = connection.channel()

channel.queue_declare(queue='rpc_queue')

def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n-1) + fib(n-2)

def on_request(ch, method, props, body):
    n = int(body)

    print " [.] fib(%s)"  % (n,)
    response = fib(n)

    ch.basic_publish(exchange='',
                     routing_key=props.reply_to,
                     properties=pika.BasicProperties(correlation_id = \
                                                     props.correlation_id),
                     body=str(response))
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(on_request, queue='rpc_queue')

print " [x] Awaiting RPC requests"
channel.start_consuming()
```

The server code is rather straightforward:
服务器端代码相当简单：

* (4) As usual we start by establishing the connection and declaring the queue.
* （4）像往常一样，我们建立连接，声明队列
* (11) We declare our fibonacci function. It assumes only valid positive integer input. (Don't expect this one to work for big numbers, it's probably the slowest recursive implementation possible).
* （11）我们声明我们的fibonacci函数，它假设只有合法的正整数当作输入。（别指望这个函数能处理很大的数值，函数递归你们都懂得...）
* (19) We declare a callback for basic_consume, the core of the RPC server. It's executed when the request is received. It does the work and sends the response back.
* （19）我们为 basic_consume 声明了一个回调，这是RPC服务器端的核心。它执行实际的操作并且作出响应。
* (32) We might want to run more than one server process. In order to spread the load equally over multiple servers we need to set the prefetch_count setting.
* （32）或许我们希望能在服务器上多开几个线程。为了能将负载平均地分摊到多个服务器，我们需要将 prefetch_count 设置好。

The code for rpc_client.py:
rpc_client.py 代码:

```python
#!/usr/bin/env python
import pika
import uuid

class FibonacciRpcClient(object):
    def __init__(self):
        self.connection = pika.BlockingConnection(pika.ConnectionParameters(
                host='localhost'))

        self.channel = self.connection.channel()

        result = self.channel.queue_declare(exclusive=True)
        self.callback_queue = result.method.queue

        self.channel.basic_consume(self.on_response, no_ack=True,
                                   queue=self.callback_queue)

    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = body

    def call(self, n):
        self.response = None
        self.corr_id = str(uuid.uuid4())
        self.channel.basic_publish(exchange='',
                                   routing_key='rpc_queue',
                                   properties=pika.BasicProperties(
                                         reply_to = self.callback_queue,
                                         correlation_id = self.corr_id,
                                         ),
                                   body=str(n))
        while self.response is None:
            self.connection.process_data_events()
        return int(self.response)

fibonacci_rpc = FibonacciRpcClient()

print " [x] Requesting fib(30)"
response = fibonacci_rpc.call(30)
print " [.] Got %r" % (response,)
```

The client code is slightly more involved:
客户端代码稍微有点难懂：

* (7) We establish a connection, channel and declare an exclusive 'callback' queue for replies.
* （7）建立连接、通道并且为回复（replies）声明独享的回调队列。
* (16) We subscribe to the 'callback' queue, so that we can receive RPC responses.
* （16）我们订阅这个回调队列，以便接收RPC的响应。
* (18) The 'on_response' callback executed on every response is doing a very simple job, for every response message it checks if the correlation_id is the one we're looking for. If so, it saves the response in self.response and breaks the consuming loop.
* （18）“on_response”回调函数对每一个响应执行一个非常简单的操作，检查每一个响应消息的correlation_id属性是否与我们期待的一致，如果一致，将响应结果赋给self.response，然后跳出consuming循环。
* (23) Next, we define our main call method - it does the actual RPC request.
* （23）接下来，我们定义我们的主要方法 call 方法。它执行真正的RPC请求。
* (24) In this method, first we generate a unique correlation_id number and save it - the 'on_response' callback function will use this value to catch the appropriate response.
* （24）在这个方法中，首先我们生成一个唯一的correlation_id值并且保存起来，'on_response'回调函数会用它来获取符合要求的响应。
* (25) Next, we publish the request message, with two properties: reply_to and correlation_id.
* （25）接下来，我们将带有 reply_to 和 correlation_id 属性的消息发布出去。
* (32) At this point we can sit back and wait until the proper response arrives.
* （32）现在我们可以坐下来，等待正确的响应到来。
* (33) And finally we return the response back to the user.
* （33）最后，我们将响应返回给用户。

Our RPC service is now ready. We can start the server:
我们的RPC服务已经准备就绪了，现在启动服务器端：

```python
$ python rpc_server.py
 [x] Awaiting RPC requests
```

To request a fibonacci number run the client:
运行客户端，请求一个fibonacci队列。

```python
$ python rpc_client.py
 [x] Requesting fib(30)
```

The presented design is not the only possible implementation of a RPC service, but it has some important advantages:

此处呈现的设计并不是实现RPC服务的唯一方式，但是他有一些重要的优势：

* If the RPC server is too slow, you can scale up by just running another one. Try running a second rpc_server.py in a new console.
* 如果RPC服务器运行的过慢的时候，你可以通过运行另外一个服务器端轻松扩展它。试试在控制台中运行第二个 rpc_server.py 。
* On the client side, the RPC requires sending and receiving only one message. No synchronous calls like queue_declare are required. As a result the RPC client needs only one network round trip for a single RPC request.
* 在客户端，RPC请求只发送或接收一条消息。像 queue_declare 非同步调用是必须的。所以RPC客户端的单个请求只需要一个网络往返。

Our code is still pretty simplistic and doesn't try to solve more complex (but important) problems, like:
我们的代码依旧非常简单，而且没有试图去解决一些复杂（但是重要）的问题，如：

* How should the client react if there are no servers running?
* 当没有服务器运行时，客户端如何作出反映。
* Should a client have some kind of timeout for the RPC?
* 客户端是否需要实现类似RPC超时的东西。
* If the server malfunctions and raises an exception, should it be forwarded to the client?
* 如果服务器发生故障，并且抛出异常，应该被转发到客户端吗？
* Protecting against invalid incoming messages (eg checking bounds) before processing.
* 在处理前，防止混入无效的信息（例如检查边界）

> If you want to experiment, you may find the [rabbitmq-management plugin][0] useful for viewing the queues.  
> 如果你想实验一下，你会发现[rabbitmq-management plugin][0]在观测队列方面很有用处。

(Full source code for [rpc_client.py][1] and [rpc_server.py][2])  
（完整的[rpc_client.py][1] 和 [rpc_server.py][2]代码)


[0]:http://www.rabbitmq.com/plugins.html
[1]:https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/rpc_client.py
[2]:https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/python/rpc_server.py
[4]:http://www.rabbitmq.com/tutorials/tutorial-two-python.html
[5]:http://zh.wikipedia.org/wiki/%E9%9D%A2%E6%9D%A1%E5%BC%8F%E4%BB%A3%E7%A0%81