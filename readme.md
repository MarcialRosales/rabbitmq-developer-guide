# RabbitMQ Developers Guide

There are lots of very good books ([RabbitMQ in Action](https://www.manning.com/books/rabbitmq-in-action), [RabbitMQ in Depth](https://www.manning.com/books/rabbitmq-in-depth)) and [extensive documentation](https://www.rabbitmq.com/documentation.html) about RabbitMQ. It is clear that we don't need yet another book. This guide collects a set of recommendations and best practices on how to use RabbitMQ.

Most of the recommendations are applicable to any AMQP library we use. However, when it matters, this guide will provide separate advise for Java AMQP and Spring AMQP clients.

Best Practices and recommendations:
- [Establishing a connection](establishConnection.md)
- [Declaring queues and exchanges and bindings (RabbitMQ resources](declaringResources.md)
- [Message Delivery](messageDelivery.md)
- [Reliable Messaging - what can go wrong?](reliableMessaging.md)

- [Message Loss Tolerance Use Cases](messageLossTolerance.md)
- [Factors that affect message's latency](latencyFactors.md)
