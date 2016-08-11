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
