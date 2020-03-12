# Redis内存淘汰策略
我们可以在`redis.conf`配置文件中配置redis的最大内存以及内存淘汰策略对应的参数
```
maxmemory 4GB
maxmemory-policy noeviction
```
Redis内置了8种策略供我们选择，如下：
1. volatile-lru -> 在设置了过期时间的key中按近似LRU淘汰
2. allkeys-lru -> 在所有key中按近似LRU淘汰
3. volatile-lfu -> 在设置了过期时间的key中按近似LFU淘汰
4. allkeys-lfu -> 在所有key中按近似LFU淘汰
5. volatile-random -> 在设置了过期时间的key中随机淘汰
6. allkeys-random -> 在所有key中随机淘汰
7. volatile-ttl -> 淘汰快要过期的key，也是近似算法，算是保证效率和精度的一种平衡手段
8. noeviction -> 不淘汰key，给客户端抛出异常  
其中，Redis默认是第8种，不淘汰key。第5.6种采用随机算法，第7种按过期时间淘汰，
第8种不淘汰，我在这里不多说，我们主要说前4种。  

我们先来说说LRU和LFU
+ LRU (Least Recently Used) 即我们常说的最近最少使用，它主要是按照时间顺序来选择，哪个key最近一次访问离现在时间最远，就选择哪个key
+ LFU (Least Frequently Used) LFU除了关注时间，还会关注在这段时间内每个key使用的次数(即频率)，应该说算法更复杂，也会消耗更多的CPU资源
 
 但是1.2.3.4以及第7种策略，Redis都使用了近似的算法，来平衡淘汰的效率和准确度。
 同样，Redis也提供了参数，可以让我们来调节近似算法。默认情况下，Redis会检查5个key来选出一个key淘汰。
 ```
maxmemory-samples 5
```

最后，在redis cluster集群环境种的replication节点，是不会执行内存淘汰策略的，他只会接收master节点的del命令来删除key。
```
replica-ignore-maxmemory yes
```