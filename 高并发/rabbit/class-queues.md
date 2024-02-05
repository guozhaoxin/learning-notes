the classic queues have 2 storage implementations, v1, the original one, and v2, which is available in rabbitmq 3.10 and later versions.

from rabbitmq 3.10.0, the broker has a new implementation of classic queues, which has a new index file format and implementations and a new per-queue storage file format, to replace the embedding of messages directly in the index.

under high memory pressure, the version 2 has a better and improved stability.

in 3.10.0, it's possible to switch between version 1 and version 2, but it may take a long time to do this if the queue is large (with too much messages),even leading to the queue unavailable. and duration the conversion, the queue will immediately convert its data to disk.

you can set version in the rabbit.conf:

```shell
classic_queue.default_version = 2
```



### about classic queue v1 version

there are 2 kinds of messages, the persistent and transient.

the persistent messages will be written to disk as soon as they reach the queue, and also kept in memory, but if under memory pressure, they may be evicted from memory. while the transient messages will be only kept in memory, and under memory pressure, they will be evicted from the memory.

every queue has an index for maintaining knowledge about where a message is in a queue, along with whether it has been delivered and acknowledged.

the messages can be stored directly in the queue index, or written to a message store.

for every disk-accessing queue, it needs a file handler, but the os has limit for the number of file handlers open  simultaneously. if beyond the limit, the queue will share the handlers by turns, which means that a queue will hold the handler for a while and then give it to another queue.