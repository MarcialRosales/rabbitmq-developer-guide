
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
