# Docker-ElasticSearch

1、下载镜像

```shell
docker pull elasticsearch:7.7.0
```



2、设置系统变量

编辑文档

```shell
vim /etc/sysctl.conf
```

文档末尾追加

```shell
vm.max_map_count=655360
```

刷新配置

```shell
sysctl -p
```





3、创建挂载目录

```shell
mkdir /usr/local/elasticsearch /usr/local/elasticsearch/config /usr/local/elasticsearch/data /usr/local/elasticsearch/logs /usr/local/elasticsearch/plugins
```

添加权限

```shell
chmod -R 777 /usr/local/elasticsearch
```



4、编写配置文件

```
vim /usr/local/elasticsearch/config/elasticsearch.yml
```

下面的配置建议你在`win`下创建编辑好，直接传到指定目录。不熟悉配置文件的话，建议你只改动`IP`，能跑就行不是吗。

节点一

```shell
# 设置集群名称，集群内所有节点的名称必须一致。
cluster.name: es-cluster
# 设置节点名称，集群内节点名称必须唯一。
node.name: node1
# 表示该节点会不会作为主节点，true表示会；false表示不会
node.master: true
# 当前节点是否用于存储数据，是：true、否：false
node.data: true
# 索引数据存放的位置
path.data: /usr/share/elasticsearch/data
# 日志文件存放的位置
path.logs: /usr/share/elasticsearch/logs
# 需求锁住物理内存，是：true、否：false，true的话启动会报错，而且开发环境也不会有很多数据导致内存交换
bootstrap.memory_lock: false
# 监听地址，用于访问该es
network.host: 192.168.200.131
# es对外提供的http端口，默认 9200
http.port: 9200
# TCP的默认监听端口，默认 9300
transport.tcp.port: 9300
# 设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。默认为1，对于大的集群来说，可以设置大一点的值（2-4）
discovery.zen.minimum_master_nodes: 1
# es7.x 之后新增的配置，写入候选主节点的设备地址，在开启服务后可以被选为主节点，不用填写端口号
discovery.seed_hosts: ["192.168.200.131", "192.168.200.132","192.168.200.133"]
discovery.zen.fd.ping_timeout: 1m
discovery.zen.fd.ping_retries: 5
# es7.x 之后新增的配置，初始化一个新的集群时需要此配置来选举master
cluster.initial_master_nodes: ["192.168.200.131", "192.168.200.132","192.168.200.133"]
# 是否支持跨域，是：true，在使用head插件时需要此配置
http.cors.enabled: true
# “*” 表示支持所有域名
http.cors.allow-origin: "*"
```

节点二

```shell
# 设置集群名称，集群内所有节点的名称必须一致。
cluster.name: es-cluster
# 设置节点名称，集群内节点名称必须唯一。
node.name: node2
# 表示该节点会不会作为主节点，true表示会；false表示不会
node.master: true
# 当前节点是否用于存储数据，是：true、否：false
node.data: true
# 索引数据存放的位置
path.data: /usr/share/elasticsearch/data
# 日志文件存放的位置
path.logs: /usr/share/elasticsearch/logs
# 需求锁住物理内存，是：true、否：false
bootstrap.memory_lock: false
# 监听地址，用于访问该es
network.host: 192.168.200.132
# es对外提供的http端口，默认 9200
http.port: 9200
# TCP的默认监听端口，默认 9300
transport.tcp.port: 9300
# 设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。默认为1，对于大的集群来说，可以设置大一点的值（2-4）
discovery.zen.minimum_master_nodes: 1
# es7.x 之后新增的配置，写入候选主节点的设备地址，在开启服务后可以被选为主节点
discovery.seed_hosts: ["192.168.200.131", "192.168.200.132","192.168.200.133"]
discovery.zen.fd.ping_timeout: 1m
discovery.zen.fd.ping_retries: 5
# es7.x 之后新增的配置，初始化一个新的集群时需要此配置来选举master
cluster.initial_master_nodes: ["192.168.200.131", "192.168.200.132","192.168.200.133"]
# 是否支持跨域，是：true，在使用head插件时需要此配置
http.cors.enabled: true
# “*” 表示支持所有域名
http.cors.allow-origin: "*"
```

节点三

```shell
# 设置集群名称，集群内所有节点的名称必须一致。
cluster.name: es-cluster
# 设置节点名称，集群内节点名称必须唯一。
node.name: node3
# 表示该节点会不会作为主节点，true表示会；false表示不会
node.master: true
# 当前节点是否用于存储数据，是：true、否：false
node.data: true
# 索引数据存放的位置
path.data: /usr/share/elasticsearch/data
# 日志文件存放的位置
path.logs: /usr/share/elasticsearch/logs
# 需求锁住物理内存，是：true、否：false
bootstrap.memory_lock: false
# 监听地址，用于访问该es
network.host: 192.168.200.133
# es对外提供的http端口，默认 9200
http.port: 9200
# TCP的默认监听端口，默认 9300
transport.tcp.port: 9300
# 设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。默认为1，对于大的集群来说，可以设置大一点的值（2-4）
discovery.zen.minimum_master_nodes: 1
# es7.x 之后新增的配置，写入候选主节点的设备地址，在开启服务后可以被选为主节点
discovery.seed_hosts: ["192.168.200.131", "192.168.200.132","192.168.200.133"]
discovery.zen.fd.ping_timeout: 1m
discovery.zen.fd.ping_retries: 5
# es7.x 之后新增的配置，初始化一个新的集群时需要此配置来选举master
cluster.initial_master_nodes: ["192.168.200.131", "192.168.200.132","192.168.200.133"]
# 是否支持跨域，是：true，在使用head插件时需要此配置
http.cors.enabled: true
# “*” 表示支持所有域名
http.cors.allow-origin: "*"
```



5、创建容器

```shell
docker run  -d  --network=host --privileged=true  \
-e ES_JAVA_OPTS="-Xms1024m -Xmx1024m"  \
--restart=always  \
-e TAKE_FILE_OWNERSHIP=true --name elasticsearch \
-v /usr/local/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /usr/local/elasticsearch/data:/usr/share/elasticsearch/data \
-v /usr/local/elasticsearch/logs:/usr/share/elasticsearch/logs \
-v /usr/local/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
elasticsearch:7.7.0
```



6、查看安装结果

测试网址：[192.168.200.131:9200](http://192.168.200.131:9200/)、[192.168.200.132:9200](http://192.168.200.132:9200/)、[192.168.200.133:9200](http://192.168.200.133:9200/)



7、安装插件

```shell
# 进入容器
docker exec -it elasticsearch /bin/bash
# 切换到插件目录
cd /usr/share/elasticsearch/plugins/
# 下载指定的ik分词器，需要和es保持一致的版本
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.7.0/elasticsearch-analysis-ik-7.7.0.zip
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.7.0/elasticsearch-analysis-ik-7.7.0.zip
# 下载指定的pinyin分词器，需要和es保持一致的版本
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v7.7.0/elasticsearch-analysis-pinyin-7.7.0.zip
# 推出docker容器
exit | ctrl + d
# 重启容器
docker restart 容器名 | 容器ID
```







ERROR: Elasticsearch did not exit normally - check the logs at /usr/share/elasticsearch/logs/es-cluster.log

解：创建的目录添加最高权限（`chmod -R 777 /usr/local/elasticsearch`）







[es部署问题处理_DJ_Aholic的博客-CSDN博客](https://blog.csdn.net/D_J1224/article/details/112602746)

问题6： java.io.StreamCorruptedException: received HTTP response on transport port, ensure that transport port (not HTTP port) of a remote node is specified in the configuration
解决方法：查看elasticsearch.yml中集群配置，只用配置IP不用端口
