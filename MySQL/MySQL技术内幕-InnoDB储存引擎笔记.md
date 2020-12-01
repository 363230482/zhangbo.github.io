
## Mysql执行流程
连接器负责检查当前用户是否在允许的ip登录(sys.user表)

## 线程类型
1.Master线程:负责定时flush内存脏页，insert buffer,redolog等到磁盘
2.IO线程:
innodb_read_io_threads
innodb_write_io_threads
3.Purge线程:
innodb_purge_threads 回收undolog内存
4.Page Cleaner Thread
Innodb1.2.x引入，用于刷新脏页，减轻master线程压力

## Buffer pool
innodb_buffer_pool_size
缓冲区主要缓冲了 索引页，数据页，undolog buffer，insert buffer，自适应哈希索引，锁信息，数据字典等。

Innodb允许有多个缓冲池实例，以减少医院竞争，增强并发能力，模式为1
innodb_buffer_pool_instances=1

Innodb缓冲区采用LRU算法淘汰使用很少的内存页(16k)，默认最新访问的页并不是放在LRU的头部，而是在LRU列表尾部37%的位置，该算法称为midpoint insertion strategy.可由 innodb_old_blocks_pct=37  配置。
innodb_old_blocks_time=1000 表示新数据页在加入到LRU列表mid位置后多久才能加入LRU中的热点部位

innodb_log_buffer_size，redolog缓冲区，默认16m，会每秒写入日志，所以只需缓冲一秒内事务产生的redolog即可。redolog flush到磁盘的情况:
1.master线程每秒刷新一次
2.事务提交时会刷新该事务redolog到磁盘
3.redolog buffer剩余小于1/2时

## 刷新内存脏页策略
Sharp checkpoint 在mysql服务关闭时强制刷新所有脏页 innodb_fast_shutdown=1
Fuzzy checkpoint 运行中刷新部分脏页

InnoDB存储引擎开创性地设计了Insert Buffer，对于非聚集索引的插入或更新操作，不是每一次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入；若不在，则先放入到一个Insert Buffer对象。然后在后台进行merge.
Insert buffer只能在非唯一的二级索引上使用
Insert buffer默认最大到innodb_buffer_pool_size的1/2

innodb_change_buffer_max_size=25，最大为50，即innodb_buffer的25%，为innodb_change_buffering=all  可开启各种change buffer:inserts、deletes、purges、changes、all、none，默认all
Insert buffer merge策略:
1.select包含insert buffer中的二级非唯一索引页到内存中
2.insert buffer所在页可用空间小于1/32时
3.master thread后台每10秒一次

## InnoDB性能提升方式
### Double write
Innodb在flush脏页时，都会先将页copy到double write buffer(2m)中，在将数据按每次1m顺序写入磁盘上的共享表空间中的double write区域中，然后在写入(随机写)
对应表的数据文件(ibd)来提高数据可靠性。
skip_innodb_doublewrite 可以禁用该功能

### redo log
redo log是按512字节进行储存的，若一个页中redo log数量大于512字节，则需要分割为多个redo log。
由于大小为512字节，为磁盘扇区的大小，每次写入是原子的，故不需要double write。

redo log 条目是由4个部分组成：
□ redo_log_type占用1字节，表示重做日志的类型
□ space表示表空间的ID，但采用压缩的方式，因此占用的空间可能小于4字节
□ page_no表示页的偏移量，同样采用压缩的方式
□ redo_log_body表示每个重做日志的数据部分，恢复时需要调用相应的函数进行解析

undo log
undo log存放在一个叫segment的特殊段中，位于共享表空间中。

### AHI(Adaptive Hash Index)
innodb_adaptive_hash_index，默认启用

刷新相邻脏页，机械硬盘性能提升明显
innodb_flush_nerghbors=0可禁用

## 慢查询
long_query_time=10，默认10秒
log_slow_queries=ON，开启慢查询
log_queries_not_using_indexes=ON
log_throttle_not_using_indexes=0，表示每分钟允许记录到slow log的且未使用索引的SQL语句次数。该值默认为0，表示没有限制
long_query_io 将超过指定逻辑IO次数的SQL语句记录到slow log中。该值默认为100
slow_query_type 表示启用slow log的方式，可选值为0-3
0表示不将SQL语句记录到slow log
□ 1表示根据运行时间将SQL语句记录到slow log
□ 2表示根据逻辑IO次数将SQL语句记录到slow log
□ 3表示根据运行时间及逻辑IO次数将SQL语句记录到slow log

mysqldumpslow slow.log
mysqlbinlog -vv --start-position=100 bin.log

innodb_page_size 默认16k，可设置为4,8k

Compact行记录格式
变长字段列表 null标志位 记录头信息(是否已删除，下一条记录的相对位置等)  rowid，事务id，回滚指针，列1.2.3数据

在线对一个表创建普通索引，会对表加共享锁，导致不能写。
在线对一个表创建和删除主键索引，是通过临时表的形式进行的，导致不能读写。

innodb_online_alter_log_max_size  online ddl时写操作的缓存大小，如果期间写操作较多，超出限制，则抛出异常

## 查询计划
Cardinality，值越大，索引的区分度越好，Cardinality/表中行的数量应尽量等于1。
show index from table
采用采样统计，每次随机采样8个页，更新策略为:
表中1/16的数据已发生过变化。
stat_modified_counter>2000 000000。innodb_stats_sample_pages=8，设置对Cardinality采样统计的页数，默认8
innodb_stats_method=nulls_equal/nulls_unequal/nulls_ignored，表示统计是如何对待null值，默认相等，也可以设置不等，忽略等情况

Multi range read(MRR),将多个随机读合并为顺序读，执行计划中extra列有显示using mrr

Index condition pushdown(ICP)，在根据索引查询时，将where条件的部分过滤操作放在了存储引擎层。在某些查询下，可以大大减少上层SQL层对记录的索取（fetch），从而提高数据库的整体性能。WHERE可以过滤的条件是要该索引可以覆盖到的范围，支持range、ref、eq_ref、ref_or_null类型的查询。执行计划的列Extra看到Using index condition提示。

全文索引:倒排索引，不支持没有delimiter的语言，如中文，日文等。一张表只能建一个全文索引。查询计划的type列显示fulltext。

innodb_autoinc_lock_mode=012，主键自增默认，默认1，自增可以采用表锁或者或者基于内存的互斥量自增。0表示表锁，效率低，2表示内存互斥量，效率高，1:在bulk insert采用表锁，simple insert采用内存互斥量

加锁规则:使用等值查询时，若是唯一(主键）索引，则加record lock; 若是非唯一索引，则next key lock，还会对该值的下一个值加一个gap lock。
在InnoDB存储引擎中，对于INSERT的操作，其会检查插入记录的下一条记录是否被锁定，若已经被锁定，则不允许插入。

innodb_lock_wait_timeout=50，默认50秒锁超时时间
innodb_rollback_on_timeout=off，在锁超时后，默认不回滚事务
事务在锁超时后，既不会自动rollback，也不会自动commit，需要手动操作。

innodb_purge_batch_size=300，innodb1.2之前默认20，1.2之后默认300，表示每次purge时回收的undo页。
innodb_max_purge_lag，控制history list的长度，若长度大于该参数时，其会“延缓”DML的操作
innodb_max_purge_lag_delay，其用来控制delay的最大毫秒数。也就是当上述计算得到的delay值大于该参数时，将delay设置为innodb_max_purge_lag_delay，避免由于purge操作缓慢导致其他SQL线程出现无限制的等待。

binlog_max_flush_queue_time，用来控制事务提交时，flush阶段等待的最大时间，用于group commit(一次flush提交多个事务），提高性能，默认0。

## 备份
冷备:备份MySQL数据库的frm文件，共享表空间文件，独立表空间文件（*.ibd），重做日志文件，还有配置文件my.cnf
逻辑备份:mysqldump，select * into file
binlog,恢复mysqlbinlog binlog_file
热备:ibbackup收费.在线备份，不阻塞线上sql执行。复制物理文件，效率高。
1）记录备份开始时，InnoDB存储引擎重做日志文件检查点的LSN。
2）复制共享表空间文件以及独立表空间文件。
3）记录复制完表空间文件后，InnoDB存储引擎重做日志文件检查点的LSN。
4）复制在备份时产生的重做日志。
XtraBackup:免费。

## 基准测试,编译调试
基准测试工具：sysbench和mysql-tpcc。
innodb编译调试