



下载`RocketMQ`镜像

```shell
$ docker pull foxiswho/rocketmq:4.7.0
```



创建挂载目录

```shell
## 创建 Broker 持久化目录
$ mkdir -p /usr/local/rocketmq/broker/conf && \
  mkdir -p /usr/local/rocketmq/broker/logs && \
  mkdir -p /usr/local/rocketmq/broker/store 

## 创建 NameServer 持久化目录
$ mkdir -p /usr/local/rocketmq/server/logs
```



创建`Broker`配置文件

`n0`

```shell
$ cat > /usr/local/rocketmq/broker/conf/broker.conf

brokerIP1=192.168.200.131
listenPort=30911
brokerClusterName=RaftCluster
brokerName=RaftNode00
namesrvAddr=192.168.200.131:9876;192.168.200.132:9876;192.168.200.133:9876
## DLeger
dLegerSelfId=n0
dLegerGroup=RaftNode00
enableDLegerCommitLog=true
## must be unique
dLegerPeers=n0-192.168.200.131:40911;n1-192.168.200.132:40911;n2-192.168.200.133:40911
sendMessageThreadPoolNums=2
```



`n1`

```shell
$ cat > /usr/local/rocketmq/broker/conf/broker.conf

brokerIP1=192.168.200.131
listenPort=30911
brokerClusterName=RaftCluster
brokerName=RaftNode00
namesrvAddr=192.168.200.131:9876;192.168.200.132:9876;192.168.200.133:9876
## DLeger
dLegerSelfId=n0
dLegerGroup=RaftNode00
enableDLegerCommitLog=true
## must be unique
dLegerPeers=n0-192.168.200.131:40911;n1-192.168.200.132:40911;n2-192.168.200.133:40911
sendMessageThreadPoolNums=2
```



`n2`

```shell
$ cat > /usr/local/rocketmq/broker/conf/broker.conf

brokerIP1=192.168.200.131
listenPort=30911
brokerClusterName=RaftCluster
brokerName=RaftNode00
namesrvAddr=192.168.200.131:9876;192.168.200.132:9876;192.168.200.133:9876
## DLeger
dLegerSelfId=n0
dLegerGroup=RaftNode00
enableDLegerCommitLog=true
## must be unique
dLegerPeers=n0-192.168.200.131:40911;n1-192.168.200.132:40911;n2-192.168.200.133:40911
sendMessageThreadPoolNums=2
```





查看镜像设置的用户与组的配置

```shell
$ docker history foxiswho/rocketmq:4.7.0

IMAGE               CREATED             CREATED BY                                      SIZE 
1cf46e8f03d0        7 months ago        /bin/sh -c #(nop) WORKDIR /home/rocketmq/roc…   0B                  
<missing>           7 months ago        /bin/sh -c #(nop)  USER rocketmq                0B                  
<missing>           7 months ago        |5 gid=3000 group=rocketmq uid=3000 user=roc…   1.92kB              
<missing>           7 months ago        |5 gid=3000 group=rocketmq uid=3000 user=roc…   0B                  
<missing>           7 months ago        |5 gid=3000 group=rocketmq uid=3000 user=roc…   11.3kB              
<missing>           7 months ago        /bin/sh -c #(nop)  EXPOSE 10909 10911 10912     0B                  
<missing>           7 months ago        |5 gid=3000 group=rocketmq uid=3000 user=roc…   10.1kB              
<missing>           7 months ago        /bin/sh -c #(nop)  EXPOSE 9876                  0B                  
<missing>           7 months ago        |5 gid=3000 group=rocketmq uid=3000 user=roc…   15.1MB              
<missing>           7 months ago        /bin/sh -c #(nop) COPY dir:bdc4a8518539da6ce…   11.4kB              
<missing>           7 months ago        |5 gid=3000 group=rocketmq uid=3000 user=roc…   15.1MB              
<missing>           7 months ago        /bin/sh -c #(nop) WORKDIR /home/rocketmq/roc…   0B                  
<missing>           7 months ago        /bin/sh -c #(nop)  ENV ROCKETMQ_HOME=/home/r…   0B                  
<missing>           7 months ago        /bin/sh -c #(nop)  ENV ROCKETMQ_VERSION=4.7.0   0B                  
<missing>           7 months ago        /bin/sh -c #(nop)  ARG version=4.7.0            0B                  
<missing>           7 months ago        |4 gid=3000 group=rocketmq uid=3000 user=roc…   1.07MB              
<missing>           7 months ago        /bin/sh -c #(nop)  ARG gid=3000                 0B                  
<missing>           7 months ago        /bin/sh -c #(nop)  ARG uid=3000                 0B                  
<missing>           7 months ago        /bin/sh -c #(nop)  ARG group=rocketmq           0B                  
<missing>           7 months ago        /bin/sh -c #(nop)  ARG user=rocketmq            0B                  
<missing>           7 months ago        /bin/sh -c yum install -y java-1.8.0-openjdk…   264MB               
<missing>           11 months ago       /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
<missing>           11 months ago       /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B                  
<missing>           11 months ago       /bin/sh -c #(nop) ADD file:45a381049c52b5664…   203MB
```

可以观察到：

-   组名：`rocketmq`，`组ID`：`3000`
-   用户名：`rocketmq`，`用户ID`：`3000`



创建对应的用户与组

```shell
## 创建组
$ groupadd rocketmq

## 增加用户并加入组
$ useradd -g rocketmq rocketmq

## 设置用户密码
$ passwd rocketmq

## 更改组的 gid
$ groupmod -g 3000 rocketmq

## 更改用户的 uid
$ usermod -u 3000 rocketmq

## 查看是否更改成功
$ id rocketmq
```

三台服务器上分别更改上面创建的目录的权限为上面创建的组与用户：

```shell
$ chown -R rocketmq:rocketmq /usr/local/rocketmq
```





安装`RocketMQ NameServer`

```shell
docker run -d --name rmqnamesr --net host \
-v /usr/local/rocketmq/server/logs:/home/rocketmq/logs \
-e "JAVA_OPT_EXT=-Xms256M -Xmx256M -Xmn64m" \
-p 9876:9876 \
--restart=always \
foxiswho/rocketmq:4.7.0 \
sh mqnamesrv
```





安装`RocketMQ Broker`

```shell
docker run -d --name rmqbroker --net host \
-e "JAVA_OPT_EXT=-Xmx516m -Xms516m -Xmn128m" \
-v /usr/local/rocketmq/broker/logs:/home/rocketmq/logs \
-v /usr/local/rocketmq/broker/store:/home/rocketmq/store \
-v /usr/local/rocketmq/broker/conf:/home/rocketmq/conf \
-p 30909:30909 -p 30911:30911 -p 30912:30912 -p 40911:40911 \
--restart=always \
foxiswho/rocketmq:4.7.0 \
sh mqbroker -c /home/rocketmq/conf/broker.conf
```







安装控制台

下载`rocketmq-console`镜像

```shell
$ docker pull apacherocketmq/rocketmq-console:2.0.0
```

创建容器

```shell
docker run -d --name rmqconsole \
-p 8080:8080 \
--restart=always \
-e "JAVA_OPT_EXT=-Xms256M -Xmx256M -Xmn64m" \
-e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.200.131:9876;192.168.200.132:9876;192.168.200.133:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false" \
apacherocketmq/rocketmq-console:2.0.0
```



输入 http://192.168.200.130:8080 访问在服务器一部署的 RocketMQ 控制台，进入到如下界面：

![img](C:/Users/22489/OneDrive/%E7%94%B0%E5%A5%87%E6%9D%AD/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/TyporaImg/shuiyin.png)





原文地址：[通过 Docker 部署 RocketMQ Dledger 集群模式（ 版本v4.7.0） | 小豆丁技术栈 (mydlq.club)](http://www.mydlq.club/article/97/)