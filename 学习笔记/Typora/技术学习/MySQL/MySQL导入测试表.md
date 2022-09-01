# mysql官方有个自带的测试数据库

mysql官方有个自带的测试数据库，叫employees，超过三十万的数据，含六个表格。在MySQL官网上提供的GitHub链接可以下载
官网地址：https://dev.mysql.com/doc/employee/en/
github地址：https://github.com/datacharmer/test_db

官方安装教程：https://dev.mysql.com/doc/employee/en/employees-installation.html

1、 将下载的到包上传至服务器

 [test_db-master.zip](..\..\..\TyporaPackage\test_db-master.zip) 

有道云路径：https://note.youdao.com/s/92wiOwQr

2、解压 [test_db-master.zip](H:\迅雷下载\test_db-master.zip) 

```shell
unzip test_db-master.zip -C 指定解压的目录
```

3、`Docker`版`MySQL`需要将文件上传到容器内部

```shell
# 查看容器长名称
docker inspect -f '{{.ID}}' 容器名称
# 将文件拷贝到指定容器内部（如果相传到容器根目录使用 / 即可）
docker cp 文件全路径 容器长ID:容器内部目录
```

4、进入到解压的目录`test_db-master`

```shell
cd test_db-master/
```

5、导入数据

```shell
## 通过root进行导入、在下面的代码后键入当前数据库的密码即可导入
mysql -t <employees.sql -u root -p
```

