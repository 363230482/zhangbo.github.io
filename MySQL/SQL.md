```
CREATE TABLE IF NOT EXISTS `table_name` (
	`id` BIGINT(20) NOT NULL auto_increment comment = '自增主键',
	`version` INT(11) DEFAULT NULL comment = '版本号',
	`username` varchar(64) NOT NULL comment = '用户名',
	`record_time` datetime DEFAULT CURRENT_TIMESTAMP comment = '记录时间',
    primary key (id)
) ENGINE = INNODB DEFAULT CHARSET = utf8mb4 comment = '用户信息表';

ALTER TABLE table_name ADD COLUMN id BIGINT not null auto_increment PRIMARY KEY;  

ALTER TABLE table_name ADD COLUMN record_time datetime DEFAULT CURRENT_TIMESTAMP;

alter table table_name modify column column_name varchar(64) default null comment = '注释' after other_column_name;

alter table table_name add index index_name(column_name) using btree;
alter table table_name drop index index_name;

[https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html]
mysqldump -h host -P port -u user -p password --add-locks=0 --no-create-info --single-transaction --set-gtid-purged=off --result-file=/dump.sql --databases db1 db2 --tables db3 table3
## 1. –single-transaction 的作用是，在导出数据的时候不需要对表 db1.t 加表锁，而是使用 START TRANSACTION WITH CONSISTENT SNAPSHOT 的方法；
## 2. –add-locks 设置为 0，表示在输出的文件结果里，不增加" LOCK TABLES t WRITE;"
## 3. –no-create-info 的意思是，不需要导出表结构；
## 4. –set-gtid-purged=off 表示的是，不输出跟 GTID 相关的信息；
## 5. –result-file 指定了输出文件的路径，其中 client 表示生成的文件是在客户端机器上的。
## 6. -databases -B 指定需要导出的databases，多个以空格分隔
## 7. -tables 会覆盖-databases，后面可以跟table

## 将查询结果导出到mysql服务器的本地文件，文件路径受到系统参数`secure_file_priv`的控制；
## 如果设置为 empty，表示不限制文件生成的位置，这是不安全的设置；
   如果设置为一个表示路径的字符串，就要求生成的文件只能放在这个指定的目录，或者它的子目录；
   如果设置为 NULL，就表示禁止在这个 MySQL 实例上执行 select … into outfile 操作。
select * from mytest.t_test where id > 100 into outfile  ‘/path/t.csv’;

load data infile '/var/lib/mysql-files/t.csv' into table mytest.t_test2;
```
