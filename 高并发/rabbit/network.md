you can configure the rabbit server listening ipv4 and ipv6 like this:

```shell
listeners.tcp.1 = 127.0.0.1:5672
listeners.tcp.2 = ::1:5672
```

but on moden linux, if you only congiure ipv6 on all ipv6 address, the ipv4 address will be included:

```shell
listeners.tcp.1 = :::5672
==
listeners.tcp.1 = 127.0.0.1:5672
listeners.tcp.2 = ::1:5672
```



#### How to temporarily stop the new  client connections?

you can config the rabbit to stop all new connections temporarily, when configured, new connections will blocked, while old connections kept.

```shell
rabbitmqctl suspend_listeners [-n rabbit@hostname]
rabbitmqctl resume_listeners [-n rabbit@hostname]
```

shown as above, the first is used to stop connections, while second used to resume connections. on both cases, you can invoke commands against an arbitrary node, if no -n parameter, localhost default.