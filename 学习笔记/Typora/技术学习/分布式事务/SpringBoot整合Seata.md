# `SpringBoot`整合`Seata`

**请按照：`开发工具.CentOS.CentOS-Seata`，安装好`Seata`后，再开始本教程**

**请按照：`开发工具.CentOS.CentOS-Seata`，安装好`Seata`后，再开始本教程**

**请按照：`开发工具.CentOS.CentOS-Seata`，安装好`Seata`后，再开始本教程**





## 架构版本

1.   `jdk：1.8`
2.   `spring-boot：2.3.12.RELEASE`
3.   `spring-cloud：Hoxton.SR12`
4.   `spring-cloud-alibaba：2.2.7.RELEASE`
5.   `nacos：2.0.3`
6.   `seata：1.3.0`



## 引入`seata`依赖

```pom
<dependencies>
        <!-- 注册中心 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>

        <!-- seata 分布式事务依赖 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>seata-all</artifactId>
                    <groupId>io.seata</groupId>
                </exclusion>
                <exclusion>
                    <groupId>io.seata</groupId>
                    <artifactId>seata-spring-boot-starter</artifactId>
                </exclusion>
                <exclusion>
                    <artifactId>spring-boot-starter</artifactId>
                    <groupId>org.springframework.boot</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-all</artifactId>
            <version>1.3.0</version>
            <exclusions>
                <exclusion>
                    <artifactId>fastjson</artifactId>
                    <groupId>com.alibaba</groupId>
                </exclusion>
            </exclusions>
        </dependency>
</dependencies>
```



创建`seata`配置类`SeataAutoConfig`

```java
package extend.config;

import com.alibaba.druid.pool.DruidDataSource;
import io.seata.rm.datasource.DataSourceProxy;
import io.seata.spring.annotation.GlobalTransactionScanner;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.jdbc.DataSourceProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.DependsOn;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

@Slf4j
@Configuration
public class SeataAutoConfig {

    @Autowired
    private DataSourceProperties dataSourceProperties;

    /**
     * 创建代理数据源
     *
     * @return druidDataSource
     */
    @Bean(name = "druidDataSource")
    public DataSource druidDataSource() {
        DruidDataSource druidDataSource = new DruidDataSource();
        log.info("dataSourceProperties.getUrl():{}", dataSourceProperties.getUrl());
        druidDataSource.setUrl(dataSourceProperties.getUrl());
        druidDataSource.setUsername(dataSourceProperties.getUsername());
        druidDataSource.setPassword(dataSourceProperties.getPassword());
        druidDataSource.setDriverClassName(dataSourceProperties.getDriverClassName());
        druidDataSource.setInitialSize(0);
        druidDataSource.setMaxActive(180);
        druidDataSource.setMaxWait(60000);
        druidDataSource.setMinIdle(0);
        druidDataSource.setValidationQuery("Select 1 from DUAL");
        druidDataSource.setTestOnBorrow(false);
        druidDataSource.setTestOnReturn(false);
        druidDataSource.setTestWhileIdle(true);
        druidDataSource.setTimeBetweenEvictionRunsMillis(60000);
        druidDataSource.setMinEvictableIdleTimeMillis(25200000);
        druidDataSource.setRemoveAbandoned(true);
        druidDataSource.setRemoveAbandonedTimeout(1800);
        druidDataSource.setLogAbandoned(true);
        log.info("装载dataSource........");
        return druidDataSource;
    }

    /**
     * init datasource proxy
     *
     * @Param: druidDataSource datasource bean instance
     * @Return: DataSourceProxy datasource proxy
     */
    @Bean(name = "dataSource")
    @Primary // 在同样的DataSource中，首先使用被标注的DataSource
    @DependsOn(value = {"druidDataSource"}) // druidDataSource 加载到容器中后才会被加载，避免因为加载顺序导致NP
    public DataSourceProxy dataSourceProxy(@Qualifier(value = "druidDataSource") DruidDataSource druidDataSource) {
        log.info("代理dataSource........");
        return new DataSourceProxy(druidDataSource);
    }

    /**
     * init global transaction scanner
     *
     * @Return: GlobalTransactionScanner
     */
    @Bean
    @DependsOn(value = {"druidDataSource"}) // druidDataSource 加载到容器中后才会被加载，避免因为加载顺序导致NP
    public GlobalTransactionScanner globalTransactionScanner() {
        log.info("配置seata........");
        return new GlobalTransactionScanner("test-service", "my_test_tx_group");
    }
}
```





## 导入`seata`配置文件

放入`web`的`resources`目录下，直接创建

`file.conf`

```conf
transport {
  # tcp udt unix-domain-socket
  type = "TCP"
  #NIO NATIVE
  server = "NIO"
  #enable heartbeat
  heartbeat = true
  #thread factory for netty
  thread-factory {
    boss-thread-prefix = "NettyBoss"
    worker-thread-prefix = "NettyServerNIOWorker"
    server-executor-thread-prefix = "NettyServerBizHandler"
    share-boss-worker = false
    client-selector-thread-prefix = "NettyClientSelector"
    client-selector-thread-size = 1
    client-worker-thread-prefix = "NettyClientWorkerThread"
    # netty boss thread size,will not be used for UDT
    boss-thread-size = 1
    #auto default pin or 8
    worker-thread-size = 8
  }
  shutdown {
    # when destroy server, wait seconds
    wait = 3
  }
  serialization = "seata"
  compressor = "none"
}
service {
  #vgroup->rgroup
  vgroupMapping.my_test_tx_group = "default"
  #only support single node
  default.grouplist = "81.70.96.232:8091"
  #degrade current not support
  enableDegrade = false
  #disable
  disable = false
  #unit ms,s,m,h,d represents milliseconds, seconds, minutes, hours, days, default permanent
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
}

client {
  async.commit.buffer.limit = 10000
  lock {
    retry.internal = 10
    retry.times = 30
  }
  report.retry.count = 5
}

## transaction log store
store {
  ## store mode: file、db
  mode = "db"

  ## database store
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp) etc.
    datasource = "dbcp"
    ## mysql/oracle/h2/oceanbase etc.
    db-type = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://bj-cdb-6r45n79k.sql.tencentcdb.com:60062/seata?useUnicode=true&rewriteBatchedStatements=true"
    user = "root"
    password = "Tqh962454294"
    min-conn = 1
    max-conn = 3
    global.table = "global_table"
    branch.table = "branch_table"
    lock-table = "lock_table"
    query-limit = 100
  }
}
lock {
  ## the lock store mode: local、remote
  mode = "remote"

  local {
    ## store locks in user's database
  }

  remote {
    ## store locks in the seata's server
  }
}
recovery {
  committing-retry-delay = 30
  asyn-committing-retry-delay = 30
  rollbacking-retry-delay = 30
  timeout-retry-delay = 30
}

transaction {
  undo.data.validation = true
  undo.log.serialization = "jackson"
}

## metrics settings
metrics {
  enabled = false
  registry-type = "compact"
  # multi exporters use comma divided
  exporter-list = "prometheus"
  exporter-prometheus-port = 9898
}
```

`registry.conf`

```conf
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    application = "seata-server"
    serverAddr = "81.70.96.232:8848"
    group = "SEATA_GROUP"
    namespace = "public"
    cluster = "default"
    username = "nacos"
    password = "nacos"
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





## 完了，就这么简单

