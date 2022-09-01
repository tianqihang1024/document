

















> 使用心得

redis 哨兵模式自动实现主从复制，但不是集群



> 使用中出现的错误

==io.lettuce.core.RedisCommandExecutionException: ERR This instance has cluster support disabled==

==radis 集群此实例已禁用集群支持==

解决办法：redis.conf 1171行的 \# cluster-enabled yes 注释取消掉



在集群模式下不允许复制指令

![在集群模式下不允许复制指令](E:\学习资料\TyporaImg\在集群模式下不允许复制指令.png)





==redis配置哨兵问题，当主库宕机后，不自动切换==

解决办法：这个是哨兵之间通信中断引起的，把原本的两个哨兵判断是否下线，改为1个就行

sentinel monitor mymaster 127.0.0.1 6379 1





redis.conf配置	

protected-mode yes（设置成：protected-mode no；保护模式关闭，如果你不关闭保护模式，启动哨兵的时候，无法正常运行)

