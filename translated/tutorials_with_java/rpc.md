# Remote procedure call (RPC)

在[第2篇教程](https://www.rabbitmq.com/tutorials/tutorial-two-java.html)中，我们学习了如何使用工作队列在多个工作程序中分配耗时的任务。

但是，如果我们需要在远程计算机上运行一个函数并等待结果呢？那就另当别论了。这种模式通常称为`远程过程调用或RPC`。

在本教程中，我们将使用RabbitMQ来构建一个RPC系统：一个客户端和一个可扩展的RPC服务器。由于我们没有任何值得分发的耗时任务，我们将创建一个返回斐波那契数的虚拟RPC服务。

## 客户端接口

为了说明如何使用RPC服务，我们将创建一个简单的客户机类。它将暴露一个名为`call`的方法，该方法发送一个RPC请求并阻塞，直到接收到答案：

```java
FibonacciRpcClient fibonacciRpc = new FibonacciRpcClient();
String result = fibonacciRpc.call("4");
System.out.println( "fib(4) is " + result);
```

> ### 关于RPC的说明
>
> 尽管RPC在计算中是一种非常常见的模式，但它经常受到批评。当程序员不知道函数调用是本地的还是缓慢的RPC时，就会出现问题。这样的混淆会导致不可预测的系统，并给调试增加不必要的复杂性。误用RPC不会简化软件，反而会导致不可维护的意大利面条式代码。
>
> 考虑到这一点，请考虑以下建议：
>
> - 确保很明显哪个函数调用是本地的，哪个函数调用是远程的。
> - 文件系统。明确组件之间的依赖关系。
> - 处理错误情况。当RPC服务器长时间关闭时，客户机应该如何反应。
>
> 当有疑问时，避免RPC。如果可以的话，应该使用异步管道——而不是类似rpc的阻塞，结果被异步推到下一个计算阶段。

## 回调队列

一般来说，在RabbitMQ上执行RPC很容易。客户端发送请求消息，服务器用响应消息进行响应。为了接收响应，我们需要发送一个“回调”队列地址与请求。我们可以使用默认队列（它在Java客户机中是独占的）。让我们试一试：

```java
callbackQueueName = channel.queueDeclare().getQueue();

BasicProperties props = new BasicProperties
                            .Builder()
                            .replyTo(callbackQueueName)
                            .build();

channel.basicPublish("", "rpc_queue", props, message.getBytes());

// ... then code to read a response message from the callback_queue ...
```

我们需要这个新的引入：

```java
import com.rabbitmq.client.AMQP.BasicProperties;
```

> ### 消息属性
>
> AMQP 0-9-1协议预定义了一组14个随消息而来的属性。大多数属性很少使用，除了下面的：
>
> - deliveryMode：将消息标记为持久（值为2）或瞬态（任何其他值）。您可能还记得[第2节教程](https://www.rabbitmq.com/tutorials/tutorial-two-java.html)中提到的这个属性。
> - contentType：用于描述编码的mime类型。例如，对于经常使用的JSON编码，最好将此属性设置为：`application/json`。
> - replyTo：通常用于命名回调队列。
> - correlationId：用于将RPC响应与请求关联起来。

## 关联ID

在上述方法中，我们建议为每个RPC请求创建一个回调队列。这是非常低效的，但幸运的是有一种更好的方法——让我们为每个客户机创建一个回调队列。

这引发了一个新问题，在该队列中接收到响应后，不清楚该响应属于哪个请求。这就是使用`correlationId`属性的时候。我们会为每个请求设置一个唯一的值。稍后，当我们在回调队列中接收到消息时，我们将查看此属性，并基于此，我们将能够匹配响应与请求。如果我们看到一个未知的`correlationId`值，我们可以安全地丢弃消息——它不属于我们的请求。

你可能会问，为什么我们要忽略回调队列中的未知消息，而不是因为错误而失败？这是由于服务器端可能存在竞争条件。虽然不太可能，但RPC服务器可能会在向我们发送应答之后，向请求发送确认消息之前死亡。如果发生这种情况，重新启动的RPC服务器将再次处理请求。这就是为什么在客户端我们必须优雅地处理重复的响应，并且RPC在理想情况下应该是幂等的。

## 总结

![img](https://www.rabbitmq.com/img/tutorials/python-six.png)

我们的RPC将像这样工作：

- 对于RPC请求，客户机发送一个带有两个属性的消息：`replyTo`，它被设置为一个只为请求创建的匿名独占队列；`correlationId`，它被设置为每个请求的唯一值。
- 请求被发送到rpc_queue队列。
- RPC 工作程序(又名：server)正在该队列上等待请求。当请求出现时，它执行任务并使用来自`replyTo`字段的队列将结果发送回客户机。
- 客户端在应答队列上等待数据。当消息出现时，它检查`correlationId`属性。如果匹配来自请求的值，它将响应返回给应用程序。

## 把它们放在一起

斐波那契数的任务：

```java
private static int fib(int n) {
    if (n == 0) return 0;
    if (n == 1) return 1;
    return fib(n-1) + fib(n-2);
}
```

我们声明斐波那契函数。它只假设有效的正整数输入。（不要期望这个方法适用于大的数字，它可能是最慢的递归实现）。

我们的RPC服务器的代码可以在这里找到：[RPCServer.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCServer.java)。

服务器代码相当简单：

- 像往常一样，我们首先建立连接、通道和声明队列。


- 我们可能希望运行多个服务器进程。为了将负载平均分配到多个服务器上，我们需要在`channel.basicQos`中设置`prefetchCount`。


- 我们使用`basicConsume`来访问队列，其中我们提供一个对象形式的回调（DeliverCallback），它将完成工作并将响应发送回来。

我们的RPC客户端的代码可以在这里找到：[RPCClient.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCClient.java)。

客户端代码稍微复杂一些：

- 我们建立了联系和渠道。
- 我们的调用方法发出实际的RPC请求。
- 这里，我们首先生成一个唯一的`correlationId`号并保存它——我们的消费者回调将使用这个值来匹配相应的响应。
- 然后，我们为应答创建一个专用的独占队列并订阅它。
- 接下来，我们发布请求消息，使用两个属性：`replyTo`和`correlationId`。
- 在这一点上，我们可以坐下来等待适当的响应。
- 因为我们的消费者交付处理是在一个单独的线程中进行的，所以我们需要一些东西在响应到达之前暂停主线程。使用`BlockingQueue`是一种可能的解决方案。在这里，我们将创建容量设置为1的`ArrayBlockingQueue`，因为我们只需要等待一个响应。
- 消费者正在做一项非常简单的工作，对于每一条被消费的响应消息，它都会检查`correlationId`是否是我们正在寻找的那个。如果是，它将响应放置到`BlockingQueue`。
- 与此同时，主线程正在等待从`BlockingQueue`获取它的响应。
- 最后，我们将响应返回给用户。

现在是查看完整的[RPCClient.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCClient.java)和[RPCServer.java](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/RPCServer.java)示例源代码（包括基本异常处理）的好时机。

像往常一样编译和设置类路径（参见[教程1](https://www.rabbitmq.com/tutorials/tutorial-one-java.html)）：

```bash
javac -cp $CP RPCClient.java RPCServer.java
```

我们的RPC服务现在准备好了。我们可以启动服务器：

```bash
java -cp $CP RPCServer
# => [x] Awaiting RPC requests
```

要请求一个斐波那契数列，运行客户端：

```bash
java -cp $CP RPCClient
# => [x] Requesting fib(30)
```

这里提出的设计不是RPC服务的唯一可能实现，但它有一些重要的优点：

- 如果RPC服务器太慢，您可以通过运行另一个来扩展。尝试在新的控制台中运行第二个`RPCServer`。
- 在客户端，RPC只需要发送和接收一条消息。不需要像`queueDeclare`这样的同步调用。因此，RPC客户机对于单个RPC请求只需要一次网络往返。

我们的代码仍然非常简单，并没有试图解决更复杂（但重要）的问题，如：

- 如果没有服务器在运行，客户机应该如何反应？
- 客户端应该有某种RPC超时吗？
- 如果服务器发生故障并引发异常，是否应该将其转发给客户端？
- 在处理之前防止无效传入消息（如检查边界、类型）。