#
指定配置文件：
`ES_PATH_CONF=/home/zhangbo/apps/elasticsearch-7.6.1/config1  ./bin/elasticsearch -d`

配置文件`elasticsearch.yml`
```
#集群名称，必须一致
cluster.name: elasticsearch

#节点名称，不一样，这里按照node-1、node-2、node-3进行命名
node.name: node-3

path.data: /home/zhangbo/apps/elasticsearch-7.6.1/data3
path.logs: /home/zhangbo/apps/elasticsearch-7.6.1/logs3
#注意：需要进行配置，如果使用默认配置是很危险的。当es被卸载，数据和日志将完全丢失。可以根据具体环境将数据目录和日志目录>保存到其他路径下以确保数据和日志完整、安全。

discovery.seed_hosts: ["127.0.0.1:9301","127.0.0.1:9302","127.0.0.1:9303"]
cluster.initial_master_nodes: ["127.0.0.1:9301","127.0.0.1:9302","127.0.0.1:9303"]

#把 bootstrap.memory_lock: false 注释放开
#是否锁住内存。因为当jvm开始swapping时es的效率会降低，配置为true时要允许elasticsearch的进程可以锁住内存，同时一定要保证>机器有足够的内存分配给es。如果不熟悉机器所处环境，建议配置为false。

#添加 bootstrap.system_call_filter: false
#Centos6不支持SecComp，而es5.2.0版本默认bootstrap.system_call_filter为true
#禁用：在elasticsearch.yml中配置bootstrap.system_call_filter为false，注意要在Memory的后面配置该选项。

network.host: 127.0.0.1
http.port: 9203 #设置端口为9200
transport.port: 9303

#这里需要配置多个，为了演示非同一IP段，所以IP段不同
#因为我们搭建了3台服务器用于演示，所以此处为三个服务器配置的network.host地址，
# 不能和 discovery.seed_hosts 一起配置
#discovery.zen.ping.unicast.hosts: [ "127.0.0.1"]

#master集群中最小的master数量，集群都是过半投票制，所以3台服务器设置2个master节点，如果19台服务器可以设置5个master节点，因为设置的是最小master节点数量防止宕机过多。
#在Elasticsearch7.0版本已被移除，配置无效
#为了避免脑裂，集群的最少节点数量为，集群的总节点数量除以2加一

discovery.zen.minimum_master_nodes: 2
```