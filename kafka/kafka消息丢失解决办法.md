#kafka消息丢失的解决办法  
首先是对消息丢失的定义：kafka只保证对"已提交的消息"做到不丢失。"已提交的消息"即已写入日志文件的消息。  
消息丢失，可能是在producer端发送的时候丢失，可能是在broker端丢失，也可能是consumer自动提交offset但没有消费成功导致的。  

我们来一一解决；  
##producer端
>1.默认的producer.send(record)是异步发送的，可能会出现网络抖动导致发送失败的情况；
推荐使用producer.send(record, callback)方法发送，在回调函数中对发送失败的消息做相应的处理。  
2.producer端配置`acks = all`,表明所有broker都接收了消息才算"已提交"。  
3.设置`reties`为一个合理的值，让producer能够自动重试消息发送。
##broker端
>1.设置`unclean.leader.election.enable = false`，避免消息落后很多的broker被选举成为leader。  
2.设置`replication.factor`大于等于3，通过消息冗余的方式防止消息丢失。  
3.设置`min.insync.replicas`大于1，保证broker在生产环境中存活的数量大于1。默认值为1。  
4.确保`replication.factor` > `min.insync.replicas`。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。
我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成replication.factor = min.insync.replicas + 1
##consumer端
>1.设置`enable.auto.commit = false`，关闭自动提交offset。