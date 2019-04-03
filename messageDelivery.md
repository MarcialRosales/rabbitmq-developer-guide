# Message Delivery

## I need to know if a message was successfully sent to RabbitMQ and made it to all the bound queues

There are 2 ways of doing it:
- [`publisher confirmations`](https://www.rabbitmq.com/confirms.html) : RabbitMQ sends back to the application a confirmation as soon as the message made it to the queue(s) -if message is persistent, the confirmation is not sent until the message is persisted. We can use publisher confirms in [Spring AMQP](https://docs.spring.io/spring-amqp/reference/#template-confirms) too.
- [`alternate exchange`](https://www.rabbitmq.com/ae.html) or [mandatory message flag]((https://www.rabbitmq.com/api-guide.html#returning) : If there are no queues bound to the exchange, RabbitMQ will still send a confirmation to our application. If we cannot afford to loose messages, we need to make sure the message goes somewhere and it is not lost. We can configure the exchange with an `alternate exchange` and the alternate exchange bound to a queue, typically called, Dead Letter Queue. Or instead we can send the message with the `mandatory` flag and register a [ReturnListener](https://www.rabbitmq.com/api-guide.html#returning).


## Publisher Confirmation How-to


1) The Producer’s AMQP channel must be set to “Publisher Confirms” mode: `channel.confirmSelect();`

2) Every message sent by the Publisher to the channel must not be considered as sent until a “Publisher Confirm” for that particular message arrives.

3) The publisher must keep which messages have not received a confirmation yet.

4) After we send a message, we insert it into a data structure as a raw byte array and index it by its deliveryTag :
```
  deliveryTag = channel.getNextPublishSeqNo();
  channel.basicPublish(....);
```

5) Once we receive a confirmation via the `ConfirmListener.handleAck(deliveryTag, multiple)` method, we clear from the data structure all the messages up to the `deliveryTag ` if `multiple` is true else only the message with that `deliveryTag`.


6) If we receive a negative confirmation via the `ConfirmListener.handleNack(deliverTag, multiple)` method, we need to resend all the messages up to the `deliveryTag` if `multiple` is true. We retrieve the messages from the data structure.

Warning: We have to resend the messages from the same publishing thread, not from the `ConfirmListener` callback. We cannot simultaneously use the same channel from different threads.

7) How long do we wait for publisher confirmations? It really depends on our application design. See the following scenarios we may encounter, hopefully some of them applies to your application design:

**Scenario 1)**

```
      [Upstream App]  ----(sends)---> [Our producer application]-----(publish)------>[RabbitMQ]
                                                               <-----(confirms)------
                     <----(acks)-----
```

**Our producer application** receives messages from an **up-stream** application. The upstream application expects an acknowledgment (positive or negative) for each message it delivers to our producer application. This is the best scenario we can encounter because our producer application simply forwards messages to **RabbitMQ** and publisher confirmation acknowledgements back to the **up-stream** application.

Our producer app only needs to remember 2 values:
  - `LastDeliveryTagIdSent`
  - and `LastDeliveryTagIdReceived`

The difference between these two values is the number of messages not acknowledged yet.

Let's see a sequence of events so that we see this in action:
  1. **Our producer application** receives 3 messages from **Upstream app**.
  2. **Our producer application** sends those 3 messages to **RabbitMQ** and updates the internal local variables as follows:  `LastDeliveryTagIdSent = 3` and the `LastDelivetyTagIdReceived = 0`.
  3. **Our producer application** receives a *positive publisher confirmation* with `deliveryTagId = 2` (with `multiple = true`). It forwards the acknowledgements (for the first 2 messages) to **Upstream App** and update the internal local variables as follows: `LastDeliveryTagIdReceived = 2`.
  4. **Our producer application** receives a *negative publisher confirmation* for `deliveryTagId = 3`. It forwards the negative acknowledgment (for the last sent message) to the **Upstream app** and updates the internal variable: `LastDeliveryTagIdReceived = 3`.
  5. It is up to the **upstream application** to resend the message, not our application.

How long shall we wait for a confirmation? That depends on the contract with the upstream. If the upstream is capable of resending a message if it does not receive an acknowledgment on-time, we don't need to do anything at all. Else, we have to implement a Timer that sends negative acknowledgements for those messages we have not received a publisher confirmation yet.

**Scenario 2)**

```
     [Upstream App]  ----(sends and forgets)---> [Our producer application]-----(publish)------>[RMQ]
                                                                           <-----(confirms)------
```

**Our producer application** receives messages from an **upstream application** which delivers the messages and forgets about them.

The **upstream application** assumes the message is delivered as soon as it sends it out. Therefore it is the responsibility of our producer application to guarantee message delivery.

This is the worst scenario because we have to deal with several scenarios we did not have in the previous scenario.

For instance, if we receive a *negative publisher confirmation* we need to be able to send the message again. For that, we need to keep the message body in a reliable message store. By reliable I mean any message store we can trust should our producer application crashed. Certainly we should not use local memory. On top of that, we have to have a timer that sends the message again if we don't receive a publisher confirmation after certain amount of time. And also, we need to think what do we do with those messages in the message store if our producer application crashes or terminates.

8) Spring AMQP has methods that we can call to retrieve the last messages sent but not confirmed in the last N millisec (RabbitTemplate.getUnconfirmed(millis)); And we can resend those messages right away via the RabbitTemplate.


## I need to know if a message made it to a consumer
AMQP defines another delivery-related flag named immediate, which is a step beyond mandatory in the sense it ensures that the message has been actually delivered to a consumer. RabbitMQ has chosen not to support this feature.

## Can I send messages to my queues without using any exchanges, like in JMS....?
Use of `default exchange` (direct, durable) to publish messages to queues is very convenient because it resembles other messaging styles like JMS where we published messages directly to queues. But this convenience creates a tight coupling between producers and consumers because the producer becomes aware of particular queue names. This basically voids the benefits of having the layer of indirection.

Limit the use of default exchanges to PoC.


## What if I never get a Publisher confirmation?
There are many reasons for confirmations to not come back:
- Failed to replicate the message in a rabbit node
- Failed to persist the message
- Network Partition
- Timeout
- Shutdown/Channel closed unexpectedly


## Do not requeue poison messages
Try to avoid requeuing messages that are likely to be repeatedly rejected by the consumer (poison messages).  
<b>Poison message</b>: A message that contains badly formatted payload or invalid data. Sending it back to the queue is simply a waste of resources because the broker will keep delivering and the consumers will keep rejecting it.

<b>Java AMQP</b>  
Wrap your consumers around a filter that based on the type of exception, it decides whether to requeue or reject the message.

<b>Spring AMQP</b>  
When a listener throws an exception, it is wrapped in a `ListenerExecutionFailedException` and, normally the message is rejected and requeued by the broker. Setting `defaultRequeueRejected` in the `MessageListenerContainer` to false will cause messages to be discarded (or routed to a dead letter exchange). The listener can throw an `AmqpRejectAndDontRequeueException` to conditionally control this behavior.

If your application relies on Spring `MessageConverter`, Spring can fail to convert the message without our application knowing it. When this situation occurs, Spring will automatically throw `MessageConversionException` which is handled by the `ErrorHandler` configured in the `MessageListenerContainer`. The default implementation is  `ConditionalRejectingErrorHandler` which automatically rejects the message. We can customize this class by injecting our implementation of `FatalExceptionStrategy`.

## Handle outage of external parties
Sometimes, our consumers need to talk to an external service over the network, say a database or a log repository like Splunk, or any other service you can think of. Plan in advance in case of outage of that external service.
If the queue has a TTL or a max depth, there are no issues because it is bounded (we need to cater for this in our capacity planning). If the expired messages go to a DLQ, we need to be sure that we consume those messages too.
If the queue has no TTL or max length constrains, we either stop the publisher or we use Lazy Queues and provision disk space to handle the outage. The latter is quite difficult to get right because sometimes it is not easy to estimate the maximum outage period.

## My queue cannot take any more load -> Scale out using consistent-hashing
Eventually, one queue will not be able to keep up with the demands. We will notice it when the queue is constantly in flow control (io bound) or when the queue cannot deliver messages at the same pace as they come in and the consumer utilization is 100% (i.e. consumers are willing to take more messages). One cpu can only get up to certain amount of work done (1 queue process is only executed by one cpu at once). The only way out is to spread the load and have multiple queues, one per node. This is called sharding. We can shard the queue ourselves along with the consistent-hashing plugin or rely on RabbitMq to shard the queues for us using a sharding plugin.

Another way to scale out is to spread the load across other clusters using federated queues.

## What do we do with the messages in the Dead-Letter-Queues?
We should have a DLX/DLQ associated to a queue especially if we cannot afford to loose messages. But once those messages are in the DLQ queue what do we do with them? Shall we retry them? Shall we move them somewhere else for auditing purposes? Regardless, we need to drain them from the queue otherwise we are creating another problem.
If we want to implement an automated redelivery of those messages back to the original queue if the consumer(s) dropped them. We don’t want to retry indefinitely though. There is a paper that describes in detail how to implement it using RabbitMq features: http://dev.venntro.com/2014/07/back-off-and-retry-with-rabbitmq/
If messages are being dropped because of TTL and/or max depth reasons, we certainly want to know about it. Maybe we want to add more consumers or make those consumers consume faster (e.g. using larger prefetch and/or batching acks).

## Do I need to care about Consumer Cancellations?
<b>Java AMQP</b>  
It depends:  
- If there is a possibility of someone deleting the queue while consumers are attached to it. We use the callback to declare the queue again.
- If you are using non-durable queues and the consumer is connected to a different node where the queue was created. This is very plausible when we use `master-queue-locator = min-masters`. We use the callback to declare the queue again.
- If you are using mirrored queues with client acknowledgement there is no reason why we need to handle consumer cancellations. By default, RabbitMQ will not notify consumer cancellations for mirrored queues unless it is not able to elect a master queue.
- If you are using mirrored queues without client acknowledgements there is a possibility of loosing messages if the master queue goes down. If we want to be notified in this case, say to refresh the application state from an external storage, we need to tell RabbitMq to notify us.

By default (RabbitMq's capability `consumer_cancel_notify = true`), all clients are notified via the `Consumer.handleCancel` method, except for mirrored queues.

If we are consuming from a non-mirrored durable queue and the node with that queue goes down, the queue is not available. Like in the previous case, the client will receive a callback thru the `Consumer.handleCancel` method. But in this case, the client will not be able to create/declare the queue because it is durable. But at least the client knows that the consumer does not exist anymore and it has to resubscribe once it is able to declare the queue again. This will only be possible when the failed node comes back up.

If we are consuming from a mirrored queue and the node with the master queue goes down, the queue is still available provided RabbitMQ can elect a new master. Consumers are not cancelled and therefore clients are NOT notified of this situation. This is because RabbitMq transparently subscribes the client to the new master queue.
If RabbitMQ is not able to elect a new master, either because there are no more nodes or the available slaves are not synchronized (and `ha-promote-on-shutdown = when-synced`, the default value), the queue is not available and consumers are cancelled and get notified via `Consumer.handleCancel` method. We need to handle this situation so that at least our application is aware that we have stopped listening.

If we are consuming from a mirrored queue using auto-ack and the node with the master queue goes down right after sending a number of messages, the client may never receive those messages and they don’t know it. However, consumers may request that they should be cancelled when a mirrored queue fails over.
```
Map<String, Object> args = new HashMap<String, Object>(); args.put("x-cancel-on-ha-failover", true);
String consumerTag = channel.basicConsume(queueName, autoAck, args, myconsumer);
```

<b>Be aware that</b> our consumer may stop receiving messages if it does not know that RabbitMq closed its channel.

<b>Spring AMQP</b>  
TODO

# How to improve publishing throughput

* Do not use Mandatory flag
* If you don't require guarantee of delivery and can afford to loose messages, do not use publisher confirmations
* If you can afford to loose messages even upon a RabbitMQ node failure, do not use message persistency (Spring AMQP's `MessageBuilder` builds `Message`(s) with persistency)
* Use Micro-batching technique -when applicable- which consists on sending more than one message into a single AQMP Message. Consumers must support this model. Be careful with the overall size of the batch message as we don't want to send Messages of several Megabytes in size.

# How to improve delivery throughput

* If your consumer requires manual acknowledgements, try to increase the prefetch count until your consumers does not need to wait for messages (default prefetch count is 1 which is typically quite low).  Prefetch-count is the maximum number of unacknowledged messages allowed in the client.
* If your consumer requires manual acknowledgements, consider acknowledging messages in batches (a good ratio is 50% of the prefetch-count)
* If your consumer does not require manual acknowledgement (i.e. auto acks), RabbitMQ will send as many messages as the client can hold in its TCP buffer. The prefetch-count is not used because as soon RabbitMQ sends the message it is considered as acknowledged.
