





## ContOS8.2安装Docker

1、安装一些必要的工具

```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

2、添加软件源信息

```shell
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

3、更新 Docker-CE

```shell
sudo yum makecache
```

4、安装 Docker-CE

```shell
sudo yum -y install docker-ce
```

5、开启Docker服务

```shell
# 启动docker
sudo service docker start
# 开机自启
sudo systemctl enable docker
```

6、配置Docker阿里云镜像

```shell
vim /etc/docker/daemon.json
# 添加进/etc/docker/daemon.json
{
"registry-mirrors": ["https://4wgtxa6q.mirror.aliyuncs.com"]
}
```

7、相关命令

```shell
# 查看docker版本
docker version
# 查看镜像
docker images
# 查看容器内存占比
docker stats
# 查看docker运行中的容器
docker ps
# 这个选项用于列出所有容器，包括停止运行的。
docker ps -a
# 这个选项列出容器的数字 ID，而不是容器的所有信息。
docker ps -p
# 停止容器
docker stop 容器ID
# 删除容器
docker rmi 容器ID
# 查看容器分配的IP
docker inspect --format='{{.NetworkSettings.IPAddress}}' 容器ID
# 查看虚拟网卡IP地址列表（自己根据端口号联想是那个服务）
iptables -t nat -nL --line-number
# 查看最新的100条日志，并持续更新
docker logs -f --tail=100 容器ID
# 查看最近30分钟的日志:
docker logs --since 30m 容器ID
# 查看某时间之后的日志：
docker logs -t --since="2018-02-08T13:23:37" 容器ID
# 查看某时间段日志：
docker logs -t --since="2018-02-08T13:23:37" --until "2018-02-09T12:23:37" 容器ID
# 创建子网段
docker network create --driver bridge --subnet 172.18.0.0/24 --gateway 172.18.0.1 docker_net
# 创建容器时指定容器子网段IP，在创建容器命令中指定 --network docker_net --ip 172.18.0.5
docker run -d --name nacos -p 8848:8848 --network docker_net --ip 172.18.0.5 nacos/nacos-server:v2.0.3
```



## 相关链接

[配置Docker镜像加速器-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/606808?spm=a2c6h.14164896.0.0.79281b2cAIzB3I)

[Centos8.2轻松安装docker_俗人世界-CSDN博客](https://blog.csdn.net/weixin_41887155/article/details/107232529)

































