# RabbitMQ Developers Guide

## Establishing a connection

### The application should start whether a connection to RabbitMQ can be established or not

<b>Spring AMQP </b>
Our Spring application will start just fine even though it cannot connect to RabbitMQ. Spring will attempt to reconnect every so often.

<b>Java AMQP</b>
Our Java application should not attempt to connect to RabbitMq from the main thread otherwise our application will fail to start if we do not handle the exception properly. At least, we should catch the exception and schedule another attempt (ideally using exponential back-off).

### If the connection to RabbitMQ is lost, our application should reconnect

In case of connection failure, we can delegate to Spring AMQP and/or Java AMQP to automatically recover the connection, and channels and consumers by using `connectionFactory.setAutomaticRecovery(true)`.

<b>Spring AMQP </b>
We have the possibility of setting up exponential back-off retries and maximum number of retries too.

<b>Java AMQP</b>
There are 2 issues with Java AMQP. The first one is that it is very limited and the retry interval is fixed and we define a maximum number of attempts either.
And the second issue is that we need to wait until we have a connection in order to create the queues, exchanges and bindings. This may not be a problem and it boils down to application design and preferences.

### If the library reconnects by itself, should I add a ShutdownListener to the connection?

It really depends on whether the shutdown was initiated by the application and how much you rely on the library to automatically recover “everything” if the shutdown was not initiated by the app.
If the shutdown was initiated by the application, you definitely don’t want to reconnect but to terminate the application. The question is whether you need to do some clean up and where you want to implement them. Should this `ShutdownListener` trigger the clean up of resources or instead should the application logic who initiated the shutdown who is in charge of it?
If the shutdown was not initiated by the application and we rely on library to automatically recover then why would I need a `ShutdownListener`? Definitely for tracking purposes, either to log it or to record it in some Health Status bean exposed via a `/health` http endpoint.

### Once more, do I need to add a ShutdownListener to each channel?
A channel will receive a shutdown event if:

- the connection closed (`isHardError = true`). If we are using automatic
recovery we don’t have much to do but to wait until it recovers (we can add a RecoveryListener, https://www.rabbitmq.com/api- guide.html#recovery-listeners). Regardless who initiated the close operation, we may want to clean up some resources (e.g. db connection, socket connection, etc) when we disconnect from Rabbit or only when the application terminates.
- the application closed the connection or the channel itself (`initByAp = true`). If we need to do some clean up, when do we do it? On this ShutdownListener’s callback or before this callback?
- The application invoked an operation in the channel that caused an exception (`isHardError = false`). The channel is automatically closed, consumers will stop receiving messages and automatic recovery does not trigger in this case. It is up to the application do deal with it. The producer thread can quickly handle the situation within the try/catch block. It does not need a ShutdownListener. If the exception originates within the consumer’s callback then the consumer has to create a new channel and registers again as a consumer because its subscription is already cancelled. But this scenario could have been handled via the Consumer.handleCancel method.

Regardless what triggers the shutdown event and whether we want to do something about it or not, we should at least track it. Spring AMQP automatically logs these events but we may want to change the logging statements and/or track in any other way.

### Which methods, in the DefaultConsumer class, should I override in addition to handleDelivery?
The other relevant methods are `handleCancel` and `handleShutdownSignal`. We ignore `handleCancelOk` unless we want to do something in our application when we cancel the subscription. Maybe we want to log it for troubleshooting purposes. Or maybe our application has to call some tear down procedure.

`handleCancel` is called when RabbitMq cancels the subscription. This could be happen due to the following reasons:
- The node with non-HA queue goes down while the consumer is connected to a different node, or
- The node with the master HA queue goes down while the consumer is connected to a different node and the consumer has instructed RabbitMq to notify when this happens:

    ```
    Channel channel = ...;
    Consumer consumer = ...;
    Map<String, Object> args = new HashMap<String, Object>(); args.put("x-cancel-on-ha-failover", true);
    ￼￼
    channel.basicConsume("my-queue", false, args, consumer);
    ```

It is advisable that we implement this method so that we can track it (e.g. have a counter and expose that counter thru a `/health` HTTP endpoint) and we can tear down some state and issue another subscription.

It is mandatory to implement this method for non-HA queues. For non-durable we could use this method to re-subscribe but be aware that we need to declare the queue again because it will not exist.

`handleShutdownSignal` is called when either the connection or the channel was closed, unexpectedly or gracefully. Do we need to care about this method if we already have a `ShutdownListener` on the channel and/or on the connection? It really depends on your application design. If your consumer needs to do some tear down procedure, say it has to release some resources (db connection, socket connection, etc.), this method and also the `handleCancel` are very convenient to trigger a tear down procedure. Furthermore if you keep some health or stats state about your subscriptions so that you can expose thru a `/health` HTTP endpoint, you need these two methods to keep that state consistent in addition to `handleCancelOk`.

### Do I need to add an ExceptionHandler to the ConnectionFactory?
By default, Java AMQP and Spring AMQP will log any exception to the standard error

<b>Spring AMQP</b>
If our DefaultConsumer throws an exception from the `handleDelivery` method, Spring automatically rejects the (`requeue`) message so that RabbitMQ can redeliver it again to another consumer.

<b>Java AMQP</b>
In Java AMQP there is a catch we need to be aware of. If we are using client acknowledgements and our consumer’s method handleDelivery throws an exception, and we don’t have a custom ExceptionHandler, the message remains not acknowledged as far RabbitMQ is concern. If our Prefetch size is 1, that means RabbitMQ will not deliver any more messages. When our application closes the channel and/or the connection, RabbitMq makes the message “ready” again.

### How many channels do I need per connection?
There are no hard rules here. We must ask ourselves why would I want to have more than one channel per connection? One pretty valid reason is to separate producers from consumers, in other words, one channel to send messages and another to receive them. Another reason is to separate different type of producers, say one producer sends messages of type A and another producer sends messages of type B and most likely those producers are running simultaneously, i..e on different threads. Another reason, which may not be that evident, is to increase throughput. Rather than having one thread sending data to an exchange, we have several threads sending data to the same exchange over the same connection. Do we gain any increase in throughput? In a multi-core machine, various threads could run on different cores, in parallel, against a single connection. However, as we add more channels to a connection we get a diminish return. We hit a limit, either in the number of cores available or due to heavy contention writing to the single connection. In a 4 cpu machine, I have noticed that beyond 8-10 channels I get diminshing return.

On the client side, a channel may be a costless but on the server side, in RabbitMq, they are expensive in terms of memory and cpu cost. Each channel means 4 Erlang processes with their associated data structures (memory) and stats emissions (cpu) that runs every N seconds which sends stats to the stats db. As the number of channels (and obviously connections) increases, so does the load in RabbitMq.


## Declaring queues and exchanges and bindings (RabbitMQ resources)

### Let each application declare the RabbitMq resources they need
Client applications should always assume their queues, exchanges and bindings do not exist and do not survive node restart and therefore they have to create them. Every time our application connects to RMQ we should create all our resources and this means every time, even after a connection failure.

Some teams prefer to delegate the declaration of queues, exchanges, bindings, policies, etc, to a central application. I am not saying this is pattern is wrong but it certainly comes at a price.
<b>Cons</b>:
- You need to make sure the central application runs before you start your applications.
- The central application must have some configuration file where we declare all the RabbitMq resources our applications need. This mean application owners must push their changes to the central application before they start and before the central application starts. In a distributed system this level of coordination is a cumbersome and root causes of issues.

### Do I need to declare the RabbitMq resources every time my application starts?
It is a good practice to declare your RabbitMq resources when your application establishes a connection with RabbitMQ. This is regardless it is a reconnection attempt or it is the first connection.

We can rely on the automatic topology recovery feature that comes with Java AMQP library or do it all ourselves in the `ShutdownListener` when `isHardError = true`.

Just for Reference:
```
Channel.queueDeclare(“queueName”, durable, exclusive, autoDelete, arguments);
```
- `durable` = true if we want messages to survive broker restarts
- `exclusive` = true if we want a queue dedicated to our channel and which is deleted once the connection is closed (e.g. chat application where we don’t care to loose messages while we are offline, reply-queue in an RPC scenario)
- `autodelete` = true if we want RabbitMq to delete the queue after the last subscription is removed from it.

### Should I create `durable` queues and exchanges?
`durable` means a Queue and/or an Exchange survives a broker restarts.

<b>If our application always declares the resource (queue, exchange, bindings), why do I need durability?</b>:
- If your application cannot afford to loose messages, you need the queue to be `durable` and of course, you must send your messages with `deliveryMode = 2` (persistent) rather than 1 (non-persistent, default).
- If your application can afford to loose messages you don't need to declare queues as `durable` but make sure your application always declare them every time it opens a connection.
- If you don't declare your exchanges as `durable`, you know you need to declare them every time you open a connection.

<b>Be aware that</b> : Exclusive queues will never be mirrored (even if they match a policy) and they are neither durable (even if declared as such).

<b>Be aware that</b> : If your declare your queues as `Durable` and the node where they were declared goes down, your producer applications will not notice messages are not getting to your queue, your consumer applications will receive a `handleCancel` event and applications trying to declare the queue will fail too. Why all this? Because the queue can only exist in one node. If RabbitMq would allow us to declare the queue in another node, we essentially end up with more than one copy or instance of a queue. In terms of the CAP theorem, RabbitMq has preferred Consistent over Availability. What if I also need availability? You need to make the queue `mirrored`.

### Mirrored queues

### how do I customize queues and exchanges, for instance, with a Dead-Letter-Exchange or with a TTL?
There are 2 ways of customizing a queue or exchange. One way is programmatically when we declare queue/exchange by passing a set of arguments.
```
```
The second, and i should say the proper way of doing it, is to use `Policies`.
```
```

The reason is quite simple. AMQP protocol establishes that once a queue/exchange is declared, we cannot change its settings. Say for instance, version 1 of our application creates a queue as durable without any settings. On version 2 we realize that we need to configure a Dead-Letter-Queue and put some TTL on the messages so that old messages go to the DLQ. Our v2 application will not be able to declare the queue because it conflicts with whats in RabbitMq. We would have to create a new queue with the new settings, empty the v1 queue to the v2 queue. A big hassle!

TODO: talk about how we deal with the policy constraint where there can only be one policy applied to a queue/exchange.

## Message Delivery

### I need to know if a message was successfully sent to RabbitMQ and made it to at least a queue
There are 2 ways of doing it:
- `publisher confirmations` : RabbitMq sends back to the application a confirmation as soon as the message made it to the queue(s) -if message is persistent, the confirmation is not sent until the message is persisted.
- `alternate exchange` : If there are no queues bound to the exchange, RabbitMQ will still send a confirmation to our application. If we cannot afford to loose messages, we need to make sure the message goes somewhere and it is not lost. We can configure the exchange with an `alternate exchange` and the alternate exchange bound to a queue, typically called, Dead Letter Queue.

These 2 methods will not handle one edge situation which is sending a message to a non-existing exchange. RabbitMq will drop the messages right away. The only way to know if RabbitMq did send the message is to rely on the `mandatory` flag and registering a `ReturnListener` (https://www.rabbitmq.com/api-guide.html#returning) however this mechanism has an important performance penalty. I would rather change my application design to make sure that the exchange always exist before sending any message to it.


### Publisher Confirmation How-to

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
<p/>Scenario 1): Our producer application receives messages from an up-stream application. The upstream application expects an acknowledgment (positive or negative) for each message it delivers to our producer application. This is the best scenario we can encounter because our producer application simply sends messages to RabbitMQ. Our producer app only needs to remember 2 values: LastDeliveryTagIdSent and LastDeliveryTagIdReceived. The difference between these two values is the number of messages not acknowledged yet. Say for instance, we have sent 3 messages. The LastDeliveryTagIdSent = 3 and the LastDelivetyTagIdReceived = 0. After we receive a positive publisher confirmation for deliveryTagId = 2 (with multiple = true), we update the LastDeliveryTagIdReceived and send an acknowledment to the upstream for the messages 1 and 2. Later on we receive a negative publisher confirmation for deliveryTagId = 3. We update the LastDeliveryTagIdReceived = 3 and we send a negative acknowledgement to the upstream for message 3. It is up to the upstream application to resend the message, not our application.
How long shall we wait for a confirmation? That depends on the contract with the upstream. If the upstream is capable of resending a message if it does not receive an acknowledgment on-time, we don't need to do anything at all. Else, we have to implement a Timer that sends negative acknowledgements for those messages we have not received a publisher confirmation yet.

<p/>Scenario 2): Our producer application receives messages from an upstream application which delivers the messages and forgets about them. The upstream application assumes the message is delivered as soon as it sends it out. Therefore it is the responsibility of our producer application to guarantee message delivery. This is the worst scenario we can be because we have to deal with several scenarios we did not have in the previous scenario. For instance, if we receive a negative publisher confirmation we need to be able to send the message again. For that, we need to keep the message body in a reliable message store. By reliable I mean any message store we can trust should our producer application crashed. Certainly we should not use local memory. On top of that, we have to have a timer that sends the message again if we don't receive a publisher confirmation after certain amount of time. And also, we need to think what do we do with those messages in the message store if our producer application crashes or terminates.

8) Spring AMQP has methods that we can call to retrieve the last messages sent but not confirmed in the last N millisec (RabbitTemplate.getUnconfirmed(millis)); And we can resend those messages right away via the RabbitTemplate.



### I need to know if a message made it to a consumer
AMQP defines another delivery-related flag named immediate, which is a step beyond mandatory in the sense it ensures that the message has been actually delivered to a consumer. RabbitMQ has chosen not to support this feature.

### can I send messages to my queues without using any exchanges, like in JMS....?
Use of `default exchange` (direct, durable) to publish messages to queues is very convenient because it resembles other messaging styles like JMS where we published messages directly to queues. But this convenience creates a tight coupling between producers and consumers because the producer becomes aware of particular queue names. This basically voids the benefits of having the layer of indirection.

Limit the use of default exchanges to PoC.


### What if I never get a Publisher confirmation?
There are many reasons for confirmations to not come back:
- Failed to replicate the message in a rabbit node
- Failed to persist the message
- Network Partition
- Timeout
- Shutdown/Channel closed unexpectedly


### Do not requeue poison messages
Try to avoid requeuing messages that are likely to be repeatedly rejected by the consumer (poison messages).
Poison message: contains badly formatted payload or invalid data. Sending it back to the queue is simply a waste of resources because the broker will keep delivering and the consumers will keep rejecting it. Wrap your consumers around a filter that based on the type of exception, it decides whether to requeue or not the message. This feature is available in Spring AMQP (FatalExceptionStrategy).

### Handle outage of external parties
Sometimes, our consumers need to talk to an external service over the network, say a database or a log repository like Splunk, or any other service you can think of. Plan in advance in case of outage of that external service.
If the queue has a TTL or a max depth, there are no issues because it is bounded (we need to cater for this in our capacity planning). If the expired messages go to a DLQ, we need to be sure that we consume those messages too.
If the queue has no TTL or max length constrains, we either stop the publisher or we use Lazy Queues and provision disk space to handle the outage. The latter is quite difficult to get right because sometimes it is not easy to estimate the maximum outage period.

### My queue cannot take any more load -> Scale out using consistent-hashing
Eventually, one queue will not be able to keep up with the demands. We will notice it when the queue is constantly in flow control (io bound) or when the queue cannot deliver messages at the same pace as they come in and the consumer utilization is 100% (i.e. consumers are willing to take more messages). One cpu can only get up to certain amount of work done (1 queue process is only executed by one cpu at once). The only way out is to spread the load and have multiple queues, one per node. This is called sharding. We can shard the queue ourselves along with the consistent-hashing plugin or rely on RabbitMq to shard the queues for us using a sharding plugin.

Another way to scale out is to spread the load across other clusters using federated queues.

### What do we do with the messages in the Dead-Letter-Queues?
We should have a DLX/DLQ associated to a queue especially if we cannot afford to loose messages. But once those messages are in the DLQ queue what do we do with them? Shall we retry them? Shall we move them somewhere else for auditing purposes? Regardless, we need to drain them from the queue otherwise we are creating another problem.
If we want to implement an automated redelivery of those messages back to the original queue if the consumer(s) dropped them. We don’t want to retry indefinitely though. There is a paper that describes in detail how to implement it using RabbitMq features: http://dev.venntro.com/2014/07/back-off-and-retry- with-rabbitmq/
If messages are being dropped because of TTL and/or max depth reasons, we certainly want to know about it. Maybe we want to add more consumers or make those consumers consume faster (e.g. using larger prefetch and/or batching acks).

### Do I need to care about Consumer Cancellations?
Yes, if you are using non-mirrored queues or there is the possibility of someone deleting the queue while consumers are attached to it.
Yes, if you are using non-durable queues and the consumer is connected to different node where the queue was created. This is very plausible when we use `master-queue-locator = min-masters`.
Yes, if you are using mirrored queues. See below.

By default (capability `consumer_cancel_notify = true`), all clients are notified via the `Consumer.handleCancel` method. Therefore, we need to implement this method to create a new transient queue and subscribe to it.

If we are consuming from a non-mirrored durable queue and the node with that queue goes down, the queue is not available. Like in the previous case, the client will receive a callback thru the `Consumer.handleCancel` method. But in this case, the client will not be able to create/declare the queue because it is durable. But at least the client knows that the consumer does not exist anymore and it has to resubscribe once it is able to declare the queue again. This will only be possible when the failed node comes back up.

If we are consuming from a mirrored queue and the node with the master queue goes down, the queue is still available provided RabbitMQ can elect a new master. Consumers are not cancelled and therefore clients are NOT notified of this situation. This is because RabbitMq transparently subscribes the client to the new master queue.
If RabbitMQ is not able to elect a new master, either because there are no more nodes or the available slaves are not synchronized (and `ha-promote-on-shutdown = when-synced`, the default value), the queue is not available and consumers are cancelled and get notified via `Consumer.handleCancel` method. We need to handle this situation so that at least our application is aware that we have stopped listening.

If we are consuming from a mirrored queue using auto-ack and the node with the master queue goes down right after sending a number of messages, the client may never receive those messages and they don’t know it. However, consumers may request that they should be cancelled when a mirrored queue fails over.
```
Map<String, Object> args = new HashMap<String, Object>(); args.put("x-cancel-on-ha-failover", true);
String consumerTag = channel.basicConsume(queueName, autoAck, args, myconsumer);
```

<b>Be aware that</b> our consumer may stop receiving messages if it does not know that RabbitMq closed its channel. TODO prepare a lab to test this.


## Reliable Messaging – what can go wrong?

Many things can go wrong during message publication and consumption. AMQP specifies how to exchange messages in a reliable manner. RMQ implements these reliable mechanisms.
```
   [Publisher]1----{network}2--->[Broker(master)]3--->{network}4---->[Consumer]5
                                      |
                                 {network}6
                                      |
                              [Broker(mirror)]7
```

### The broker and/or publisher can crash while published messages are in flight; these messages can be lost
<b>Problem</b>: The broker and/or publisher can crash while published messages are in flight; these messages can be lost
</br><b>Action</b>: Enable publisher confirmation if your application cannot tolerate message loss. Publisher confirmation does not mean we wont loose messages. If a publisher receives a positive publisher confirmation, it means that the broker received the message and it will make its best to deliver it. It is the responsibility of the publisher to resend the message.
There are many reasons for Publisher confirmations not to come back:
- Failed to replicate the message to a mirror
- Failed to persist the message
- Network Partition
- Timed out
- Shutdown/Channel closed unexpectedly

Note about duplicate messages: Publisher confirmations could lead to duplicates if the publisher looses its connection before the confirmation arrives. If the publisher resends the message, the broker has the same message twice.

### The broker crashes; sent but not-yet consumed messages are lost
<b>Problem</b>: The broker crashes; sent but not-yet consumed messages are lost
</br><b>Action</b>: Enable message persistence (+ durable queues) so that they are stored in disk.

### The consumer crashes after the broker delivered N messages
<b>Problem</b>: The consumer crashes after the broker delivered N messages; messages are lost for auto-ack consumers
</b><b>Action</b>: Enable manual acknowledgements so that the broker can redeliver all not acknowledged messages. Manual acknowledgements are necessary if your application cannot tolerate message loss

### The broker crashes, sent but not-yet consumed persistent messages are not lost
<b>Problem</b>: The broker crashes; sent but not-yet consumed persistent messages are not lost but publishers and consumers cannot use the queue until it broker comes back
</br><b>Action 1</b>: If the mean time to recover (MTTR) the broker (i.e. the time it takes to restart the VM where the broker was running) is acceptable, we don’t need anything else to do. The applications must be written with an exponential-
backoff connection retry policy and eventually they connect again and resume their operations.
</br><b>Action 2</b>: If the MTTR is not acceptable, enable Mirror or HA Queues. We will trade off throughput, memory scalability and network bandwidth for availability.

### The broker with the master queue crashes; but there are fully synchronized slaves
<b>Problem</b>: The broker with the master queue crashes; but there are fully synchronized slaves
</br><b>Action</b>: A slave is elected as master. Applications connected to the failed broker have to reconnect. Applications connected to other nodes are not affected. There is no message loss.

### The broker with the master queue crashes; delivered but not-yet acknowledged messages are resent
<b>Problem</b>: The broker with the master queue crashes; delivered but not-yet acknowledged messages are resent; this leads to duplicate messages
</br><b>Action 1</b>: Write idempotent applications (e.g. rely on primary key constraints) so that they gracefully handle duplicate messages.
</br><b>Action 2</b>: Put de-duplication filter in front of the consumers backed by an in- memory database like Redis or Gemfire.

### The broker with the master queue is gracefully shutdown; but there are only unsynchronized slaves; Queue is not available
<b>Problem</b>: The broker with the master queue is gracefully shutdown; but there are only unsynchronized slaves; RabbitMQ will not elect a slave as master; Queue is not available
</br><b>Action</b>: We need to bring the shutdown node back up.

Note: Unsynchronized slaves do not bring much redundancy. Carefully monitor which queues have unsynchronized messages. If the number of unsynchronized messages becomes too risk consider manually synchronizing it (`rabbitmqctl sync_queue <queue>`]. Be aware that producers will be blocked while the manual synchronization takes place.

### The entire cluster goes down unexpectedly; we have HA queues without persistency; sent but no-yet consumed messages will be lost
<b>Problem</b>: The entire cluster goes down unexpectedly; we have HA queues without persistency; sent but no-yet consumed messages will be lost
</br><b>Action</b>: We need to evaluate how likely this scenario is and whether are willing to pay the additional price of having persistency on top of HA.

### During a rolling upgrade, which does not require a full cluster shutdown; transient messages on non-HA queues are lost
<b>Problem</b>: During a rolling upgrade, which does not require a full cluster shutdown; transient messages on non-HA queues are lost
</br><b>Action 1</b>: Most likely these messages are not critical and therefore there is no risk of losing them. If we did not want to loose them we could move them to another RMQ node/cluster using the Shovel plugin. But before we move messages from one node to another node we should consider blocking all producers by setting the high water mark to 0 (rabbitmqctl set_vm_memory_high_watermark 0).
</br><b>Action 2</b>: Another way to prevent any message loss is to block producers and wait until the consumers drain all the messages from the queue before proceeding with the rolling upgrade.

### During a rolling upgrade, which requires a full cluster shutdown; transient messages, on HA or non-HA queues, are all lost
<b>Problem</b>: During a rolling upgrade, which requires a full cluster shutdown; transient messages, on HA or non-HA queues, are all lost
</br><b>Action</b>: Apply the same technique described in the previous problem.

### Network partition occurs between RMQ nodes; pause_minority; master HA queues that happened to be in the minority site loose their messages once the nodes in the minority join the majority
<b>Problem</b>: Network partition occurs between RMQ nodes; cluster_partition_handling = pause_minority; master HA queues that happened to be in the minority site loose their messages once the nodes in the minority join the majority
</br><b>Action</b>: If you cannot afford to loose messages then pause_minority is not the right mode.

### Network partition occurs between RMQ nodes; auto_heal; master HA queues that happened to be in the minority site loose their messages if there are no consumers on the minority site draining the queues.
<b>Problem</b>: Network partition occurs between RMQ nodes; `cluster_partition_handling = auto_heal`; master HA queues that happened to be in the minority site loose their messages if there are no consumers on the minority site draining the queues. When the minority joins back the majority, any not-yet consumed messages will be lost
</br><b>Action</b>: If you cannot afford to loose messages then auto_heal is not the right mode although there is less chance to loose messages compared to the pause_minority mode.

About `cluster_partition_handling = ignore`. With this mode we don’t loose messages however it requires a manual procedure (that it is feasible to automate) to merge the messages from the minority site into the majority site before the minority joins the majority.

### Network partition occurs between clients and Rabbit
<b>Problem</b>: Network partition occurs between clients and Rabbit (i.e. between 1 and 3 and 5) not between Rabbit nodes; publishers may loose messages sent without publisher confirmations and auto-ack consumers may loose messages too.
</br><b>Action</b>: Enable publisher confirmation and client acknowledgements if we cannot afford to loose messages

Note: Thanks to the heartbeat timeout (https://www.rabbitmq.com/heartbeats.html), AMQP clients detect the network partition. Clients get a connection failure, which they can use to reconnect.

## Message Loss Tolerance Use Cases

### No Tolerance to Message Loss
Application cannot recreate lost messages and/or cannot communicate the sender to send them again.
We are not assessing critically from a business or risk standpoint.

Minimum Required Features:
- Publisher confirmations (it is on the publisher’s responsibility to resend
messages which RabbitMq could not send)
- Manual Client acknowledgment
- Durable queues
- Message Persistence (deliveryMode = 2)
- Cluster partition handling = ignore
- Dead-letter exchange bound to every queue (in case the application nack
messages with requeue=false)
- Alternate exchange bound to every exchange (in case publishers start up
before consumers declare and bind their queues) with a queue bound to
it.
- No policies with TTL or max queue depth or autoexpire parameters

It has occurred that RabbitMQ’s message store gets corrupted making impossible to recover the messages contained in it. It could happen that the VM crashed before RabbitMQ completed the write operation. I am just speculating as to what could have happened. But regardless, this is something completely outside of RabbitMq.

If we make the queues mirrored, we have at least multiple copies of that file. Should the master failed to start because the file is corrupted another slave node takes over. We are playing here with probabilities, what is the probability that 2 files are corrupted at the same time, or 3, etc.

Another option, a certainly a bit overkill, is to replicate (federated exchange) messages to a separate RMQ node/cluster that servers as a backup.

Note about DLX: RabbitMQ injects a custom header name x-death, which contains extra contextual information about the cause of death. This extra header is a key-value map with the following entries:
- queue: This indicates the queue name where the message was stored before it expired
- exchange: This indicates the exchange that this message was sent to
- reason: This indicates whether the message is rejected, the TTL for the
message has expired, or the queue length limit is exceeded
- time: This indicates the date and time when the message was dead
lettered
- routing keys: This indicates all the routing keys associated with the
message (RabbitMQ supports multiple routing keys in an extension to AMQP known as the sender-selected destination, which is beyond the scope of this book and is fully documented at http://www.rabbitmq.com/ sender-selected.html)

In the event of an unclean broker shutdown, the same message may be duplicated on both the original queue and on the dead-lettering destination queues. This is because the queue could have sent it to the DLX/DLQ and the publisher confirmation message back from DLX/DLQ to the original queue did not arrive. This could happen due to a network partition or an unclean broker shutdown.

Additional Features to add redundancy:
- Mirrored queue (with policy parameters ha-sync-mode = automatic, and
ha-promote-on-shutdown = when-synced –this is the default value
though, do not use “always” because you could loose messages)
- Federated Exchange

### Low Tolerance to Message Loss
Application does not expect message loss except in rare or exceptional circumstances.

Minimum Required Features:
- Publisher confirmations (it is on the publisher’s responsibility to resend
messages which RabbitMq could not send)
- Manual Client acknowledgment
- Durable queues
- Message Persistence (deliveryMode = 2)
- Dead-letter exchange bound to every queue (in case the application nack
messages with requeue=false)
- No policies with TTL or max queue depth or autoexpire parameters

The likelihood of network partitions or incomplete exchange-to-queue bindings is considered very low and if it occurred the application tolerates any message loss.

### Medium Tolerance to Message Loss
Application can handle message loss due to exceptional circumstances and/or as a safety mechanism to prevent excessive load (e.g. message TTL, max depth).

It is worth adding an extra classification:
a. Applications which favours high availability and low latency over message
loss
b. Applications which prefers not to loose messages over high availability

For applications of type a), these are the minimum required features:
- Publisher confirmations (it is on the publisher’s responsibility to resend
messages which RabbitMq could not send)
- Manual Client acknowledgment
- Policy with TTL and/or max depth

If the node where our queues are goes down, our client application can connect to another node and recreate the exchanges and queues and the bindings and carry on publishing and consuming.
Only in case of node failure and/or excessive load, this application will suffer message loss.

For applications of type b), these are the minimum required features:
- Publisher confirmations (it is on the publisher’s responsibility to resend
messages which RabbitMq could not send)
- Manual Client acknowledgment
- Durable queues
- Message Persistence (deliveryMode = 2)
- Policy with TTL and/or max depth

If the node where our queues are goes down, the application does not loose any messages but it looses availability. Publishers cannot publish and consumers see how their subscription is cancelled. The only way to restore the service is to bring the node back up.

### High Tolerance to Message Loss
Application can handle message loss. Distribution of log messages is not critical; it is ok to loose messages. Distribution of indicative prices (e.g. forex or stocks) is not critical; it is ok to loose some prices. But it is very important to have high availability.

Required Features:
- Non durable queues
- Policy with TTL and/or max depth

Additional Features:
- Shard the queues (manually using consistent-sharding plugin or automatically using sharding plugin)

## Factors that affect message’s latency

### Mirroring
It slows down publishing rate because the message has to get to N slaves queues and those queues may well be in flow control. If we use “Publisher confirmation”, publishing rate is dropped even further.

Note: When a node joins a cluster (say it was down due to maintenance or it crashed) and RabbitMQ allocates a slave queue on it, the queue is automatically considered unsynchronized if the master queue is not empty. If the queue is configured with `ha_sync_mode = automatic` (the default is `manual`), RabbitMq will block the producers until the new slave has a replica of all the messages in the master. This `ha_sync_mode = automatic` has a negative impact on the publishing rate and it may also affect the consumers because we are moving messages from one queue to another over the network.

To make the synchronization faster we can tune how many messages Rabbit should send to the slave in one go (https://www.rabbitmq.com/ha.html#sync- batch-size). By default, RabbitMQ sends one by one. Take into account the average message size to calculate the size of the batch and compare it with your network capacity. We don’t want to max out the network and cause net_ticktime packets to time out and cause Network partitions.


### Persistency
It slows down publishing and delivery rate due to disk IO operations. There is just one message store (for persistent messages and another for transient messages) where Rabbit stores the message’s body and one index file per queue. RabbitMQ has a number of specialized threads (we can tune the size of this pool) to deal with IO operations. Regardless, disk IO is considerably slower compared to network access.

If we need persistency but still low latency, consider sharding your queues. Effectively you would be writing in parallel to different files on different file systems.

### Mandatory flag
This flag has a negative impact on the publishing rate because RabbitMq has to synchronously wait for the queue(s) to confirm that they received the message. We should not use this flag and instead use publisher confirmation and an alternate exchange.

When a message is published on an exchange with the mandatory flag set to true, it will be returned by RabbitMQ if the message cannot be delivered to a queue. A message cannot be delivered to a queue either because no queue is bound to the exchange, or because none of the bound queues have a routing key that would match the routing rules of the exchange

### Client Acknowledgements
They impact performance because client has to send explicit ACKs to the broker and the broker has to manage additional cases: unacknowledged messages, acknowledgements and rejections.

Typically, consumers can process messages quicker than the time it takes for the messages to go from Rabbit to the consumer. When we use manual acknowledgements, it is important that we adjust the prefetch size so that the consumer does not need to wait for messages. RabbitMq will send as many unacknowledged messages to the consumer as indicated in the prefetch size reducing this way the number of round-trips between broker and consumer.

To further reduce message latency and queue depth and increase overall performance of Rabbit we should acknowledge messages in batches rather than one by one. The benefits of this technique are more evident when RabbitMq has to go to disk to read the messages (for instance, when it has to page messages to disk or when using Lazy queues). Every time the consumer acks a message, Rabbit has to go to disk. If we reduce the number of acks we are reducing the number of disk operations and as a consequence we are reducing latency and improving overall performance.

Note: In worst scenario, an acknowledgement queue can grow as big as the current active queue or until memory capacities are starved. Would this occur, producer would be under flow-control. Therefore, we need to monitor the number of unacknowledged messages. If they grow beyond acceptable levels (> #consumer x batch ack size) there is something wrong with the consumers.

There are no time outs in AMQP for client acknowledgements. Try to design your application so that it raises an alarm (or it actually rejects the message) when it takes too long to ack the message. It is also recommended to measure the average time (and max) to handle a message. This information can help us tune the prefetch size.

Note: We can use basicNack to nack multiple messages (this is not part of the AMQP specification) at once: basicNack(deliveryTag, multiple, requeue).

### Publisher Confirmations
They reduce publishing rate and increase message latency. The worse latency occurs when we combine mirroring and persistency because a message can only be confirmed if it has been persisted in the master and all slave queues.

To reduce publishing rate and overall message latency consider batching the messages, i.e. send N messages in 1 message and awaits one confirmation. This reduces chattiness between RMQ and the publisher. The result is a quicker publisher confirmation messages especially for fast publishers. This technique is appropriate if the overall message size does not become an impediment, e.g. if it is greater than 1Mbyte. The drawback is that consumers have to be aware that they can receive batches.

### Clustering
RabbitMQ clustering feature allows us to connect to any node in the cluster regardless of the location of the queue we are publishing (via an exchange) to or consuming from. This comes at a price. If the AMQP client is not directly connected where the queue is, RabbitMq has to redirect the traffic consuming additional network bandwidth and cpu and memory. This additional network hop means an increase in message latency.

### Message size
It is a profound negative effect directly proportional to its size. The message needs to traverse the network at least twice (1). The larger the message the longer it takes to write it to disk and more disk space occupies (in case of persistent messages or in case of paging out transient messages). And more memory it occupies.

(1) If consumer and producer are connected to the same node where the queue is and the message is only published to queues on the same node and we are not using mirroring.
Note: Consider using efficient encoding protocols like ProtoBuf or compressing the message’s body if you use a text-based encoding like Json, YAML or XML.

￼￼
### Unbalanced connections and queues across the cluster
Having a 5-node cluster makes no difference if we have all connections and queues concentrated in 1 or 2 nodes.
To achieve a better-balanced cluster we can use a Load Balancer (round-robin, or least-loaded node) in front of the cluster or make some applications connect to their preferred nodes first with a fall-back strategy (i.e. connect to any other node if we cannot connect to the preferred ones).

The former approach only distributes the connection but not necessarily the queues. To even distribute the queues we can use a policy’s parameter called queue-master-locator (https://www.rabbitmq.com/ha.html#queue-master- location) = min-masters. The default value is client-local which is the standard behaviour where queue gets created in the node where we were connected when we declared it. This can also apply to mirror queues which is very convenient.

The latter approach is far more flexible but also more complex. The application is fully aware of the cluster topology, which nodes exist, where queues exist and where producers and consumers connect. But on the other hand, it may bring lots of dividends in terms of performance. It really depends on the application design. For instance, it does not matter that we declare the queue in node A, if we are using mirror all. This is just one example.

### Unbalanced producer/consumer ratio
If publishing rate exceeds consumption rate, sooner or later, we have message backlog, either in memory or in disk. And this results in poor performance and eventually blocking publishers.

It is difficult to give a good ratio because at the end of the day it really depends on producing vs consumption rate. In other words, if the consumer can consume 10 times quicker than the producer can produce then a 5:1 ratio (i.e. 5 concurrent producers and 1 concurrent consumer) could be more than enough. In the ideal world, 10:1 should work but that assumes the consumer is working at full throttle and it never has to wait for messages to arrive. If we cannot meet any of these two premises we consumer will lag behind and soon we have message backlog and its consequences. That is why, we have to carefully find the right balance between concurrent producers and consumers in addition to other settings like prefetch size (a.k.a. QoS) and ack batch sizes.
