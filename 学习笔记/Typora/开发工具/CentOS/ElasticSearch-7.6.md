



切换到指定目录

```shell
cd /usr/local
```



下载（上传）安装包

```shell
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.2-linux-x86_64.tar.gz
```



解压

```shell
tar -zxvf elasticsearch-7.6.2-linux-x86_64.tar.gz
```



修改配置文件

```shell
# ======================== Elasticsearch Configuration =========================
#
# NOTE: Elasticsearch comes with reasonable defaults for most settings.
#       Before you set out to tweak and tune the configuration, make sure you
#       understand what are you trying to accomplish and the consequences.
#
# The primary way of configuring a node is via this file. This template lists
# the most important settings you may want to configure for a production cluster.
#
# Please consult the documentation for further information on configuration options:
# https://www.elastic.co/guide/en/elasticsearch/reference/index.html
#
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: wooola-es
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: node-1
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
#path.data: /path/to/data
#
# Path to log files:
#
#path.logs: /path/to/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
#bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: 0.0.0.0
#
# Set a custom port for HTTP:
#
#http.port: 9200
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
#discovery.seed_hosts: ["host1", "host2"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
cluster.initial_master_nodes: ["node-1"]
#
# For more information, consult the discovery and cluster formation module documentation.
#
# ---------------------------------- Gateway -----------------------------------
#
# Block initial recovery after a full cluster restart until N nodes are started:
#
#gateway.recover_after_nodes: 3
#
# For more information, consult the gateway module documentation.
#
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#
#action.destructive_requires_name: true

```



添加配置文件

```shell
echo "vm.max_map_count=262144" > /etc/sysctl.conf
```



创建用户（`root`账户启动`elasticsearch`会报错）

```shell
adduser elastic
passwd elastic
```



修改新建用户权限（`elastic`创建文件描述的权限至少需要`65536`）

```shell
# 编辑指定系统文件
vim /etc/security/limits.conf
# 文档底部追加如下内容
elastic hard nofile 65536
elastic soft nofile 65536
```



修改脚本权限

```shell
chmod +x /usr/local/elasticsearch-7.6.2/bin/elasticsearch
```



添加开机自启脚本

```shell
[Unit]
#服务描述
Description=elasticsearch
[Service]
LimitNOFILE=100000
LimitNPROC=100000
#服务命令 elasticsearch-server命令的绝对路径 elasticsearch.conf配置文件的绝对路径
ExecStart=/usr/local/elasticsearch-7.6.2/bin/elasticsearch
#执行脚本的账户
User=elastic
#执行脚本的账户分组（应该是没什么用，但是加上也没出错）
Group=elastic
[Install]
#运行级别下服务安装的相关设置，可设置为多用户，即系统运行级别为3
WantedBy=multi-user.target

```



常用命令

```shell
#启动elasticsearch服务
systemctl start elasticsearch.service
#设置开机自启动
systemctl enable elasticsearch.service
#停止开机自启动
systemctl disable elasticsearch.service
#查看服务当前状态
systemctl status elasticsearch.service
#重新启动服务
systemctl restart elasticsearch.service
#查看所有已启动的服务
systemctl list-units --type=service
```

