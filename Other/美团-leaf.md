#美团Leaf 分布式ID生成
[https://tech.meituan.com/2017/04/21/mt-leaf.html]  

## segment模式
双buffer，支持服务高可用(集群)，削减数据库update的突刺，在DB宕机时buffer仍可支持几分钟
在current buffer剩余90%可用时，由后台线程异步更新next buffer
可自定义max_id，方便迁移原数据
可自定义step，用于缓存与内存中，使用AtomicLong#addAndIncrement()
id趋势递增，但ID是可计算的，竞争对手可计算每天的ID新增量，猜测订单等
读写锁优化，只在update DB时用写锁，在内存中递增ID时，读锁，降低锁开销

## snowflake模式
沿袭snowflake的方案：1(占位默认0)+41(时间戳)+10(workId)+12(seq,最大4095)
其中workId采用zookeeper的持久化顺序节点自动分配，如节点少，也可用手动分配，减少zookeeper的依赖
定时(3s)给zookeeper上报时间戳

### 解决时钟回拨问题：
在启动时，检查在zookeeper中储存的时间戳，如果小于上次上报的时间戳，则启动失败
如果为第一次启动，则获取zookeeper中目前存活的server的时间戳的平均时间，和当前系统的时间比较，如果在一定阈值(3s)内，则成功，否则，启动失败


