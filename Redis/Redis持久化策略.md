#Redis持久化策略
我们都知道Redis持久化可以分为`RDB`和`AOF`两种方式，下面我们来具体说说两种方式的优缺点。
详情可参见官方文档[https://redis.io/topics/persistence]及配置文件说明[https://redis.io/topics/config]
文件保存路径可通过配置：
```text
dir ./    #数据保存目录
dbfilename dump.rdb    #RDB文件名称
appendfilename "appendonly.aof"    #AOF文件名称
```

##RDB
Redis默认的持久化策略。RDB会以fork的方式生成一个子进程dump内存中的数据。  
此时如果客户端有写命令，则会采取copy-on-write的方式，复制一份内存数据，以保障子进程能正常dump。
新的写命令同时会写入一个内存buffer中，在子进程dump完数据后，会将buffer中缓冲的写命令添加到RDB文件中。
Redis在生成dump临时文件后，会以`rename`的方式，原子的将老的dump文件覆盖，不存在dump文件写一半的情况。
```text
#   save ""    # 如不需要RDB持久化，则可这样配置
# 默认配置如下
save 900 1    #在900秒(15分钟)内，有一个key改变
save 300 10   #在300秒内，有10个key改变
save 60 10000 #在60秒内，有10000个key改变
```
###RDB优点
>1.以二进制格式保存，内容更紧凑，便于远程传输及基于时间点备份不同的版本。  
2.在重启恢复时加载速度更快
###RDB缺点
>1.如果你在乎数据的丢失，那RDB不是好的选择。它根据我们配置的策略，最多会丢失几分钟的数据。  
2.如果数据很大，fork的时候可能会比较耗时，甚至导致Redis停止为客户端服务几毫秒甚至1秒。

##AOF
以Redis文本协议的方式，直接将写命令追加在AOF文件尾部。  
```text
appendonly no    #默认不开启AOF，如需开启，设置为yes
# appendfsync always    #每次都fsync刷新，性能低，一般不使用
appendfsync everysec    #每秒调用fsync刷新缓冲区到磁盘文件
# appendfsync no    #不主动刷新，依赖操作系统的刷新机制。Linux下默认30秒刷新一次
no-appendfsync-on-rewrite no    #在bgsave和rewrite时，不禁用fsync，保证不丢失数据，但可能造成主进程阻塞。如果存在客户端请求延迟的情况，可设置为yes
auto-aof-rewrite-percentage 100    #重写策略，默认AOF文件增大到之前的一倍时开始重写
auto-aof-rewrite-min-size 64mb    #AOF重写策略，当AOF文件达到64M时才会重写
```
###AOF优点
>1.数据丢失少，默认丢失1秒的数据。  
2.由于明文保存命令，当我们错误的执行了如`FLUSHALL`等命令时，我们只需到AOF文件中将该命令删除即可恢复。
###AOF缺点
>1.AOF文件较大，不利于远程传输。  
2.启动恢复时，加载数据比较慢。  

##混合使用
其实在Redis4.0版本开始，`AOF`的重写策略可以变为RDB+AOF的方式。
即AOF在重写时，会先根据内存dump一份RDB格式数据写到AOF文件，后续的写命令则会以AOF的方式追加到文件末尾。

此种方式可有效缩小AOF文件的大小，以及在重启恢复时能更快的恢复加载数据。
但现在Redis默认不会启动该策略，需要我们手动配置开启：
```text
aof-use-rdb-preamble yes #默认为no
```
目前不开启混合使用的原因是，该协议后期可能会更改，导致不同版本之间数据无法恢复的问题。