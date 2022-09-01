# `Docker`部署`WEB`项目



创建目录存放`WEB`项目和`Docker`文件，上传`jar`

```shell
mkdir /usr/local/web
```

创建`Docker`执行文件`Dockerfile`，内容如下

```shell
# 运行环境
FROM openjdk:8
# 维护者
MAINTAINER qihang
# 添加jar包
ADD WEB服务名称 WEB服务名称
# 运行命令
ENTRYPOINT ["java","-jar","WEB服务名称"]
```

编译`jar`为镜像

```shell
# gateway 是编译后的镜像名称
docker build -t gateway .
docker build -t seckill .
```

创建容器，`-p`宿主机的`9000`端口与`docker`的`9000`端口进行映射

```shell
docker run -d --restart always --name gateway --network docker_net --ip 172.18.0.20 -e JVM_XMS=128m -e JVM_XMX=128m -e TZ="Asia/Shanghai" -p 9000:9000 gateway
docker run -d --restart always --name seckill -e JVM_XMS=1024m -e JVM_XMX=1024m -e TZ="Asia/Shanghai" -p 9030:9030 seckill
```









