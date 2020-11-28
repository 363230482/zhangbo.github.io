#查询死锁
```sql
-- 查看当前连接情况。
show processlist;  
--查看InnoDB行锁情况。
select * from sys.innodb_lock_waits;
-- 查看表锁情况
select * from sys.schema_table_lock_waits;	
-- 然后根据锁的情况，适当的kill掉相关的连接，释放锁。
kill $id;
```

#长事务
`information_schema.INNODB_TRX`表中包含了当前innodb内部正在运行的事务信息  

