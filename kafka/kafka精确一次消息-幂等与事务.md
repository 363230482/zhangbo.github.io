#kafka精确一次消息实现
通常来说，消息中间件对消息交付可靠性保障可分为3个层次：
>1.最多一次(at most once),消息可能丢失，但不会重复发送  
2.精确一次(exactly once)，消息不会丢失，也不会重复发送  
3.最少一次(at least once)，消息不会丢失，但可能重复发送

kafka默认提供最少一次的保证，由于producer会在没有收到ack时自动重试，故可能导致重复消息。

kafka也可以实现最多一次的语义，即关闭producer的重试即可。

今天我们主要讨论kafka时怎么实现精确一次的。简单来说，kafka通过两种机制来实现精确一次。即：
>1.幂等性(Idempotence)  
2.事务(Transaction)  

## 幂等性 Idempotence
幂等性我们可以在producer端设置参数`enable.idempotence=true`即可保证在单个producer会话期间，消息在单个分区上不会重复。
它是有broker在保存消息时，额外多保存了一些字段，以此区分重复消息。它在producer重启后生成新的会话，即不会与之前会话的消息校验重复。

## 事务 Transaction
事务能够保证将消息原子性地写入到多个分区中。这批消息要么全部写入成功，要么全部失败。另外，事务型Producer也不惧进程的重启。Producer重启回来后，Kafka依然保证它们发送消息的精确一次处理。

要开启事务型消息，需要设置producer端参数`enable.idempotence=true`与`transctional. id`。另外在发送消息，也需要额外的事务操作。
```text
producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(record1);
    producer.send(record2);
    producer.commitTransaction();
} catch (KafkaException e) {
    producer.abortTransaction();
}
```
事务消息可以保证多条消息原子的写入，要么都成功，要么都失败。实际上即使写入失败，kafka也会把它们写入到底层的日志中，也就是说Consumer还是会看到这些消息。
故提交事务消息后，还需要修改consumer端的默认参数`isolation.level=read_committed`，该值默认为`read_uncommitted`.