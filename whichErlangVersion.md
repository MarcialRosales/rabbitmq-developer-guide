## Which Erlang version is RabbitMQ using

We obtained the following using `rabbitmqctl status|environment|report`:
```
{running_applications,
     [{rabbitmq_web_stomp,"Rabbit WEB-STOMP - WebSockets to Stomp adapter",
          "3.6.15"},
      {rabbitmq_shovel_management,
          "Management extension for the Shovel plugin","3.6.15"},
      {rabbitmq_management,"RabbitMQ Management Console","3.6.15"},
      {rabbitmq_web_dispatch,"RabbitMQ Web Dispatcher","3.6.15"},
      {rabbitmq_stomp,"RabbitMQ STOMP plugin","3.6.15"},
      {rabbitmq_shovel,"Data Shovel for RabbitMQ","3.6.15"},
      {rabbitmq_mqtt,"RabbitMQ MQTT Adapter","3.6.15"},
      {rabbitmq_management_agent,"RabbitMQ Management Agent","3.6.15"},
      {rabbit,"RabbitMQ","3.6.15"},
      {os_mon,"CPO  CXC 138 46","2.4.2"},
      {amqp_client,"RabbitMQ AMQP Client","3.6.15"},
      {rabbit_common,
          "Modules shared by rabbitmq-server and rabbitmq-erlang-client",
          "3.6.15"},
      {compiler,"ERTS  CXC 138 10","7.0.4.1"},
      {cowboy,"Small, fast, modular HTTP server.","1.0.4"},
      {sockjs,"SockJS","0.3.4"},
      {xmerl,"XML parser","1.3.14"},
      {ranch,"Socket acceptor pool for TCP protocols.","1.3.2"},
      {recon,"Diagnostic tools for production use","2.3.2"},
      {ssl,"Erlang/OTP SSL application","8.1.3.1"},
      {public_key,"Public key infrastructure","1.4"},
      {asn1,"The Erlang ASN1 compiler version 4.0.4","4.0.4"},
      {cowlib,"Support library for manipulating Web protocols.","1.0.2"},
      {crypto,"CRYPTO","3.7.4"},
      {syntax_tools,"Syntax tools","2.1.1"},
      {inets,"INETS  CXC 138 49","6.3.9"},
      {mnesia,"MNESIA  CXC 138 12","4.14.3"},
      {sasl,"SASL  CXC 138 11","3.0.3"},
      {stdlib,"ERTS  CXC 138 10","3.3"},
      {kernel,"ERTS  CXC 138 10","5.2"}]},
{os,{unix,linux}},             
{erlang_version,
     "Erlang/OTP 19 [erts-8.3.5.3] [source] [64-bit] [smp:2:2] [async-threads:64] [hipe] [kernel-poll:true]\n"},
 ```

Given the information above, which Erlang Version is RabbitMQ running with?

We follow these steps to find that out:

**1. Erlang major version**

Erlang major version is `19` -> We get it from Erlang/OTP **19** [erts-8.3.5.3]

**2. Erlang minor version**

Erlang minor version is `3`-> We get it from Erlang/OTP 19 [erts-8.**3**.5.3]

**3. Search for the release and patch versions**

We are going to search for OTP releases that bundles `erts-8.3.5.3`.

1. Go to `https://github.com/erlang/otp/tags?after=OTP-19.3`. It lists the releases older than `OTP-19.3`. The most recent one we get is `OTP-19.2.3 `. We click on `Previous` button so that we list the next releases.
2. On the `OTP-19.3` release, we click on the `...` symbol to expand it and see its content. It bundles `erts-8.3` not `erts-8.3.5.3`.
3. We continue searching for newer `OTP-19.3` versions until we find `OTP-19.3.6.3` which bundles `erts-8.3.5.3`.
4. But to be 100% sure that is the OTP release, we need to check `OTP-19.3.6.2` which happened to bundle `erts-8.3.5.2`. So, we are ok, `OTP-19.3.6.2`  is not our release.
5. However if we look at the next immediate release, which is `OTP-19.3.6.4`, it also bundles `erts-8.3.5.3`, so how do we know which OTP version are we using? We have to look for other core libraries like `ssl`. Our RabbitMQ is running with `ssl-8.1.3.1` thus we need to find an OTP which also bundles that `ssl` version. We find that `OTP-19.3.6.4` bundles `ssl-8.1.3.1` whereas `OTP-19.3.6.3` bundles `ssl-8.1.3`. We have finally found that RabbitMQ is running with `OTP-19.3.6.4`.
