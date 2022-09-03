



1、下载框架对应的版本

```shell
$ docker pull seataio/seata-server:1.3.0
```



2、创建容器

-   `--restart=always`：生命周期绑定到`docker`
-   `SEATA_IP`：公网`IP`（非公网会导致服务无法连接）
-   `JVM_XMS、JVM_XMX`：指定初始化堆大小、最大内存占比

```shell
$ docker run -d --name seata --restart=always -p 8091:8091 --network docker_net --ip 172.18.0.6 -e JVM_XMS=256m -e JVM_XMX=256m -e SEATA_IP=81.70.96.232 -e SEATA_PORT=8091 seataio/seata-server:1.3.0
docker run -d --name seata --restart=always -p 8091:8091 -e JVM_XMS=516m -e JVM_XMX=516m -e SEATA_IP=81.70.96.232 -e SEATA_PORT=8091 seataio/seata-server:1.3.0
```



3、创建`seata`的数据库

```sql
/*
 Navicat Premium Data Transfer

 Source Server         : 腾讯云-mysql
 Source Server Type    : MySQL
 Source Server Version : 50718
 Source Host           : bj-cdb-6r45n79k.sql.tencentcdb.com:60062
 Source Schema         : seata

 Target Server Type    : MySQL
 Target Server Version : 50718
 File Encoding         : 65001

 Date: 22/10/2022 21:34:31
*/

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for branch_table
-- ----------------------------
DROP TABLE IF EXISTS `branch_table`;
CREATE TABLE `branch_table`  (
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `transaction_id` bigint(20) NULL DEFAULT NULL,
  `resource_group_id` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `resource_id` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `branch_type` varchar(8) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `status` tinyint(4) NULL DEFAULT NULL,
  `client_id` varchar(64) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `application_data` varchar(2000) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `gmt_create` datetime(6) NULL DEFAULT NULL,
  `gmt_modified` datetime(6) NULL DEFAULT NULL,
  PRIMARY KEY (`branch_id`) USING BTREE,
  INDEX `idx_xid`(`xid`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Table structure for global_table
-- ----------------------------
DROP TABLE IF EXISTS `global_table`;
CREATE TABLE `global_table`  (
  `xid` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `transaction_id` bigint(20) NULL DEFAULT NULL,
  `status` tinyint(4) NOT NULL,
  `application_id` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `transaction_service_group` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `transaction_name` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `timeout` int(11) NULL DEFAULT NULL,
  `begin_time` bigint(20) NULL DEFAULT NULL,
  `application_data` varchar(2000) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `gmt_create` datetime(0) NULL DEFAULT NULL,
  `gmt_modified` datetime(0) NULL DEFAULT NULL,
  PRIMARY KEY (`xid`) USING BTREE,
  INDEX `idx_gmt_modified_status`(`gmt_modified`, `status`) USING BTREE,
  INDEX `idx_transaction_id`(`transaction_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Table structure for lock_table
-- ----------------------------
DROP TABLE IF EXISTS `lock_table`;
CREATE TABLE `lock_table`  (
  `row_key` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `xid` varchar(96) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `transaction_id` bigint(20) NULL DEFAULT NULL,
  `branch_id` bigint(20) NOT NULL,
  `resource_id` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `table_name` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `pk` varchar(36) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `gmt_create` datetime(0) NULL DEFAULT NULL,
  `gmt_modified` datetime(0) NULL DEFAULT NULL,
  PRIMARY KEY (`row_key`) USING BTREE,
  INDEX `idx_branch_id`(`branch_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

SET FOREIGN_KEY_CHECKS = 1;

```



4、编辑配置文件

```shell
# 查看容器docker内部ip
$ docker inspect --format='{{.NetworkSettings.IPAddress}}' 容器ID

# 进入容器
$ docker exec -it 容器id/容器名称 /bin/bash
$ docker exec -it 容器id/容器名称 /bin/sh

```

`file.conf`修改为`db`，修改`url`、`user`、`password`。`mysql 8`需要修改`driverClassName`.

```shell
service{
   vgroup_mapping.my_test_tx_group = "default"
   ## nacos的IP:port
   ## 如果 nacos 也是以容器的形式存在，并且和 seata 在一台服务器上，这里的 IP 为 docker 给 nacos 分配的内部 IP
   ## seata 和 nacos 不在同一台机器，IP 为 nacos 的公网 IP
   default.grouplist = "81.70.96.232:8091"
   disableGlobalTransaction = false
}

## transaction log store, only used in seata-server
store {
  ## store mode: file、db、redis
  mode = "db"

  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    ## driverClassName = "com.mysql.jdbc.Driver" # mysql5.7
    driverClassName = "com.mysql.cj.jdbc.Driver" # mysql8
    # seata 与 mysql 在同一台服务器的docker中，链接地址需要使用docker分配的ip
    url = "jdbc:mysql://bj-cdb-6r45n79k.sql.tencentcdb.com:60062/extend_seata?characterEncoding=utf8&useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true"
    user = "root"
    password = "W6vSN.6j:OT8"
    minConn = 5
    maxConn = 30
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }

  ## redis store property
  redis {
    host = "127.0.0.1"
    port = "6379"
    password = ""
    database = "0"
    minConn = 1
    maxConn = 10
    queryLimit = 100
  }

}
```

`registry.conf`注册中心使用`nacos`，配置文件使用本地的`file.conf`。

```shell
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    application = "seata-server" # 注册到nacos的服务名
    ## 172.17.0.3 为docker的虚拟网卡IP，通过命令查找：docker inspect --format='{{.NetworkSettings.IPAddress}}' 容器ID
    serverAddr = "172.18.0.5:8848" 
    group = "SEATA_GROUP" # nacos分组
    namespace = ""
    cluster = "default"
    username = ""
    password = ""
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"
  
  file {
    name = "file.conf"
  }
}

```



5、重启`seata`容器

```shell
$ docker restart 容器id/容器名
```



6、查看日志

```shell
$ docker logs -f --tail=200 容器id/容器名
```











