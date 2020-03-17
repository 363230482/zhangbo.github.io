#kafka配置推荐
具体配置信息参见官方网站[http://kafka.apache.org/documentation/#configuration]：  

##broker端参数
```text
# 可配置多个路径，可以将数据挂载到不同的物理磁盘上，提升吞吐量以及实现故障转移
log.dirs=/home/kafka1,/home/kafka2,/home/kafka3  
# 只能配置单个路径
log.dir=/home/kafka

# 最后的/kafka1 可适用于多个kafka集群使用同一个zookeeper集群，区分不同的kafka集群
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka1

# 默认为true，最好设置为false，避免topic写错出现各种奇怪的topic
auto.create.topics.enable=false
# 建议设置为false，避免那些数据落后很多的replica被选举成leader，造成数据丢失
unclean.leader.election.enable=false
# 避免leader定期切换带来的影响
auto.leader.rebalance.enable=false

# 数据保存的时间，默认是168hour
log.retention.{hour|minutes|ms}=168
# broker允许使用的最大c磁盘容量，一般用于云 服务器的多租户场景
log.retention.bytes=1000000000
# broker能接收的最大消息大小，最好设置大一点，默认1000012，不到1M
message.max.bytes=5000
```
##topic级别
> 可使用kafka-topics.sh在创建topic时设置以下参数，也可使用kafka-configs.sh来修改指定的参数
```text
# 该topic的消息保存时间，默认7天，可根据业务设置不同的topic
retention.ms
# 该topic使用的最大磁盘容量，适用于多租户场景
retention.bytes
```
##JVM参数
> JVM参数需要设置为环境变量
```text
# 一般可设置heap为6G，使用G1GC，可根据实际情况调优
export KAFKA_HEAP_OPTS=--Xms6g  --Xmx6g
export  KAFKA_JVM_PERFORMANCE_OPTS= -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true
```
##操作系统级
> 1.根据官网的测试报告，XFS的性能要强于ext4，所以生产环境最好还是使用XFS.  
> 2.建议将swappniess配置成一个接近0但不为0的值，比如1.因为一旦设置成0，当物理内存耗尽时，操作系统会触发OOM killer这个组件，它会随机挑选一个进程然后kill掉，即根本不给用户任何的预警。但如果设置成一个比较小的值，当开始使用swap空间时，你至少能够观测到Broker性能开始出现急剧下降，从而给你进一步调优和诊断问题的时间。  
> 3.提交时间或者说是Flush落盘时间.操作系统默认5秒/30秒会刷新脏页到磁盘。但鉴于Kafka在软件层面已经提供了多副本的冗余机制，因此这里稍微拉大提交间隔去换取性能还是一个合理的做法。
```text
# 尽量设置文件描述符大点
ulimit -n 1000000
```