# MySQL 索引

[![img](https://upload.jianshu.io/users/upload_avatars/5114528/3c67dd2b-1f2b-47ee-8baf-bcc801189150.jpeg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/6fb3be4add8d)

[PC_Repair](https://www.jianshu.com/u/6fb3be4add8d)关注

12019.02.20 16:50:14字数 2,863阅读 3,899

# MySQL 索引

数据库索引的原理：数据库索引，是数据库管理系统中一个排序的数据结构，以协助快速查询、更新数据库表中数据。索引的实现通常使用 **BTree** 及其变种 **B+Tree**。

本文从如何建立 MySQL 索引以及介绍 MySQL 的索引类型，再讲 MySQL 索引的利与弊，以及建立索引时需要注意的地方。

首先:先假设有一张表,表的数据有 10W 条数据,其中有一条数据是 `nickname='css'` ,如果要拿这条数据的话需要些的 sql 是



```mysql
 SELECT * FROM award WHERE nickname = 'css';
```

一般情况下,在没有建立索引的时候, mysql 需要扫描全表及扫描 10W 条数据找这条数据,如果我在 nickname 上建立索引,那么mysql只需要扫描一行数据及为我们找到这条 nickname='css' 的数据,是不是感觉性能提升了好多咧....

mysql 的索引分为 单例索引（主键索引、唯一索引、普通索引）和 组合索引。

- 单例索引：一个索引只包含一个列，一个表可以有多个单例索引。
- 组合索引：一个组合索引包含两个或两个以上的列。

本文使用的案例的表：



```mysql
CREATE TABLE `award` (
   `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '用户id',
   `aty_id` varchar(100) NOT NULL DEFAULT '' COMMENT '活动场景id',
   `nickname` varchar(12) NOT NULL DEFAULT '' COMMENT '用户昵称',
   `is_awarded` tinyint(1) NOT NULL DEFAULT 0 COMMENT '用户是否领奖',
   `award_time` int(11) NOT NULL DEFAULT 0 COMMENT '领奖时间',
   `account` varchar(12) NOT NULL DEFAULT '' COMMENT '帐号',
   `password` char(32) NOT NULL DEFAULT '' COMMENT '密码',
   `message` varchar(255) NOT NULL DEFAULT '' COMMENT '获奖信息',
   `created_time` int(11) NOT NULL DEFAULT 0 COMMENT '创建时间',
   `updated_time` int(11) NOT NULL DEFAULT 0 COMMENT '更新时间',
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COMMENT='获奖信息表';
 
INSERT INTO `award` (`nickname`, `account`, `message`, `created_time`)
VALUES ('rSUQFzpkDz3R', 'DYxJoqZq2rd7', 'aaabbbccccxuxuxuxuuxux', 1449567822);
```

## (一) 索引的创建

**1. 单例索引**

1.1）普通索引，这个是最基本的索引。

其 sql 格式是 :



```mysql
CREATE INDEX IndexName ON `TableName`(`字段名`(length));
# 或者
ALTER TABLE TableName ADD INDEX IndexName(`字段名`(length));
```

第一种方式：



```mysql
CREATE INDEX account_Index ON `award`(`account`);
```

第二种方式：



```mysql
ALTER TABLE award ADD INDEX account_Index(`account`);
```

如果是 CHAR , VARCHAR 类型, length 可以小于字段的实际长度,如果是BLOB和TEXT类型就必须指定长度。

1.2）唯一索引，与普通索引类似，但是不同的是唯一索引要求所有的列的值是唯一的，这一点和主键索引一样，但是它允许有空值。

其 sql 格式是：



```mysql
CREATE UNIQUE INDEX IndexName ON `TableName`(`字段名`(length));
# 或者
ALTER TABLE TableName ADD UNIQUE (column_list); 
```

第一种方式：



```mysql
CREATE UNIQUE INDEX account_UNIQUE_Index ON `award`(`account`);
```

1.3）主键索引，不允许有空值,（在 B+Tree 中的 InnoDB 引擎中，主键索引起到了至关重要的位置）

主键索引建立的规则是 int 优于 varchar，一般在建表的时候创建，最好是与表的其他字段不相关的列或者是业务不相关的列。一般会设为 int 而且是 AUTO_INCREMENT 自增类型的。

**2. 组合索引**

一个表中含有多个单例索引不代表是组合索引，通俗一点讲，组合索引是：包含多个字段但是只有索引名称。

其 sql 格式是：



```mysql
CREATE INDEX nickname_account_createdTime_Index ON `award`(`nickname`, `account`, `created_time`);
```

![img](https://upload-images.jianshu.io/upload_images/5114528-706c6315f0abee2e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

查看索引.png

如果你建立了组合索引 (nickname_account_createdTime_Index)，那么它实际包含的是 3 个索引 (nickname) (nickname,account)(nickname,account,created_time)

在使用查询的时候遵循 mysql 组合索引的 “最左前缀”，即索引 where 时的条件要按照建立索引的时候字段的排列方式。

1、不按索引最左列开始查询（多列索引） 例如



```mysql
 index(‘c1’, ‘c2’, ‘c3’) where c2 = 'aaa' # 不使用索引
 where c2 = 'aaa' and c3='sss' # 不能使用索引
```

2、查询某个列有范围查询，则其右边的所有列都无法使用查询（多列查询）



```mysql
Where c1= 'xxx' and c2 like 'aa%' and c3='sss'
# 该查询只会使用索引中的前两列,因为like是范围查询
```

3、不能跳过某个字段来进行查询，这样利用不到索引，比如：



```mysql
EXPLAIN SELECT * FROM `award` WHERE nickname > 'rSUQFzpkDz3R' AND account = 'DYxJoqZq2rd7' AND created_time = 1449567822; 
```

因为我的索引是 (nickname, account, created_time)，如果第一个字段出现 范围符号 的查找，那么将不会用到索引，如果我是第二个或者第三个字段使用范围符号的查找，那么它会利用索引，利用的索引是 (nickname)，因为上面说了建立组合索引 (nickname, account, created_time)，会出现三个索引。

![img](https://upload-images.jianshu.io/upload_images/5114528-e89fefe045067907.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

组合索引.png

*注：使用组合检索的时候可能需要把前面创建的单例检索删除，否则可能会使用单例检索*

**3.全文索引**

文本字段上（text）如果建立的是普通索引，那么只有对文本的字段内容前面的字符进行索引，其字符大小根据索引建立索引时声明的大小来规定。

如果文本中出现多个一样的字符，而且需要查找的话，那么其条件只能是 `where column like '%xxxx%'`， 这样做会让索引失效

这个时候全文索引就有作用了



```mysql
ALTER TABLE TableName ADD FULLTEXT(column1, column2);
```



```mysql
ALTER TABLE `award` ADD FULLTEXT(`message`);
```

有了全文索引，就可以用 SELECT 查询命令去检索那些包含着一个或多个给定单词的数据记录了。



```mysql
SELECT * FROM TableName WHERE MATCH(column1, column2) AGAINST('xxx', 'sss', 'ddd');
```



```mysql
 EXPLAIN SELECT * FROM `award` WHERE MATCH(message) AGAINST('aaa');
```

上述命令将把 column1 和 column2 字段里有 xxx、sss、和 ddd 的数据记录全部查询出来。

![img](https://upload-images.jianshu.io/upload_images/5114528-1940bc1006952c47.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

全文检索.png

## (二) 索引的删除

- 查询索引



```mysql
SHOW INDEX FROM TableName;
```



```mysql
SHOW INDEX FROM award;
```

- 删除索引



```mysql
DROP INDEX IndexName ON `TableName`;
```

## (三) 使用索引的优点

1）可以通过建立唯一索引或者主键索引，保证数据库表中每一行数据的唯一性

2）建立索引可以大大提高检索的数据，以及减少表的检索行数

3）在表连接的连接条件，可以加速表与表直接的相连

4）在分组和排序字句进行数据检索，可以减少查询时间中 分组 和 排序 所消耗的时间（数据库的记录会重新排序）

5）建立索引，在查询中使用索引，可以提高性能

## (四) 使用索引的缺点

1）创建索引和维护索引会消耗时间，随着数据量的增加而增加

2）索引文件会占用物理空间，除了数据表需要占用物理空间之外，每一个索引还会占用一定的物理空间

3）当对表的数据进行 INSERT,UPDATE,DELETE 的时候,索引也要动态的维护,这样就会降低数据的维护速度,(建立索引会占用磁盘空间的索引文件。一般情况这个问题不太严重，但如果你在一个大表上创建了多种组合索引，索引文件的会膨胀很快)。

## (五) 使用索引需要注意的地方

在建立索引的时候应该考虑索引该建立在数据库表中的某些列上，哪一些索引需要建立，哪一些索引是多余的，一般来说：

1）在经常需要搜索的列上,可以加快索引的速度

2）主键列上可以确保列的唯一性

3）在表与表的连接条件上加上索引,可以加快连接查询的速度

4）在经常需要排序 (order by) ,分组 (group by) 和的 distinct 列上加索引 可以加快排序查询的时间, (单独order by 用不了索引，索引考虑加where 或加limit)

5）在一些 where 之后的 < <= > >= BETWEEN IN 以及某个情况下的like 建立字段的索引(B-TREE)

6）like语句，前导模糊查询 like "%XXX" 不能使用索引，而非前导模糊查询 like "XXX%" 则可以

7）索引不会包含 NULL 列,如果列中包含 NULL 值都将不会被包含在索引中,复合索引中如果有一列含有NULL值那么这个组合索引都将失效,一般需要给默认值0或者 ' ' 字符串

8）使用短索引,如果你的一个字段是 Char(32) 或者 int(32) ,在创建索引的时候指定前缀长度 比如前10个字符 (前提是多数值是唯一的..)那么短索引可以提高查询速度,并且可以减少磁盘的空间,也可以减少I/0操作.

9）不要在列上进行运算,这样会使得mysql索引失效,也会进行全表扫描

10）选择越小的数据类型越好,因为通常越小的数据类型通常在磁盘,内存,cpu,缓存中 占用的空间很少,处理起来更快

## (六) 什么情况下不创建索引

1）查询中很少使用到的列不应该创建索引,如果建立了索引然而还会降低mysql的性能和增大了空间需求.

2）很少数据的列也不应该建立索引,比如 一个性别字段 0或者1,在查询中,结果集的数据占了表中数据行的比例比较大,mysql需要扫描的行数很多,增加索引,并不能提高效率

3）定义为 text 和 image 和 bit 数据类型的列不应该增加索引

4）当表的修改(UPDATE,INSERT,DELETE)操作远远大于检索(SELECT)操作时不应该创建索引,这两个操作是互斥的关系

## (七) MySQL 优化之道

**一些常见的 SQL 实践**

- 负向条件不能使用索引



```mysql
select from order where status!=0 and status!=1
```



```mysql
not in/not exists # 都不是好习惯
```

可以优化为 `in` 查询：



```mysql
select from order where status in(2,3)
```

- 前导模糊查询不能使用索引



```mysql
select from order where desc like '%XX'
```

而非前导模糊查询则可以：



```mysql
select from order where desc like 'XX%'
```

- 数据区分度不大的字段不宜使用索引



```mysql
select from user where sex=1
```

原因：性别只有男，女，每次过滤掉的数据很少，不宜使用索引。

经验上，能过滤80%数据时就可以使用索引。对于订单状态，如果状态值很少，不宜使用索引，如果状态值很多，能够过滤大量数据，则应该建立索引。

- 在属性上进行计算不能命中索引



```mysql
select from order_ where YEAR(date) <= '2017'
```

即使 date 上建立了索引，也会全表扫描，可优化为值计算：



```mysql
select from order_ where date <= CURDATE()
```

或者：



```mysql
select from order_ where date < = '2017-01-01'
```

**并非周知的 SQL 实践**

- 如果业务大部分是单条查询，使用 Hash 索引性能更好，例如用户中心



```mysql
select from `user` where uid = ?
select from user where login_name=?
```

原因：`B-Tree`索引的时间复杂度是 `O(log(n))`；`Hash` 索引的时间复杂度是 `O(1)`

- 允许 `null`的列，查询有潜在大坑

单列索引不存 `null` 值，复合索引不存全为 `null` 的值，如果列允许为 `null`，可能会得到“不符合预期”的结果集



```mysql
select from user where name != 'shenjian'
```

如果 `name` 允许为 `null`，索引不存储`null`值，结果集中不会包含这些记录。所以，请使用 `not null` 约束以及默认值。

- 复合索引最左前缀，并不是指 SQL 语句的 `where`顺序要和符合索引一致。

用户中心建立了 (login_name, password) 的符合索引



```mysql
select from user where login_name=? and passwd=?
select from user where passwd=? and login_name=?
```

都能够命中索引



```mysql
select from user where login_name=?
```

也能命中索引，满足符合索引最左前缀



```mysql
select from user where passwd=?
```

不能命中索引，不满足符合索引最左前缀。

- 使用 `ENUM` 而不是字符串

`ENUM` 保存的是 `TINYINT`，别在枚举中搞一些“中国”“北京”“技术部”这样的字符串，字符串空间又大，效率又低。

**小众但有用的 SQL 实践**

- 如果明确知道只有一条结果返回，`limit 1` 能够提高效率



```mysql
select from user where login_name=?
```

可以优化为：



```mysql
select from user where login_name=? limit 1
```

原因：你知道只有一条结果，但数据库并不知道，明确告诉它，让它主动停止游标移动

- 把计算放到业务层而不是数据库层，除了节省数据的 CPU，还有意想不到的查询缓存优化效果



```mysql
select from order where date < = CURDATE()
```

这不是一个号的 SQL 实践，应该优化为：



```mysql
$curDate = date('Y-m-d');
$res = mysqlquery('select from order where date < = $curDate');
```

原因：释放了数据库的 CPU，多次调用，传入的SQL相同，才可以利用查询缓存

- 强制类型转换会全表扫描



```mysql
select from user where phone=13800001234
```