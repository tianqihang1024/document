



## 开放防火墙端口

```shell
# 查看已开放端口
firewall-cmd --list-all
firewall-cmd --zone=public --list-ports
# 开放端口号
firewall-cmd --zone=public --add-port=3306/tcp --permanent
# 重新加载配置
firewall-cmd --reload

# 命令含义：
–zone # 作用域
–add-port=80/tcp # 添加端口，格式为：端口/通讯协议
–permanent # 永久生效，没有此参数重启后失效
firewall-cmd --reload # 并不中断用户连接，即不丢失状态信息

# 设置成命令模式
systemctl set-default multi-user.target
# 设置成命令模式
systemctl set-default graphical.target
```

## `firewalld`的基本使用

```shell
# 启动
systemctl start firewalld
# 关闭
systemctl stop firewalld
# 查看状态
systemctl status firewalld
# 开机禁用
systemctl disable firewalld
# 开机启用
systemctl enable firewalld

```







## 设置静态`IP`

编辑配置文件

```shell
vim /etc/sysconfig/network-scripts/ifcfg-ens33
```

文本内容

```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"   # 设置静态IP模式
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="bbd2877d-5957-49e2-b32a-40ea381ee02a"
DEVICE="ens33"
ONBOOT="yes"		# 允许访问互联网

IPADDR=192.168.71.128	# 自定义网络IP
GATEWAY=192.168.71.2	# 网关地址，虚拟机-编辑-虚拟网络编辑器查看
NETMASK=255.255.255.0	# 网络子掩码
DNS1=192.168.1.1		# 虚拟机所在电脑的DNS码
DNS2=8.8.8.8			# 无特殊含义，但是也不敢删
```

