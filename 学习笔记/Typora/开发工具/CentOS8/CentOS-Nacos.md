

# CentOS8-Nacos



1、下载镜像

```shell
docker pull nacos/nacos-server:v2.0.3
```

2、创建实例

```shell
# 三个 -p 分别是，指定容器在docker内部的端口号，将docker内部的端口号映射到虚拟机的9848、9849，解决程序连接nacos出错。
docker run -d --name nacos -p 8848:8848 -p 9848:9848 -p 9849:9849 --restart=always --network docker_net --ip 172.18.0.5 -e JVM_XMS=256m -e JVM_XMX=256m --env MODE=standalone  nacos/nacos-server:v2.0.3

# 创建容器时，指定远程数据库（待实验）
docker run -d --name nacos -p 8848:8848 -p 9848:9848 -p 9849:9849 --env MODE=standalone --env SPRING_DATASOURCE_PLATFORM=mysql --env MYSQL_SERVICE_HOST=bj-cdb-6r45n79k.sql.tencentcdb.com --env MYSQL_SERVICE_PORT=60062 --env MYSQL_SERVICE_DB_NAME=nacos-config --env MYSQL_SERVICE_USER=root --env MYSQL_SERVICE_PASSWORD=TQH&19970227 --env NACOS_DEBUG=n nacos/nacos-server

```

3、与`Docker`绑定生命周期

```shell
sudo docker container update --restart=always 容器ID
```

4、访问页面，账户：nacos 密码：nacos

http://81.70.96.232:8848/nacos/index.html





```

```

