
#AMQP 0-9-1 快速参考指南

本文提供了 AMQP 0-9-1 的 RabbitMQ 实现指南。作为对 [AMQP 规范][1]所定义的类和方法的有力补充，RabbitMQ 还提供了一系列[协议扩展][2]，同样在此列出。原始以及扩展规范可以在[协议页面][3]中进行下载。

为方便大家使用，相关章节内提供了 [Java][4] 和 [.Net][5] 客户端的API指南的链接。每个方法及其参数的完整细节可以从[完整的AMQP 0-9-1 参考][6]中查看。


## Basic

basic.**ack**([**delivery-tag** delivery-tag][7], [**bit** multiple][8])

支持： 完整  
对一条或多条消息进行确认。  

当确认回执(acknowledgement)由客户端发送时，此方法用来确认单条或多条消息已经由 `Deliver` 或 `Get-Ok` 方法成功发送。当确认回执由服务器端发送时，此方用来确认单条或多条消息已经由 `Publish` 方法通过 `confirm` 模式下的信道（channel）成功发布。确认回执可用于单条消息甚至于一个消息集合，并且可以包含一条特定信息。

[javadoc][9] | [dotnetdoc][10] | [amqpdoc][11]

---

basic.**cancel**([**consumer-tag** consumer-tag][12], [**no-wait** no-wait][13]) ➔ **cancel-ok**

支持： 完整  
结束队列消费者  
 
此方法用来清除消费者。它不会影响到已经成功投递的消息，但是会使得服务器不再将新的消息投送给此消费者。客户端会在发送`cancel`方法和收到`cancel-ok`回复的过程中收到任意数量的消息。当消费者端发生不可预估的错误时，此方法也有可能由服务器发送给客户端（也就是说结束行为不是由客户端发送给服务器的basic.cancel方法所触发）。这种情况下客户端可以接收到由于队列被删除等原因引起的消费者丢失通知。需要注意的是，客户端从服务器接收`basic.cancel`方法并不是必须实现的，它通过消息代理可以辨识以协商手段接受`basic.cancel`的客户端的特性来正常工作。

[javadoc][14] | [dotnetdoc][15] | [amqpdoc][16]

---

basic.**consume**([**short** reserved-1][17], [**queue-name** queue][18], [**consumer-tag** consumer-tag][19], [**no-local** no-local][20], [**no-ack** no-ack][21], [**bit** exclusive][22], [**no-wait** no-wait][23], [**table** arguments][24]) ➔ **consume-ok**

支持：部分  
启动队列消费者  

此方法告知服务器开启一个“消费者”，此消费者实质是一个针对特定队列消息的持久化请求。消费者在声明过的信道（channel）中会一直存在，直到客户端清除他们为止。

[javadoc][25] | [dotnetdoc][26] | [amqpdoc][27]

---

basic.**deliver**([**consumer-tag** consumer-tag][28], [**delivery-tag** delivery-tag][29], [**redelivered** redelivered][30], [**exchange-name** exchange][31], [**shortstr** routing-key][32])

支持：完整  
将消费者消息通知给客户端

此方法将一条消息通过消费者投递给客户端。在异步消息投递模式中，客户端通过`Consume`方法启动消费者，然后服务器使用`Deliver`方法将消息送达。

[amqpdoc][33]

---

basic.**get**([**short** reserved-1][34], [**queue-name** queue][35], [**no-ack** no-ack][36]) ➔ **get-ok** | **get-empty**

支持：完整  
直接访问队列  

此方法提供了通过同步通讯的方式直接访问队列内消息的途径。它针对的是一些有特殊需求的应用，例如对应用来说同步的功能性意义远大于应用性能。

[javadoc][37] | [dotnetdoc][38] | [amqpdoc][39]

---

basic.**nack**([**delivery-tag** delivery-tag][40], [**bit** multiple][41], [**bit** requeue][42])

*此方法为RabbitMQ特有的AMQP扩展*

拒绝单条或多条输入消息。

此方法允许客户端拒绝单条或多条输入消息。它可以用来打断或清除大体积消息的输入，或者将无法处理的消息返回给消息的原始队列。这个方法也可以在确认模式（confirm mode）下被服务器用来通知信道（channel）上的消息发布者有未被处理的消息存在。

[RabbitMQ Documentation][43]  
[javadoc][44] | [dotnetdoc][45] | [amqpdoc][46]

---

basic.**publish**([**short** reserved-1][47], [**exchange-name** exchange][48], [**shortstr** routing-key][49], [**bit** mandatory][50], [**bit** immediate][51])

支持：完整  
发布单条消息  

此方法用来发布单条消息到指定的交换机（exchange）。消息将会通过配置好的交换机根据既定规则路由给队列（queues），之后，如果存在事务处理（transaction），并且事务已经被提交，就会分发给活跃的消费者。

[javadoc][52] | [dotnetdoc][53] | [amqpdoc][54]

---

basic.**qos**([**long** prefetch-size][55], [**short** prefetch-count][56], [**bit** global][57]) ➔ **qos-ok**

支持：部分  
指定服务质量

此方法指定服务的服务质量。QoS可以分配给当前的信道（channel）或者链接内的所有信道。qos方法的特定属性和语义依赖于内容类的语义。虽然qos方法原则上可以用于服务端及客户端，但在这里此方法仅适用于服务器端。

[javadoc][58] | [dotnetdoc][59] | [amqpdoc][60]

---

basic.**recover**([**bit** requeue][61])

支持：部分  
重新投递未被确认的消息。  

此方法会要求服务器重新投递特定信道内所有未确认的消息。零条或多条消息会被重新投递。此方法用于替代异步恢复（asynchronous Recover）。

[javadoc][62] | [dotnetdoc][63] | [amqpdoc][64]

---

basic.**recover-async**([**bit** requeue][65])

重新投递未确认的消息。  

此方法会要求服务器重新投递特定信道内所有未确认的消息。零条或多条消息会被重新投递。此方法已经弃用，取而代之的是同步 `Recover/Recover-Ok` 方法。

[javadoc][66] | [dotnetdoc][67] | [amqpdoc][68]

---

basic.**reject**([**delivery-tag** delivery-tag][69], [**bit** requeue][70])

支持：部分  
拒绝单条输入消息。

此方法允许客户端拒绝单条或多条输入消息。它可以用来打断或清除大体积消息的输入，或者将无法处理的消息返回给消息的原始队列。

[RabbitMQ blog post][71]  
[javadoc][72] | [dotnetdoc][73] | [amqpdoc][74]

---

basic.**return**([**reply-code** reply-code][75], [**reply-text** reply-text][76], [**exchange-name** exchange][77], [**shortstr** routing-key][78])

支持： 部分  
返回单条处理失败的消息
  
此方法将发布时打有`"immediate"`标签的无法投递的，或发布时打有`"mandatory"`标签的无法正确路由的单条消息返回。应答代码或文字中会注明失败原因。

[amqpdoc][79]

---

## Channel

channel.**close**([**reply-code** reply-code][80], [**reply-text** reply-text][81], [**class-id** class-id][82], [**method-id** method-id][83]) ➔ **close-ok**

支持：完整   
请求关闭信道。

此方法表明发送者希望关闭信道。这通常是由于内部条件（如强制关闭）或者由于处理特定方法引起的错误（也就是Exception）触发。当关闭行为是由 exception 触发时，发送者需提供引起 exception 的方法的 class id 和 method id。

[javadoc] | [dotnetdoc] | [amqpdoc]

---

channel.**flow**([**bit** active][84]) ➔ **flow-ok**

支持：部分  
启用/禁用对端流

此方法要求对端暂停或者重启消费者发送的内容数据流。这是一个简单的流控制机制，用来避免信道的队列溢出或者发现信道接收的消息是否超出了其处理能力。需要注意的是，此方法目的不在于控制窗口。它不会影响到`Basic.Get-Ok`方法返回的内容。

[amqpdoc]

---

channel.**open**([**shortstr** reserved-1][85]) ➔ **open-ok**

支持：完整  
打开一个信道使用。

此方法会打开一个信道用于与服务器通讯。

[amqpdoc]

---

## Confirm

*此类为RabbitMQ特有的AMQP扩展*

confirm.**select**([**bit** nowait][86]) ➔ **select-ok**

.

此方法用来设置信道以便使用发布者确认回执（acknowledgements）。客户端仅可将此方法用于非事务性信道。

[RabbitMQ Documentation][87]  
[javadoc] | [dotnetdoc] | [amqpdoc]

---

## Exchange

exchange.**bind**([**short** reserved-1][88], [**exchange-name** destination][89], [**exchange-name** source][90], [**shortstr** routing-key][91], [**no-wait** no-wait][92], [**table** arguments][93]) ➔ **bind-ok**

*此方法为RabbitMQ特有的AMQP扩展*

将两个交换机进行绑定。

此方法将一个交换机绑定到另一个交换机上。

[RabbitMQ Documentation][94]  
[RabbitMQ blog post][95]  
[javadoc] | [dotnetdoc] | [amqpdoc]

---

exchange.**declare**([**short** reserved-1][96], [**exchange-name** exchange][97], [**shortstr** type][98], [**bit** passive][99], [**bit** durable][100], [**bit** auto-delete*][101], [**bit** internal*][102], [**no-wait** no-wait][103], [**table** arguments][104]) ➔ **declare-ok**

*RabbitMQ针对AMQP协议的扩展*

支持：完整  
验证交换机是否存在，如有需要创建之。

如果指定交换机不存在，此方法会新建之。如果交换机已经存在，会验证其类型是否正确。

RabbitMQ针对AMQP规范实现了一个扩展，此扩展允许将无法正确路由的消息投递到一个替代交换机中（AE）。替代交换机的特性可以帮助判断客户端何时发布了无法路由的消息，并且能够提供 `"or else"` 路由语义去对某些消息做特殊处理，其他的消息则由通用方法进行处理。

[AE documention][105]  
[javadoc] | [dotnetdoc] | [amqpdoc]

---

exchange.**delete**([**short** reserved-1][106], [**exchange-name** exchange][107], [**bit** if-unused][108], [**no-wait** no-wait][109]) ➔ **delete-ok**

支持：部分  
删除交换机

此方法用于删除交换机。当一个交换机被删除后，与其绑定的所有队列都会被清除。

[javadoc] | [dotnetdoc] | [amqpdoc]

---

exchange.**unbind**([**short** reserved-1][110], [**exchange-name** destination][111], [**exchange-name** source][112], [**shortstr** routing-key][113], [**no-wait** no-wait][114], [**table** arguments][115]) ➔ **unbind-ok**

*此方法为RabbitMQ特有的AMQP扩展*

解除两个交换机之间的绑定关系

此方法用于解除两个交换机之间的绑定关系。

[javadoc] | [dotnetdoc] | [amqpdoc]

---

## Queue

queue.**bind**([**short** reserved-1][116], [**queue-name** queue][117], [**exchange-name** exchange][118], [**shortstr** routing-key][119], [**no-wait** no-wait][120], [**table** arguments][121]) ➔ **bind-ok**

支持：完整  
将队列绑定到交换机

此方法用于绑定队列到交换机。队列绑定到交换机之前不会接收到任何消息。在经典消息模型中，存储转发队列绑定到直连交换机，订阅队列绑定到主题交换机。

[javadoc] | [dotnetdoc] | [amqpdoc]

---

queue.**declare**([**short** reserved-1][122], [**queue-name** queue][123], [**bit** passive][124], [**bit** durable][125], [**bit** exclusive][126], [**bit** auto-delete][127], [**no-wait** no-wait][128], [**table** arguments][129]) ➔ **declare-ok**

支持：完整  
声明队列，如果队列不存在创建之。

此方法用于创建或检查队列。当新建一个队列时，客户端可以指定一系列属性用于控制队列的持久性及其内容，还有队列的分享等级。

RabbitMQ为AMQP规范实现了一些扩展，允许队列创建者控制队列各个方面的行为。

**每个队列的消息生命周期**  
这个扩展决定了一条消息从发布到被服务器丢弃的生存时间。此方法中设置生存时间的参数为 `x-message-ttl`。

**队列的过期时间**   
队列可以在声明时指定租约时限。租约时限指的是如果队列一直未被使用，多久之后服务器会将其自动删除。租约时限由此方法的`x-expires`参数指定。

[x-message-ttl documentation][130]  
[x-expires documentation][131]
[javadoc] | [dotnetdoc] | [amqpdoc]

---

queue.**delete**([**short** reserved-1][132], [**queue-name** queue][133], [**bit** if-unused][134], [**bit** if-empty][135], [**no-wait** no-wait][136]) ➔ **delete-ok**

支持：部分  
删除队列。

此方法用于删除一个队列。如果服务器设置了死信队列（dead-letter queue），当队列被某个删除时，任何依存于此队列的消息都会被发送到死信队列中，队列上的所有消费者都会被清除掉。

[javadoc] | [dotnetdoc] | [amqpdoc]

---

queue.**purge**([**short** reserved-1][137], [**queue-name** queue][138], [**no-wait** no-wait][139]) ➔ **purge-ok**

支持：完整  
清空队列。

此方法会将队列中的所有不处于等待 确认回执（acknowledgment）状态的消息全部移除。

[javadoc] | [dotnetdoc] | [amqpdoc]

---

queue.**unbind**([**short** reserved-1][140], [**queue-name** queue][141], [**exchange-name** exchange][142], [**shortstr** routing-key][143], [**table** arguments][144]) ➔ **unbind-ok**

支持：部分  
解除队列与交换机的绑定。

此方法用于解除队列与交换机的绑定关系。

[javadoc] | [dotnetdoc] | [amqpdoc]

---

## Tx

tx.commit() ➔ commit-ok

支持：完整  
提交当前事务。

此方法用于提交当前事务中所有的消息发布以及确（acknowledgments）认执行动作。

[javadoc] | [dotnetdoc] | [amqpdoc]

---

tx.rollback() ➔ rollback-ok

支持：完整  
终止当前事务。

此方法用于终止当前事务中的所有消息发布以及确认提交操作。回滚动作完成后，一个新的事务随即开始。如果有必要，应该发布一个明确的恢复操作。

[javadoc] | [dotnetdoc] | [amqpdoc]

---

tx.select() ➔ select-ok

支持：完整  
选择标准事务模式。

此方法设置信道使用标准事务模式。客户端在使用提交（Commit）或者（回滚）方法之前，需要至少在信道上使用一次此方法。

[javadoc] | [dotnetdoc] | [amqpdoc]





  [1]: http://www.rabbitmq.com/specification.html
  [2]: http://www.rabbitmq.com/extensions.html
  [3]: http://www.rabbitmq.com/protocol.html
  [4]: http://www.rabbitmq.com/java-client.html
  [5]: http://www.rabbitmq.com/dotnet.html
  [6]: http://www.rabbitmq.com/amqp-0-9-1-reference.html
  [7]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.ack.delivery-tag
  [8]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.ack.multiple
  [9]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicAck%28long,%20boolean%29
  [10]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicAck%28System.UInt64,System.Boolean%29
  [11]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.ack
  [12]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.cancel.consumer-tag
  [13]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.cancel.no-wait
  [14]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicCancel%28java.lang.String%29
  [15]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicCancel%28System.String%29
  [16]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.cancel
  [17]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.consume.reserved-1
  [18]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.consume.queue
  [19]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.consume.consumer-tag
  [20]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.consume.no-local
  [21]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.consume.no-ack
  [22]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.consume.exclusive
  [23]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.consume.no-wait
  [24]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.consume.arguments
  [25]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicConsume%28java.lang.String,%20boolean,%20java.lang.String,%20boolean,%20boolean,%20java.util.Map,%20com.rabbitmq.client.Consumer%29
  [26]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicConsume%28System.String,System.Collections.IDictionary,RabbitMQ.Client.IBasicConsumer%29
  [27]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.consume
  [28]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.deliver.consumer-tag
  [29]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.deliver.delivery-tag
  [30]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.deliver.redelivered
  [31]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.deliver.exchange
  [32]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.deliver.routing-key
  [33]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.deliver
  [34]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.get.reserved-1
  [35]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.get.queue
  [36]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.get.no-ack
  [37]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicGet%28java.lang.String,%20boolean%29
  [38]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicGet%28System.String,System.Boolean%29
  [39]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.get
  [40]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.nack.delivery-tag
  [41]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.nack.multiple
  [42]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.nack.requeue
  [43]: http://www.rabbitmq.com/nack.html
  [44]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicNack%28long,%20boolean,%20boolean%29
  [45]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicNack%28System.UInt64,System.Boolean,System.Boolean%29
  [46]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.nack
  [47]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.publish.reserved-1
  [48]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.publish.exchange
  [49]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.publish.routing-key
  [50]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.publish.mandatory
  [51]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.publish.immediate
  [52]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicPublish%28java.lang.String,%20java.lang.String,%20boolean,%20boolean,%20com.rabbitmq.client.AMQP.BasicProperties,%20byte%5B%5D%29
  [53]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicPublish%28System.String,System.String,System.Boolean,System.Boolean,RabbitMQ.Client.IBasicProperties,System.Byte%5B%5D%29
  [54]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.publish
  [55]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.qos.prefetch-size
  [56]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.qos.prefetch-count
  [57]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.qos.global
  [58]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicQos%28int,%20int,%20boolean%29
  [59]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicQos%28System.UInt32,System.UInt16,System.Boolean%29
  [60]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.qos
  [61]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.recover.requeue
  [62]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicRecover%28boolean%29
  [63]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicRecover%28System.Boolean%29
  [64]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.recover
  [65]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.recover-async.requeue
  [66]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicRecoverAsync%28boolean%29
  [67]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicRecoverAsync%28System.Boolean%29
  [68]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.recover-async
  [69]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.reject.delivery-tag
  [70]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.reject.requeue
  [71]: http://www.rabbitmq.com/blog/2010/08/03/well-ill-let-you-go-basicreject-in-rabbitmq/
  [72]: http://www.rabbitmq.com/releases/rabbitmq-java-client/v3.6.1/rabbitmq-java-client-javadoc-3.6.1/com/rabbitmq/client/Channel.html#basicReject%28long,%20boolean%29
  [73]: http://releases/rabbitmq-dotnet-client/v3.6.1/rabbitmq-dotnet-client-3.6.1-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicReject%28System.UInt64,System.Boolean%29
  [74]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.reject
  [75]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.return.reply-code
  [76]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.return.reply-text
  [77]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.return.exchange
  [78]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.return.routing-key
  [79]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.return
  [80]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#channel.close.reply-code
  [81]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#basic.return.reply-text
  [82]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#channel.close.class-id
  [83]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#channel.close.method-id
  [84]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#channel.flow.activehttp://www.rabbitmq.com/amqp-0-9-1-reference.html#channel.flow.active
  [85]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#channel.open.reserved-1
  [86]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#confirm.select.nowait
  [87]: http://www.rabbitmq.com/confirms.html
  [88]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.bind.reserved-1
  [89]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.bind.destination
  [90]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.bind.source
  [91]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.bind.routing-key
  [92]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.bind.no-wait
  [93]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.bind.arguments
  [94]: http://www.rabbitmq.com/e2e.html
  [95]: http://www.rabbitmq.com/blog/2010/10/19/exchange-to-exchange-bindings/
  [96]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.declare.reserved-1
  [97]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.declare.exchange-0-9-1-reference.html#exchange.declare.reserved-1
  [98]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.declare.type
  [99]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.declare.passive
  [100]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.declare.durable
  [101]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.declare.auto-delete
  [102]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.declare.internalcom/amqp-0-9-1-reference.html#exchange.declare.auto-delete
  [103]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.declare.no-wait
  [104]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.declare.arguments
  [105]: http://www.rabbitmq.com/ae.html
  [106]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.delete.reserved-1
  [107]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.delete.exchange
  [108]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.delete.if-unused
  [109]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.delete.no-wait
  [110]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.unbind.reserved-1http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.unbind.reserved-1
  [111]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.unbind.destination
  [112]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.unbind.source
  [113]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.unbind.routing-key
  [114]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.unbind.no-wait
  [115]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#exchange.unbind.arguments
  [116]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.bind.reserved-1
  [117]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.bind.queue
  [118]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.bind.exchange
  [119]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.bind.routing-keyom/amqp-0-9-1-reference.html#queue.bind.exchange
  [120]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.bind.no-wait
  [121]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.bind.arguments
  [122]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.declare.reserved-1
  [123]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.declare.queue
  [124]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.declare.passive
  [125]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.declare.durable
  [126]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.declare.exclusive
  [127]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.declare.auto-delete
  [128]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.declare.no-wait
  [129]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.declare.arguments
  [130]: http://www.rabbitmq.com/ttl.html#per-queue-message-ttl
  [131]: http://www.rabbitmq.com/ttl.html#queue-ttl
  [132]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.delete.reserved-1
  [133]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.delete.queue
  [134]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.delete.if-unused
  [135]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.delete.if-empty
  [136]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.delete.no-wait
  [137]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.purge.reserved-1
  [138]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.purge.queue
  [139]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.purge.no-wait
  [140]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.unbind.reserved-1
  [141]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.unbind.queue
  [142]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.unbind.exchange
  [143]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.unbind.routing-key
  [144]: http://www.rabbitmq.com/amqp-0-9-1-reference.html#queue.unbind.arguments
