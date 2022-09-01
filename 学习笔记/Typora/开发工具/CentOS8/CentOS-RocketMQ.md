[RocketMq系列02 安装集群 双主双从 docker-compose_漫步者TZ的博客-CSDN博客](https://blog.csdn.net/qq_31815507/article/details/123600046)

[RocketMQ-Docker安装 - 晓风残月的博客 (baiyp.ren)](https://baiyp.ren/RocketMQ-Docker安装.html)

[ElasticSearch基本使用总结 - 张小凯的博客 (jasonkayzk.github.io)](https://jasonkayzk.github.io/2022/06/14/ElasticSearch基本使用总结/)

[RocketMq-console-ng](http://192.168.71.128:8080/#/)



## 端口号描述

以配置的 `ListenPort` 为 `10911` 为例

| 端口号  | 作用                                                         | 描述                                                  |
| ------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| `9876`  | `nameserver` 对外暴露的端口，允许消费者和生产者连接          |                                                       |
| `10911` | `ListenPort`端口                                             | `Broker` 对外服务的监听端口                           |
| `10909` | `fastListen` 端口                                            | 在消费者或生产者中配置 `isVipChannel` 为 `false` 即可 |
| `10912` | `HA` 高可用端口，用于主从同步，为 `Master` 常见新的 `socket` 连接 | 若没有开放，则无法连接到 `Slave`                      |

1.   **端口号计算**
     -   `vip` 通道端口为 `ListenPort(10911-2)` = `10909`
     -   `HA` 高可用端口为 `ListenPort(10911+1)` = `10912`
2.   **端口号左右**
     -   `vip` 通道端口一般没什么作用
     -   而 `HA` 高可用端口用于主从集群时，创建 `Master` 和 `Slave` 之间的 `socket` 连接，故需要对外暴露
3.   **主从配置注意事项**
     -   `clusterName`和`brokerName`需要一致，才可以形成主备关系。
     -   `brokerId`为`0`以及`brokerRole`为`ASYNC_MASTER`或者`SYNC_MASTER`代表为`master`
     -   `broker`不为`0`以及`brokerRole`为`SLAVE`代表为`slave`



## 计划服务部署情况

-   `2`个`NameServer` **`rocketmq`镜像**
-   `2` 个`master`、 `2`个对应的`slave` **`rocketmq`镜像**
-   `1`个管理控制台 **`rocketmq-console-ng`镜像**

| 服务名称          | IP地址       | 监听端口号 | 暴漏端口      |
| ----------------- | ------------ | ---------- | ------------- |
| `nameserver-a`    | `172.18.0.3` | `9876`     | `9876`        |
| `nameserver-b`    | `172.18.0.4` | `9876`     | `9877`        |
| `broker-master-a` | `172.18.0.5` | `10909`    | `10909,10910` |
| `broker-master-b` | `172.18.0.6` | `10919`    | `10919,10920` |
| `broker-slave-a`  | `172.18.0.7` | `10911`    | `10911`       |
| `broker-slave-b`  | `172.18.0.8` | `10921`    | `10921`       |
| `console`         | `172.18.0.9` | `8080`     | `8080`        |



## 创建`RocketMQ`配置文件目录

创建配置文件目录

```shell
mkdir -p  /tmp/data/rocketmq/nameserver-a/logs
mkdir -p  /tmp/data/rocketmq/nameserver-b/logs
mkdir -p  /tmp/data/rocketmq/nameserver-a/store
mkdir -p  /tmp/data/rocketmq/nameserver-b/store
mkdir -p  /tmp/data/rocketmq/broker-master-a/logs
mkdir -p  /tmp/data/rocketmq/broker-master-b/logs
mkdir -p  /tmp/data/rocketmq/broker-master-a/store
mkdir -p  /tmp/data/rocketmq/broker-master-b/store
mkdir -p  /tmp/data/rocketmq/broker-slave-a/logs
mkdir -p  /tmp/data/rocketmq/broker-slave-b/logs
mkdir -p  /tmp/data/rocketmq/broker-slave-a/store
mkdir -p  /tmp/data/rocketmq/broker-slave-b/store
mkdir -p  /tmp/etc/rocketmq/broker-master-a
mkdir -p  /tmp/etc/rocketmq/broker-master-b
mkdir -p  /tmp/etc/rocketmq/broker-slave-a
mkdir -p  /tmp/etc/rocketmq/broker-slave-b
```

## 创建`Broker`配置文件

主要信息有：

1.  当前`Broker`对外暴露的端口号
2.  注册到`NameServer`的地址，看到这里有两个地址，说明`NameServer`也是集群部署。
3.  当前`Broker`的角色，是主还是从，这里表示是主。

配置文件注意事项：

1.  请搜索以下内容，并根据自身条件进行替换：**请替换成虚拟机的对外`IP`**
2.  可以的话希望你的`docker`只有`rocketmq`一个服务，不然可能存在`namesrvAddr`被占用的情况

### broker-master-a

```shell
# 创建配置文件
vim /tmp/etc/rocketmq/broker-master-a/broker.conf
```

```
# 配置内容如下
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
#namesrvAddr 地址 填写docker内网地址即可 
namesrvAddr=172.28.0.3:9876;172.28.0.4:9876
#启用自动创建主题
autoCreateTopicEnable=false
#这个很有讲究 如果是正式环境 这里一定要填写内网地址（安全）
#如果是用于测试或者本地这里建议要填外网地址，因为你的本地代码是无法连接到阿里云内网，只能连接外网。
#当前broker监听的IP
brokerIP1 = 请替换成虚拟机的对外IP
#存在broker主从时，在broker主节点上配置了brokerIP2的话,broker从节点会连接主节点配置的brokerIP2来同步。
brokerIP2 = 请替换成虚拟机的对外IP
#Broker 对外服务的监听端口
listenPort = 10909
#Broker角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
```



### broker-master-b

```shell
# 创建配置文件
vim /tmp/etc/rocketmq/broker-master-b/broker.conf
```

```
# 配置内容如下
brokerClusterName = DefaultCluster
brokerName = broker-b
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
#namesrvAddr 地址 填写docker内网地址即可
namesrvAddr=172.28.0.3:9876;172.28.0.4:9876
#启用自动创建主题
autoCreateTopicEnable=false
#这个很有讲究 如果是正式环境 这里一定要填写内网地址（安全）
#如果是用于测试或者本地这里建议要填外网地址，因为你的本地代码是无法连接到阿里云内网，只能连接外网。
#当前broker监听的IP
brokerIP1 = 请替换成虚拟机的对外IP
#存在broker主从时，在broker主节点上配置了brokerIP2的话,broker从节点会连接主节点配置的brokerIP2来同步。
brokerIP2 = 请替换成虚拟机的对外IP
#Broker 对外服务的监听端口
listenPort = 10919
#Broker角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
```



### broker-slave-a

```shell
# 创建配置文件
vim /tmp/etc/rocketmq/broker-slave-a/broker.conf
```

```
# 配置内容如下
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 1
deleteWhen = 04
fileReservedTime = 48
#namesrvAddr 地址 填写docker内网地址即可
namesrvAddr=172.28.0.3:9876;172.28.0.4:9876
#启用自动创建主题
autoCreateTopicEnable=false
#这个很有讲究 如果是正式环境 这里一定要填写内网地址（安全）
#如果是用于测试或者本地这里建议要填外网地址，因为你的本地代码是无法连接到阿里云内网，只能连接外网。
#当前broker监听的IP
brokerIP1 = 请替换成虚拟机的对外IP
#存在broker主从时，在broker主节点上配置了brokerIP2的话,broker从节点会连接主节点配置的brokerIP2来同步。
brokerIP2 = 请替换成虚拟机的对外IP
#Broker 对外服务的监听端口
listenPort = 10911
#Broker角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
```



### broker-slave-b

```shell
# 创建配置文件
vim /tmp/etc/rocketmq/broker-slave-b/broker.conf
```

```
# 配置内容如下
brokerClusterName = DefaultCluster
brokerName = broker-b
brokerId = 1
deleteWhen = 04
fileReservedTime = 48
#namesrvAddr 地址 填写docker内网地址即可
namesrvAddr=172.28.0.3:9876;172.28.0.4:9876
#启用自动创建主题
autoCreateTopicEnable=false
#这个很有讲究 如果是正式环境 这里一定要填写内网地址（安全）
#如果是用于测试或者本地这里建议要填外网地址，因为你的本地代码是无法连接到阿里云内网，只能连接外网。
#当前broker监听的IP
brokerIP1 = 请替换成虚拟机的对外IP
#存在broker主从时，在broker主节点上配置了brokerIP2的话,broker从节点会连接主节点配置的brokerIP2来同步。
brokerIP2 = 请替换成虚拟机的对外IP
#Broker 对外服务的监听端口
listenPort = 10921
#Broker角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
```



## 创建`Docker-Compose`文件

共7个容器

-   `2`个`NameServer` **`rocketmq`镜像**
-   `2` 个`master`、 `2`个对应的slave **`rocketmq`镜像**
-   `1`个管理控制台 **`rocketmq-console-ng`镜像**

```
version: '2.1'
services:
  nameserver-a:
    image: rocketmqinc/rocketmq
    container_name: nameserver-a
    networks:
      rocketmq_net:
        ipv4_address: 172.28.0.3
    environment:
      MAX_POSSIBLE_HEAP: 100000000
    ports:
      - 9876:9876
    volumes:
      - /tmp/data/rocketmq/nameserver-a/logs:/root/logs
      - /tmp/data/rocketmq/nameserver-a/store:/root/store
    command: sh mqnamesrv
  nameserver-b:
    image: rocketmqinc/rocketmq
    container_name: nameserver-b
    networks:
      rocketmq_net:
        ipv4_address: 172.28.0.4
    environment:
      MAX_POSSIBLE_HEAP: 100000000
    ports:
      - 9877:9876
    volumes:
      - /tmp/data/rocketmq/nameserver-b/logs:/root/logs
      - /tmp/data/rocketmq/nameserver-b/store:/root/store
    command: sh mqnamesrv
  broker-master-a:
    image: rocketmqinc/rocketmq
    container_name: rmqbroker-master-a
    networks:
      rocketmq_net:
        ipv4_address: 172.28.0.5
    environment:
      MAX_POSSIBLE_HEAP: 200000000
    ports:
      - 10909:10909
      - 10910:10910
    volumes:
      - /tmp/data/rocketmq/broker-master-a/logs:/root/logs
      - /tmp/data/rocketmq/broker-master-a/store:/root/store
      - /tmp/etc/rocketmq/broker-master-a/broker.conf:/opt/rocketmq/conf/broker.conf
    command: sh mqbroker -c /opt/rocketmq/conf/broker.conf
    depends_on:
      - nameserver-a
      - nameserver-b
  broker-master-b:
    image: rocketmqinc/rocketmq
    container_name: rmqbroker-master-b
    networks:
      rocketmq_net:
        ipv4_address: 172.28.0.6
    environment:
      MAX_POSSIBLE_HEAP: 200000000
    ports:
      - 10919:10919
      - 10920:10920
    volumes:
      - /tmp/data/rocketmq/broker-master-b/logs:/root/logs
      - /tmp/data/rocketmq/broker-master-b/store:/root/store
      - /tmp/etc/rocketmq/broker-master-b/broker.conf:/opt/rocketmq/conf/broker.conf
    command: sh mqbroker -c /opt/rocketmq/conf/broker.conf
    depends_on:
      - nameserver-a
      - nameserver-b
  broker-slave-a:
    image: rocketmqinc/rocketmq
    container_name: rmqbroker-slave-a
    networks:
      rocketmq_net:
        ipv4_address: 172.28.0.7
    environment:
      MAX_POSSIBLE_HEAP: 200000000
    ports:
      - 10911:10911
    volumes:
      - /tmp/data/rocketmq/broker-slave-a/logs:/root/logs
      - /tmp/data/rocketmq/broker-slave-a/store:/root/store
      - /tmp/etc/rocketmq/broker-slave-a/broker.conf:/opt/rocketmq/conf/broker.conf
    command: sh mqbroker -c /opt/rocketmq/conf/broker.conf
    depends_on:
      - nameserver-a
      - nameserver-b
      - broker-master-a
      - broker-master-b
  broker-slave-b:
    image: rocketmqinc/rocketmq
    container_name: rmqbroker-slave-b
    networks:
      rocketmq_net:
        ipv4_address: 172.28.0.8
    environment:
      MAX_POSSIBLE_HEAP: 200000000
    ports:
      - 10921:10921
    volumes:
      - /tmp/data/rocketmq/broker-slave-b/logs:/root/logs
      - /tmp/data/rocketmq/broker-slave-b/store:/root/store
      - /tmp/etc/rocketmq/broker-slave-b/broker.conf:/opt/rocketmq/conf/broker.conf
    command: sh mqbroker -c /opt/rocketmq/conf/broker.conf
    depends_on:
      - nameserver-a
      - nameserver-b
      - broker-master-a
      - broker-master-b
  console:
    image: styletang/rocketmq-console-ng
    container_name: rocketmq-console-ng
    networks:
      rocketmq_net:
        ipv4_address: 172.28.0.9
    ports:
      - 8080:8080
    depends_on:
      - nameserver-a
      - nameserver-b
    environment:
      - JAVA_OPTS= -Dlogging.level.root=info -Drocketmq.namesrv.addr=172.28.0.3:9876;172.28.0.4:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false
networks:
  rocketmq_net:
    ipam:
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1
```

## 启动容器

启动

```shell
docker-compose -f docker-compose-rocketmq-cluster.yml up -d
```

![46ace6e6677374005a8a168e7cd77eed](C:/Users/22489/OneDrive/%E7%94%B0%E5%A5%87%E6%9D%AD/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/TyporaImg/46ace6e6677374005a8a168e7cd77eed.png)

检查启动状态

```shell
docker-compose -f docker-compose-rocketmq-cluster.yml ps
```

![b457699023735f41c539a981718daa28](C:/Users/22489/OneDrive/%E7%94%B0%E5%A5%87%E6%9D%AD/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/TyporaImg/b457699023735f41c539a981718daa28.png)

访问控制台，在浏览器中输入：http://{docker宿主机ip}:8080/ 看到如下界面，表示安装成功

![ac35c1bfeb3ca9770ddb644cce5502ab](C:/Users/22489/OneDrive/%E7%94%B0%E5%A5%87%E6%9D%AD/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/TyporaImg/ac35c1bfeb3ca9770ddb644cce5502ab.png)

手动创建主题，因为创建双主双从，并且关闭了自动主题创建，需要手动创建主题

![b793d74473912e66fc676677ade3e3c9](C:/Users/22489/OneDrive/%E7%94%B0%E5%A5%87%E6%9D%AD/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/TyporaImg/b793d74473912e66fc676677ade3e3c9.png)









