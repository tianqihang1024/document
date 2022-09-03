

# [Linux：CentOS7安装MySQL8（详） - Jaywee - 博客园 (cnblogs.com)](https://www.cnblogs.com/secretmrj/p/15600144.html#page_end_html)



## 1. 下载 MySQL 的 Yum 源

```shell
wget https://dev.mysql.com/get/mysql80-community-release-el7-2.noarch.rpm
```





## 2. 用 yum命令安装下载好的rpm包。



```shell
yum -y install mysql80-community-release-el7-2.noarch.rpm
```



13.配置mysql开机启动

vi /etc/rc.local

在文件中添加 service mysqld start即可



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
2022-10-30T02:15:26.290223Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: W6vSN.6j:OT8
```

`q*tbrDpXl64L` 是`root`的默认密码

```shell
# 登陆 MySQL
mysql -u root -p
```

修改密码

```mysql
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'W6vSN.6j:OT8';
```





## 6. 设置 MySQL 外网访问



```mysql
mysql> use mysql;
mysql> update user set host="%" where user='root';
mysql> flush privileges;
mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'W6vSN.6j:OT8';
mysql> grant all privileges on *.* to root@'%' with grant option;
```



## 7.放开防火墙

```shell
# 开放端口号
firewall-cmd --zone=public --add-port=3306/tcp --permanent
# 重新加载配置
systemctl restart firewalld.service
```



## 8.开机自启

**yum安装，那么就会自动添加mysql成系统服务，那么可以直接systemctl enable mysqld.service设置开机自启动：**

```shell
# 启动mysql服务
systemctl start mysqld.service
# 停止mysql服务
systemctl stop mysqld.service
# 重启mysql服务
systemctl restart mysqld.service
# 查看mysql服务当前状态
systemctl status mysqld.service
# 开机自启
systemctl enable mysqld.service
# 停止mysql服务开机自启动
systemctl disable mysqld.service
```





网址：[CentOS 安装 MySQL8 - 掘金 (juejin.cn)](https://juejin.cn/post/7056265988673568781)


