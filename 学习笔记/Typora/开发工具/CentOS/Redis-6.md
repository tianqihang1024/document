











## 升级`gcc`版本

```shell
yum -y install centos-release-scl 
yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils 
scl enable devtoolset-9 bash
echo "source /opt/rh/devtoolset-9/enable" >>/etc/profile
# 查看 gcc 版本
gcc -v
```



## 下载`Redis`安装包，并上传到你想安装的目录中

`Redis`历史版本：[Index of /releases/ (redis.io)](http://download.redis.io/releases/)

## 

## 解压安装包

```shell
tar zxvf redis-6.2.9.tar.gz
```



## 编译`Redis`

切换解压后的文件夹内

```shell
cd redis-6.2.9
```

编译reids

```shell
make
```

安装redis

```shell
make install PREFIX=/usr/local/redis
```

拷贝redis.conf

```shell
mkdir /usr/local/redis/etc
cp redis.conf /usr/local/redis/etc
```

配置文件修改

```
vim /usr/local/redis/etc/redis.conf
```

修改项

````
# 注释 bing
bing 127.0.0.1 ===> bing 127.0.0.1
# 修改密码
requirepass 123 ===> requirepass 新密码
````





## `Redis`环境配置

设置系统变量

```
vim /etc/profile
```

文本最后追加

```shell
REDIS_HOME=/usr/local/redis/
PATH=$PATH:$REDIS_HOME/bin
```





## 设置开启自启

编辑`redis.service`

```shell
vim /etc/systemd/system/redis.service
```

插入下方内容

```sh
[Unit]
#服务描述
Description=Redis Server Manager
#服务类别
After=syslog.target network.target
[Service]
#后台运行的形式（自启+后台会导致服务异常，这里将其注释）
#Type=forking
#服务命令 redis-server命令的绝对路径 redis.conf配置文件的绝对路径
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf
#给服务分配独立的临时空间
PrivateTmp=true
[Install]
#运行级别下服务安装的相关设置，可设置为多用户，即系统运行级别为3
WantedBy=multi-user.target
```

常用命令

```shell
#启动redis服务
systemctl start redis.service
#设置开机自启动
systemctl enable redis.service
#停止开机自启动
systemctl disable redis.service
#查看服务当前状态
systemctl status redis.service
#重新启动服务
systemctl restart redis.service
#查看所有已启动的服务
systemctl list-units --type=service
```





## 配置文件自己看下面的相关链接吧



相关地址：

[CentOS升级gcc到高版本（全部版本详细过程）_乞力马扎罗の黎明的博客-CSDN博客_centos 升级gcc](https://blog.csdn.net/qq_39715000/article/details/120703444)

[Index of /releases/ (redis.io)](http://download.redis.io/releases/)

[centos7 安装 redis6 开机自启动 - swk3com - 博客园 (cnblogs.com)](https://www.cnblogs.com/swk3/p/15142530.html)

[Redis6.0.5版本配置文件说明（非常详细，全文1.5w字）_风雨兼程，披星戴月。的博客-CSDN博客_always-show-logo yes](https://blog.csdn.net/qq_42534026/article/details/106730314)