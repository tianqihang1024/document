















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



















