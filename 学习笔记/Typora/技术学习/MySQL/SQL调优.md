

## Explain

range：

ref：使用非唯一索引进行数据查找

eq_ref：使用唯一索引进行数据查找

const：表中至多有一个匹配行

system：表只有一行记录（等于系统表），这是const类型的特例，平时不会出现



## MySQL事务ID的分配时机

事务开启后的第一条SQL都会分配一个事务ID。只不过读操作分配的是一个很大的默认值，而写操作分配的是正常递增的事务ID。同一个事务内的select的trx_id是



事务的操作

```mysql
-- 开启事务，在30分时执行此句
START TRANSACTION;
-- 执行查询语句，分配假事务ID，在31分时执行此句
SELECT * FROM tx_test;
-- 查看当前活跃的事务信息，在32分时执行此句
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX;
-- 第一条更新语句，分配真正的事务ID，在33分时执行此句
UPDATE tx_test SET age = 99;
-- 查看当前活跃的事务信息，在34分时执行此句
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX;
```





