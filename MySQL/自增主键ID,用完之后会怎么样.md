#MySQL自增主键，在使用完之后会怎么样呢

##新建测试表
```sql
CREATE TABLE `t_test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(16) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

##插入测试值
我们知道，int的最大值为`2147483647`，那我们先从2147483645开始插入测试数据
```sql
insert into t_test (id, name) values (2147483645, '1');
```
然后在使用自增插入的方式，继续插入测试数据
```sql
insert into t_test (name) values ('2');
insert into t_test (name) values ('3');
insert into t_test (name) values ('4');
```
我们发现，在插入name为3的时候，id已经自增到最大值2147483647，此时我们继续插入name为4的数据，则会提示
`1062 Duplicate entry '2147483647' for key PRIMARY`,告诉我们主键重复了。

那如果我们继续使用超过2147483647的值插入测试数据呢
```sql
insert into t_test (id, name) values (2147483648, '1');
```
此时，则会提示`1264 Out of range for column id at row 1`，告诉我们超过id的范围了。