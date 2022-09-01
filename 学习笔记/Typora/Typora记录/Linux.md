```shell
# 修改文件
vim /etc/hostname

# 切换目录
cd /use/local

# 查看本机IP地址
ifconfig

# 赋予文件读写和可执行的能力
# chmod：在linux系统中它用于改变文件或目录的访问权限。用户用它控制文件或目录的访问权限。
# -R : 对目前目录下的所有档案与子目录进行相同的权限变更(即以递回的方式逐个变更) 。
# 777:分别对应文件实际拥有者，文件实际拥有者所在的组，其它用户的权限，数字权限是基于八进制数字系统而创建的，读权限（read，r）的值是4，写权限（write，w）的值是2，执行权限（execute，x）的值是1，没有授权的值是0。
# *：通配符，指当前目录下的所有文件及目录。
sudo chmod -R 777 /use/local/redis

# 查看程序是否启动
ps -ef | grep redis

# 杀死进程ID
kill -9 进程ID

# 刷新配置文件并使其生效
source 文件路径

# 防火墙相关：启动、查看、停止、禁用（firewalld = 法尔我）
systemctl start firewalld
systemctl status firewalld
systemctl stop firewalld 
systemctl disable firewalld


# 创建文件夹
mkdir 文件路径/文件名

# 创建文件


# 删除文件


# 修改文件


# 拷贝文件，-r表示递归形式的拷贝，用于要拷贝的文件存在子文件。
cp 文件名 要拷贝到的地方
cp -r 文件名 要拷贝到的地方

# 打印文件内容（会将文件内容全部打印出来，建议只对内容较少的文件使用）
cat 文件名

# 动态打印文件内容，-1000f 代表展示最新的1000条日志，并且动态刷新。
tail -f 文件名
tail -1000f 文件名


# 查询rabbitmq 默认被指文件存放地
find / -name rabbitmq-defaults

# 查看用户
id nfsuser(用户名)

# 跨域同步数据，前面为本地文件，后面的是别的机子
scp /usr/java/redis root@mq2:/usr/java/redis/

# 下载 vim 编辑器
yum -y install vim*
```































