





## 1. 下载 MySQL 的 Yum 源

```shell
wget https://dev.mysql.com/get/mysql80-community-release-el7-2.noarch.rpm
```





## 2. 用 yum命令安装下载好的rpm包。



```shell
yum -y install mysql80-community-release-el7-2.noarch.rpm
```







## 3. 安装 MySQL Server



```shell
# 启动 MySQL
yum -y install mysql-community-server
# 启动 MySQL --nogpgcheck 跳过 GPG 验证 
yum -y install mysql-community-server  --nogpgcheck
```





## 4. 登陆 MySQL

```shell
# 启动 MySQL 命令
systemctl start  mysqld.service
```

```shell
# 查看 MySQL 运行状态
systemctl status mysqld.service
```

![image-20221030101820220](C:/Users/22489/OneDrive/%E7%94%B0%E5%A5%87%E6%9D%AD/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/TyporaImg/image-20221030101820220.png)

其中Active后面代表状态启功服务后为active (running)，停止后为inactive (dead)

## 5. 登陆 MySQL



```shell
# 查看 root 初始密码 
[root@bogon ~]# grep "password" /var/log/mysqld.log
2022-10-30T02:15:26.290223Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: q*tbrDpXl64L
```

`q*tbrDpXl64L` 是`root`的默认密码

```shell
# 登陆 MySQL
mysql -u root -p
```





## 6. 设置 MySQL 外网访问



```mysql
mysql> use mysql;
mysql> update user set host="%" where user='root';
mysql> flush privileges;
mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'q*tbrDpXl64L';
```



## 7.放开防火墙

```shell
# 开放端口号
firewall-cmd --zone=public --add-port=3306/tcp --permanent
# 重新加载配置
systemctl restart firewalld.service
```









网址：[CentOS 安装 MySQL8 - 掘金 (juejin.cn)](https://juejin.cn/post/7056265988673568781)













