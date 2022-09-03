# `Docker-RocketMQ`单机版



## 拉取`rocketmq`镜像

```shell
docker pull rocketmqinc/rocketmq:4.4.0
```



## 创建`namesrv、broker`目录，`broker`配置文件

```shell
mkdir -p /usr/local/rocketmq/data/namesrv/logs /usr/local/rocketmq/data/namesrv/store
mkdir -p /usr/local/rocketmq/data/broker/logs /usr/local/rocketmq/data/broker/store /usr/local/rocketmq/conf
vim /usr/local/rocketmq/conf/broker.conf
```

```
# 所属集群名称，如果节点较多可以配置多个
brokerClusterName = DefaultCluster
#broker名称，master和slave使用相同的名称，表明他们的主从关系
brokerName = broker
#0表示Master，大于0表示不同的slave
brokerId = 0
#表示几点做消息删除动作，默认是凌晨4点
deleteWhen = 04
#启用自动创建主题
autoCreateTopicEnable=false
#Broker 对外服务的监听端口
listenPort = 10911
#在磁盘上保留消息的时长，单位是小时
fileReservedTime = 48
#有三个值：SYNC_MASTER，ASYNC_MASTER，SLAVE；同步和异步表示Master和Slave之间同步数据的机制；
brokerRole = ASYNC_MASTER
#刷盘策略，取值为：ASYNC_FLUSH，SYNC_FLUSH表示同步刷盘和异步刷盘；SYNC_FLUSH消息写入磁盘后才返回成功状态，ASYNC_FLUSH不需要；
flushDiskType = SYNC_FLUSH
# 设置broker节点所在服务器的ip地址
brokerIP1 = 81.70.96.232
# 磁盘使用达到95%之后,生产者再写入消息会报错 CODE: 14 DESC: service not available now, maybe disk full
diskMaxUsedSpaceRatio=95
```



## 创建`namesrv`容器

```shell
docker run -d \
--restart=always \
--name namesrv \
-p 9876:9876 \
--network docker_net --ip 172.18.0.9 \
-v /usr/local/rocketmq/data/namesrv/logs:/root/logs \
-v /usr/local/rocketmq/data/namesrv/store:/root/store \
-e "MAX_POSSIBLE_HEAP=268435456" \
rocketmqinc/rocketmq:4.4.0 \
sh mqnamesrv  

docker run -d \
--restart=always \
--name namesrv \
-p 9876:9876 \
-v /usr/local/rocketmq/data/namesrv/logs:/root/logs \
-v /usr/local/rocketmq/data/namesrv/store:/root/store \
-e "MAX_POSSIBLE_HEAP=268435456" \
rocketmqinc/rocketmq:4.4.0 \
sh mqnamesrv  
```

| 参数                                                  | 说明                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------ |
| -d                                                    | 以守护进程的方式启动                                         |
| --restart=always                                      | docker重启时候容器自动重启                                   |
| --name namesrv                                        | 把容器的名字设置为namesrv                                    |
| -p 9876:9876                                          | 把容器内的端口9876挂载到宿主机9876上面                       |
| --network docker_net --ip 172.18.0.7                  | **指定docker子网段IP（仅是本人为了规范，建议去掉）**         |
| -v /usr/local/rocketmq/data/namesrv/logs:/root/logs   | 把容器内的/root/logs日志目录挂载到宿主机的 /usr/local/rocketmq/data/namesrv/logs目录 |
| -v /usr/local/rocketmq/data/namesrv/store:/root/store | 把容器内的/root/store数据存储目录挂载到宿主机的 /usr/local/rocketmq/data/namesrv目录 |
| -e “MAX_POSSIBLE_HEAP=100000000”                      | 指定broker服务的最大堆内存（单位：字节，对应128M）           |
| rocketmqinc/rocketmq:4.4.0                            | 使用的镜像名称                                               |
| sh mqnamesrv                                          | 启动namesrv服务                                              |



## 创建`broker`容器

```shell
docker run -d  \
--restart=always \
--name broker \
--link namesrv:namesrv \
-p 10911:10911 \
-p 10909:10909 \
--network docker_net --ip 172.18.0.10 \
-v  /usr/local/rocketmq/data/broker/logs:/root/logs \
-v  /usr/local/rocketmq/data/broker/store:/root/store \
-v /usr/local/rocketmq/conf/broker.conf:/opt/rocketmq-4.4.0/conf/broker.conf \
-e "NAMESRV_ADDR=namesrv:9876" \
-e "MAX_POSSIBLE_HEAP=536870912" \
rocketmqinc/rocketmq:4.4.0 \
sh mqbroker -c /opt/rocketmq-4.4.0/conf/broker.conf

docker run -d  \
--restart=always \
--name broker \
--link namesrv:namesrv \
-p 10911:10911 \
-p 10909:10909 \
-v  /usr/local/rocketmq/data/broker/logs:/root/logs \
-v  /usr/local/rocketmq/data/broker/store:/root/store \
-v /usr/local/rocketmq/conf/broker.conf:/opt/rocketmq-4.4.0/conf/broker.conf \
-e "NAMESRV_ADDR=namesrv:9876" \
-e "MAX_POSSIBLE_HEAP=268435456" \
rocketmqinc/rocketmq:4.4.0 \
sh mqbroker -c /opt/rocketmq-4.4.0/conf/broker.conf
```

| 参数                                                         | 说明                                                 |
| ------------------------------------------------------------ | ---------------------------------------------------- |
| -d                                                           | 以守护进程的方式启动                                 |
| –restart=always                                              | docker重启时候镜像自动重启                           |
| --name broker                                                | 把容器的名字设置为broker                             |
| --link namesrv:namesrv                                       | 和namesrv容器通信                                    |
| -p 10911:10911                                               | 把容器的非vip通道端口挂载到宿主机                    |
| -p 10909:10909                                               | 把容器的vip通道端口挂载到宿主机                      |
| --network docker_net --ip 172.18.0.8                         | **指定docker子网段IP（仅是本人为了规范，建议去掉）** |
| -e “NAMESRV_ADDR=namesrv:9876”                               | 指定namesrv的地址为本机namesrv的ip地址:9876          |
| -e “MAX_POSSIBLE_HEAP=268435456” rocketmqinc/rocketmq sh mqbroker | 指定broker服务的最大堆内存（单位：字节，对应256M）   |
| rocketmqinc/rocketmq:4.4.0                                   | 使用的镜像名称                                       |
| sh mqbroker -c /opt/rocketmq-4.4.0/conf/broker.conf          | 指定配置文件启动broker节点                           |



## 创建`rocketmq-console-ng`控制台容器

控制台不能和`namesrv、broker`部署在同一台机器的`docker`中，应为`IP`问题会一直报错无法正常使用。鉴于本人只是用控制台创建`Topic`，所以直接另起了一个虚拟机部署控制台，正常访问

下载镜像

```shell
docker pull pangliang/rocketmq-console-ng
```

创建容器

```shell
docker run -d \
--restart=always \
--name rocketmq-console \
-e "JAVA_OPTS=-Drocketmq.namesrv.addr=81.70.96.232:9876 -Duser.timezone='Asia/Shanghai' \
-Dcom.rocketmq.sendMessageWithVIPChannel=false" \
-v /etc/localtime:/etc/localtime \
-p 8080:8080 \
pangliang/rocketmq-console-ng



docker run -d \
--restart=always \
--name rocketmq-console \
-e "-Duser.timezone='Asia/Shanghai'" \
-e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.200.131:9876 \
-Dcom.rocketmq.sendMessageWithVIPChannel=false" \
-p 8080:8080 \
pangliang/rocketmq-console-ng
```

| 参数                                                    | 说明                                       |
| ------------------------------------------------------- | ------------------------------------------ |
| -d                                                      | 以守护进程的方式启动                       |
| --restart=always                                        | docker重启时候镜像自动重启                 |
| --name rocketmq-console                                 | 把容器的名字设置为rocketmq-console         |
| -e "JAVA_OPTS=-Drocketmq.namesrv.addr=81.70.96.232:9876 | 设置namesrv服务的ip地址                    |
| -Dcom.rocketmq.sendMessageWithVIPChannel=false"         | 不使用vip通道发送消息                      |
| –p 8080:8080                                            | 把容器内的端口8080挂载到宿主机上的8080端口 |

## 访问控制台

http://容器所部署的IP:8080/#/





## 相关链接

[使用docker安装RocketMQ_皓亮君的博客-CSDN博客_docker rocketmq](https://blog.csdn.net/ming19951224/article/details/109063041)

[RocketMQ与SpringBoot整合进行生产级二次封装_TianXinCoord的博客-CSDN博客_rocketmq二次封装](https://blog.csdn.net/sinat_34104446/article/details/125355774)

[maven打包时去除不需要的jar包策略_micro_cloud_fly的博客-CSDN博客_maven打包排除依赖](https://blog.csdn.net/silk_java/article/details/45094019)
