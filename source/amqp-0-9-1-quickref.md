#AMQP 0-9-1 Quick Reference

#AMQP 0-9-1 简要参考

This page provides a guide to RabbitMQ's implementation of AMQP 0-9-1. In addition to the classes and methods defined in the AMQP specification, RabbitMQ supports several protocol extensions, 
which are also listed here. The original and extended specification downloads can be found on the protocol page.

本文提供了RabbitMQ对AMQP 0-9-1实现的指南。作为对AMQP规范所定义的类和方法的有力补充，RabbitMQ还提供了一系列扩展协议，同样在此列出。原始以及扩展规范可以在[协议页面][3]中下载。

For your convenience, links are provided from this guide to the relevant sections of the API guides for the RabbitMQ Java and .NET clients. Full details of each method and its parameters are available in our complete AMQP 0-9-1 reference.

为方便大家使用，相关章节内提供了Java和.Net客户端的的API指南的链接。每个方法及其参数的完整细节可以从[完整的AMQP 0-9-1 参考][6]中查看。


## Basic

`basic.ack(delivery-tag delivery-tag, bit multiple)`

Support: full  
支持： 完整  
Acknowledge one or more messages.  
确认一条或多条消息。  

When sent by the client, this method acknowledges one or more messages delivered via the Deliver or Get-Ok methods. When sent by server, this method acknowledges one or more messages published with the Publish method on a channel in confirm mode. The acknowledgement can be for a single message or a set of messages up to and including a specific message.  
当确认通知由客户端发出时，此方法通过`Deliver`或`Get-Ok`方法确认单条或多条消息已经送达。当确认通知由服务器发出时，此方法在`confirm`模式下通过`channel`由`Publish`方法确认单条或多条消息已成功发布。确认通知可用于单条消息或者用于包含特殊消息的一个消息集合。

[javadoc] [dotnetdoc] [amqpdoc]

---

`basic.cancel(consumer-tag consumer-tag, no-wait no-wait) ➔ cancel-ok`

Support: full  
支持： 完整  
End a queue consumer.  
结束一个队列消费者  

This method cancels a consumer. This does not affect already delivered messages, but it does mean the server will not send any more messages for that consumer. The client may receive an arbitrary number of messages in between sending the cancel method and receiving the cancel-ok reply. It may also be sent from the server to the client in the event of the consumer being unexpectedly cancelled (i.e. cancelled for any reason other than the server receiving the corresponding basic.cancel from the client). This allows clients to be notified of the loss of consumers due to events such as queue deletion. Note that as it is not a MUST for clients to accept this method from the server, it is advisable for the broker to be able to identify those clients that are capable of accepting the method, through some means of capability negotiation.  
此方法用来清除消费者。它不会影响到已经投递成功的消息，但是会使得服务器不再将新的消息投送给此消费者。客户端会在发送`cancel`方法和收到`cancel-ok`回复的过程中收到任意数量的消息。当消费者端发生不可预估的错误时，此方法也有可能由服务器发送给客户端（也就是说结束行为不是由服务器收到客户端的basic.cancel方法所触发）。次情形下客户端可以接收到由于队列被清除等原因引起的消费者丢失通知。需要注意的是，客户端从服务器接收此方法并不是必须的，它被推荐的原因在于消息代理可以通过它辨识能够通过协商方式访问方法的客户端。

[javadoc] [dotnetdoc] [amqpdoc]

---


`basic.consume(short reserved-1, queue-name queue, consumer-tag consumer-tag, no-local no-local, no-ack no-ack, bit exclusive, no-wait no-wait, table arguments) ➔ consume-ok`

Support: partial
Start a queue consumer.

This method asks the server to start a "consumer", which is a transient request for messages from a specific queue. Consumers last as long as the channel they were declared on, or until the client cancels them.

[javadoc] [dotnetdoc] [amqpdoc]

---

`basic.deliver(consumer-tag consumer-tag, delivery-tag delivery-tag, redelivered redelivered, exchange-name exchange, shortstr routing-key)`

Support: full
Notify the client of a consumer message.

This method delivers a message to the client, via a consumer. In the asynchronous message delivery model, the client starts a consumer using the Consume method, then the server responds with Deliver methods as and when messages arrive for that consumer.

[amqpdoc]

---

`basic.get(short reserved-1, queue-name queue, no-ack no-ack) ➔ get-ok | get-empty`

Support: full
Direct access to a queue.

This method provides a direct access to the messages in a queue using a synchronous dialogue that is designed for specific types of application where synchronous functionality is more important than performance.

[javadoc] [dotnetdoc] [amqpdoc]

---

`basic.nack(delivery-tag delivery-tag, bit multiple, bit requeue)`

*THIS METHOD IS A RABBITMQ-SPECIFIC EXTENSION OF AMQP*

Reject one or more incoming messages.

This method allows a client to reject one or more incoming messages. It can be used to interrupt and cancel large incoming messages, or return untreatable messages to their original queue. This method is also used by the server to inform publishers on channels in confirm mode of unhandled messages. If a publisher receives this method, it probably needs to republish the offending messages.

RabbitMQ Documentation
[javadoc] [dotnetdoc] [amqpdoc]

---

`basic.publish(short reserved-1, exchange-name exchange, shortstr routing-key, bit mandatory, bit immediate)`

Support: full
Publish a message.

This method publishes a message to a specific exchange. The message will be routed to queues as defined by the exchange configuration and distributed to any active consumers when the transaction, if any, is committed.

[javadoc] [dotnetdoc] [amqpdoc]

---

`basic.qos(long prefetch-size, short prefetch-count, bit global) ➔ qos-ok`

Support: partial
Specify quality of service.

This method requests a specific quality of service. The QoS can be specified for the current channel or for all channels on the connection. The particular properties and semantics of a qos method always depend on the content class semantics. Though the qos method could in principle apply to both peers, it is currently meaningful only for the server.

[javadoc] [dotnetdoc] [amqpdoc]

---

`basic.recover(bit requeue)`

Support: partial
Redeliver unacknowledged messages.

This method asks the server to redeliver all unacknowledged messages on a specified channel. Zero or more messages may be redelivered. This method replaces the asynchronous Recover.

[javadoc] [dotnetdoc] [amqpdoc]

---

`basic.recover-async(bit requeue)`

Redeliver unacknowledged messages.

This method asks the server to redeliver all unacknowledged messages on a specified channel. Zero or more messages may be redelivered. This method is deprecated in favour of the synchronous Recover/Recover-Ok.

[javadoc] [dotnetdoc] [amqpdoc]

---

`basic.reject(delivery-tag delivery-tag, bit requeue)`

Support: partial
Reject an incoming message.

This method allows a client to reject a message. It can be used to interrupt and cancel large incoming messages, or return untreatable messages to their original queue.

RabbitMQ blog post
[javadoc] [dotnetdoc] [amqpdoc]

---

`basic.return(reply-code reply-code, reply-text reply-text, exchange-name exchange, shortstr routing-key)`

Support: full
Return a failed message.

This method returns an undeliverable message that was published with the "immediate" flag set, or an unroutable message published with the "mandatory" flag set. The reply code and text provide information about the reason that the message was undeliverable.

[amqpdoc]

---

## Channel

`channel.close(reply-code reply-code, reply-text reply-text, class-id class-id, method-id method-id) ➔ close-ok`

Support: full
Request a channel close.

This method indicates that the sender wants to close the channel. This may be due to internal conditions (e.g. a forced shut-down) or due to an error handling a specific method, i.e. an exception. When a close is due to an exception, the sender provides the class and method id of the method which caused the exception.

[javadoc] [dotnetdoc] [amqpdoc]

---

`channel.flow(bit active) ➔ flow-ok`

Support: partial
Enable/disable flow from peer.

This method asks the peer to pause or restart the flow of content data sent by a consumer. This is a simple flow-control mechanism that a peer can use to avoid overflowing its queues or otherwise finding itself receiving more messages than it can process. Note that this method is not intended for window control. It does not affect contents returned by Basic.Get-Ok methods.

[amqpdoc]

---

`channel.open(shortstr reserved-1) ➔ open-ok`

Support: full
Open a channel for use.

This method opens a channel to the server.

[amqpdoc]

---

## Confirm

*THIS CLASS IS A RABBITMQ-SPECIFIC EXTENSION OF AMQP*

`confirm.select(bit nowait) ➔ select-ok`

.

This method sets the channel to use publisher acknowledgements. The client can only use this method on a non-transactional channel.

RabbitMQ Documentation
[javadoc] [dotnetdoc] [amqpdoc]

---

## Exchange

`exchange.bind(short reserved-1, exchange-name destination, exchange-name source, shortstr routing-key, no-wait no-wait, table arguments) ➔ bind-ok`

*THIS METHOD IS A RABBITMQ-SPECIFIC EXTENSION OF AMQP*

Bind exchange to an exchange.

This method binds an exchange to an exchange.

RabbitMQ Documentation
RabbitMQ blog post
[javadoc] [dotnetdoc] [amqpdoc]

---

`exchange.declare(short reserved-1, exchange-name exchange, shortstr type, bit passive, bit durable, bit auto-delete*, bit internal*, no-wait no-wait, table arguments) ➔ declare-ok`

*RABBITMQ-SPECIFIC EXTENSION OF AMQP*

Support: full
Verify exchange exists, create if needed.

This method creates an exchange if it does not already exist, and if the exchange exists, verifies that it is of the correct and expected class.

RabbitMQ implements an extension to the AMQP specification that allows for unroutable messages to be delivered to an Alternate Exchange (AE). The AE feature helps to detect when clients are publishing messages that cannot be routed and can provide "or else" routing semantics where some messages are handled specifically and the remainder are processed by a generic handler.

AE documention
[javadoc] [dotnetdoc] [amqpdoc]

---

`exchange.delete(short reserved-1, exchange-name exchange, bit if-unused, no-wait no-wait) ➔ delete-ok`

Support: partial
Delete an exchange.

This method deletes an exchange. When an exchange is deleted all queue bindings on the exchange are cancelled.

[javadoc] [dotnetdoc] [amqpdoc]

---

`exchange.unbind(short reserved-1, exchange-name destination, exchange-name source, shortstr routing-key, no-wait no-wait, table arguments) ➔ unbind-ok`

*THIS METHOD IS A RABBITMQ-SPECIFIC EXTENSION OF AMQP*

Unbind an exchange from an exchange.

This method unbinds an exchange from an exchange.

[javadoc] [dotnetdoc] [amqpdoc]

---

## Queue

`queue.bind(short reserved-1, queue-name queue, exchange-name exchange, shortstr routing-key, no-wait no-wait, table arguments) ➔ bind-ok`

Support: full
Bind queue to an exchange.

This method binds a queue to an exchange. Until a queue is bound it will not receive any messages. In a classic messaging model, store-and-forward queues are bound to a direct exchange and subscription queues are bound to a topic exchange.

[javadoc] [dotnetdoc] [amqpdoc]

---

`queue.declare(short reserved-1, queue-name queue, bit passive, bit durable, bit exclusive, bit auto-delete, no-wait no-wait, table arguments) ➔ declare-ok`

Support: full
Declare queue, create if needed.

This method creates or checks a queue. When creating a new queue the client can specify various properties that control the durability of the queue and its contents, and the level of sharing for the queue.

RabbitMQ implements extensions to the AMQP specification that permits the creator of a queue to control various aspects of its behaviour.

**Per-Queue Message TTL**
This extension determines for how long a message published to a queue can live before it is discarded by the server. The time-to-live is configured with the x-message-ttl argument to the arguments parameter of this method.

**Queue Expiry**
Queues can be declared with an optional lease time. The lease time determines how long a queue can remain unused before it is automatically deleted by the server. The lease time is provided as an x-expires argument in the arguments parameter to this method.

x-message-ttl documentation
x-expires documentation
[javadoc] [dotnetdoc] [amqpdoc]

---

`queue.delete(short reserved-1, queue-name queue, bit if-unused, bit if-empty, no-wait no-wait) ➔ delete-ok`

Support: partial
Delete a queue.

This method deletes a queue. When a queue is deleted any pending messages are sent to a dead-letter queue if this is defined in the server configuration, and all consumers on the queue are cancelled.

[javadoc] [dotnetdoc] [amqpdoc]

---

`queue.purge(short reserved-1, queue-name queue, no-wait no-wait) ➔ purge-ok`

Support: full
Purge a queue.

This method removes all messages from a queue which are not awaiting acknowledgment.

[javadoc] [dotnetdoc] [amqpdoc]

---

`queue.unbind(short reserved-1, queue-name queue, exchange-name exchange, shortstr routing-key, table arguments) ➔ unbind-ok`

Support: partial
Unbind a queue from an exchange.

This method unbinds a queue from an exchange.

[javadoc] [dotnetdoc] [amqpdoc]

---

## Tx

`tx.commit() ➔ commit-ok`

Support: full
Commit the current transaction.

This method commits all message publications and acknowledgments performed in the current transaction. A new transaction starts immediately after a commit.

[javadoc] [dotnetdoc] [amqpdoc]

---

`tx.rollback() ➔ rollback-ok`

Support: full
Abandon the current transaction.

This method abandons all message publications and acknowledgments performed in the current transaction. A new transaction starts immediately after a rollback. Note that unacked messages will not be automatically redelivered by rollback; if that is required an explicit recover call should be issued.

[javadoc] [dotnetdoc] [amqpdoc]

---

`tx.select() ➔ select-ok`

Support: full
Select standard transaction mode.

This method sets the channel to use standard transactions. The client must use this method at least once on a channel before using the Commit or Rollback methods.

[javadoc] [dotnetdoc] [amqpdoc]




[3]: http://www.rabbitmq.com/protocol.html
[6]: http://www.rabbitmq.com/amqp-0-9-1-reference.html

