（1）连接操作命令

quit：关闭连接（connection）

auth：简单密码认证

help cmd： 查看cmd帮助

（2）持久化

save：将数据同步保存到磁盘

bgsave：将数据异步保存到磁盘

lastsave：返回上次成功将数据保存到磁盘的Unix时戳

shundown：将数据同步保存到磁盘，然后关闭服务

（3）远程服务控制

info：提供服务器的信息和统计

monitor：实时转储收到的请求

slaveof：改变复制策略设置

config：在运行时配置Redis服务器

（4）对value操作的命令

exists(key)：确认一个key是否存在

del(key)：删除一个key

type(key)：返回值的类型

keys(pattern)：返回满足给定pattern的所有key

randomkey：随机返回key空间的一个key

rename(oldname, newname)：重命名key

dbsize：返回当前数据库中key的数目

expire：设定一个key的活动时间（s）

ttl：获得一个key的活动时间

select(index)：按索引查询

move(key, dbindex)：移动当前数据库中的key到dbindex数据库

flushdb：删除当前选择数据库中的所有key

flushall：删除所有数据库中的所有key



### **keys**

`keys`命令的作用是列出`Redis`所有的`key`,该命令的时间复杂度为**O(N)**，**N**随着`Redis`中`key`的数量增加而增加，因此`Redis`有大量的key，`keys`命令会执行很长时间，而由于`Redis`是单线程，某个命令耗费过长时间，则会导致后面的的所有请求无法得到响应，因此，千万不要在生产服务器上使用`keys`命令。

```text
# key命令,时间复杂度为O(n)
keys pattern #pattern可为一个包含匹配模式的字符串，可以包含*,+,?,[a-z]等模式。
```



### **exists**

`exists`命令用于判断一个或多个key是否存在，判断多个`key`时，`key`之间用空格分隔,`exists`的返回值为整数，表示当前判断有多少个`key`是存在的。

```text
# exists命令，时间复杂度O(1)
exists key [key ...]
```



### **del**

`del`命令用于删除一个或多个`key`，多个`key`之间用空格分隔，其返回值为整数，表示成功删除了多少个存在的`key`，因此，如果只删除一个key，则可以从返回值中判断是否成功，如果删除多个key，则只能得到删除成功的数量。

```text
# del命令,时间复杂度O(n)
del key [key ...]
```



### **expire,pexpire**

expire设置key在多少秒之后过期，pexpire设置key在多少毫秒之后过期,成功返回1，失败返回0。

```text
# expire命令，时间复杂度为O(1)
expire key seconds

# pexpire命令，时间复杂度为O(1)
pexpire key milliseconds
```



### **ttl,pttl**

ttl和pttl命令用于获取key的过期时间，其返回值为整型，代表的意义分为几种情况：

1. 当key不存在或过期时间，返回-2。
2. 当key存在且永久有效时，返回-1。
3. 当key有设置过期时间时，返回为剩下的秒数(pttl为毫秒数)

```text
# ttl命令，时间复杂度O(1)
ttl key

# pttl命令，时间复杂度O(1)
pttl key
```



### **expireat,pexpireat**

设置key在某个时间戳过期,expreat参数时间戳用秒表示，而pexpireat则用毫秒表示，与expire和pexpire功能类似，返回1表示成功，0表示失败。

```text
#expireat命令，时间复杂度为O(1)
expireat key timestamp

#pexpireat命令，时间复杂度为O(1)
pexpireat key milliseconds-timestamp
```



### **persist**

移除key的过期时间，将key设置为永久有效，当key设置了过期时间，使用persist命令移除后返回1，如果key不存在或本身就是永久有效的，则返回0。

```text
# persist命令,时间复杂度O(1)
persist key
```



### **type**

判断key是什么类型的数据结构,返回值为`string`,`list`,`set`,`hash`,`zset`,分别表示我们前面介绍的`Redis`的5种基础数据结构。

> geo,hyperloglog,bitmaps等复杂的数据结构，都是在这五种基础数据结构上实现，比如geo是zset类型，hyperloglog和bitmaps都为string。

```text
![v2-c5d893d6717b5b976974db1c0f89708e_r](E:\学习资料\TyporaImg\v2-c5d893d6717b5b976974db1c0f89708e_r.jpg)# type命令，时间复杂度O(1)
type key
```



#### **Redis中的自增命令和自减命令**

![v2-88e916245bec9ca58f9ad803c60de4d8_r](E:\学习资料\TyporaImg\v2-88e916245bec9ca58f9ad803c60de4d8_r.jpg)











































































