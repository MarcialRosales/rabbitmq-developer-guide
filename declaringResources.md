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
