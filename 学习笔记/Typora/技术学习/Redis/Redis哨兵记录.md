

#端口port 26379
#后台启动daemonize yes
#运行时PID文件pidfile /var/run/redis-sentinel.pid
#日志文件(绝对路径)logfile "/opt/app/redis6/sentinel.log"
#数据目录dir /tmp/sentinel_26379
#监控的节点名字可以自定义，后边的2代表的：如果有俩个哨兵判断这个主节点挂了那这个主节点就挂了，通常设置为哨兵个数一半加一sentinel monitor mymaster 127.0.0.1 6379 2
#哨兵连接主节点多长时间没有响应就代表主节点挂了，单位毫秒。默认30000毫秒，30秒。sentinel down-after-milliseconds mymaster 30000
#在故障转移时，最多有多少从节点对新的主节点进行同步。这个值越小完成故障转移的时间就越长，这个值越大就意味着越多的从节点因为同步数据而暂时阻塞不可用sentinel parallel-syncs mymaster 1
#在进行同步的过程中，多长时间完成算有效，单位是毫秒，默认值是180000毫秒，3分钟。sentinel failover-timeout mymaster 180000
#禁止使用SENTINEL SET设置notification-script和client-reconfig-scriptsentinel deny-scripts-reconfig yes

