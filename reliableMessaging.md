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
