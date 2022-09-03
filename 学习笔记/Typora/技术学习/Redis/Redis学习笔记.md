















### Zset

常用命令：

```
# 添加元素
zset key score name score name ...

# 返回对应的元素
zrange key 0 -1 withscores

# 从小到大的前两名
zrange key 0 1

# 从大到小的前两名
zrange key 
```



面试问题：

​	在动态插入数据是是否会对数据进行动态排序？	==否==

​	









elasticesearch 怎么优化索引？

​		我当时有三个物理节点，但是考虑到后期数据量上来，就做了超过节点数量的分区，这样后期添加数据，就只会从别的分区中拿过去一两个到新机器上，从而避免了那个无法分区的尴尬







什么是`lua`

`lua`是一个十分轻量的语言，一般都是集成到其他语言作为嵌套脚本。当然在`redis`中也有使用，并且在重要的场景起了大作用。你像`redis`每条命令都是单独执行，虽然有事务可以批量这些一些命令，但是不具备`DB`回滚效果。嵌套lua脚本后就能够解决这个问题。



怎么编写`lua`脚本呢

`lua`的简单语法

1.  `nil` 空
2.  `boolean` 布尔值
3.  `number` 数字
4.  `string` 字符串
5.  `table` 表

声明类型

声明类型非常简单，不用携带类型。

```lua
--- 全局变量 
name = 'felord.cn'
--- 局部变量
local age = 18
```

>   Redis脚本在实践中不要使用全局变量，局部变量效率更高。



判断

判断非常简单，格式为：

```lua
local a = 10
if a < 10  then
	print('a小于10')
elseif a < 20 then
	print('a小于20，大于等于10')
else
	print('a大于等于20')
end
```



返回值

可以通过`return`关键词即可，使用中建议返回`string、boolean、long`，其他类型的话需要考虑适配。

```lua
local a = 10
if a < 10  then
	print('a小于10')
elseif a < 20 then
	print('a小于20，大于等于10')
else
	print('a大于等于20')
end
```



两个常用的方法

tonumber()：将字符串转换为整型数字，浮型字符串转换时存在失真。因此不建议对其进行转化，而是字符串存储使用。

tostring()：将数字转换为字符串







### `lua`脚本中使用`zset`的`range`命令查询结果为`null`

```redis控制台
> EVAL "redis.call('zrange', KEYS[1], ARGV[1],ARGV[2])" 1 lock 0 1
null
```











