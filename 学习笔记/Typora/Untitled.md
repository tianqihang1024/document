



```
mkdir -p /data/mysql_data

chown -R mysql:mysql /data/mysql_data

chmod -R 750 /data/mysql_data

mysqld --defaults-file=/usr/local/etc/my.cnf --basedir=/usr/local/mysql --datadir=/data/mysql_data/mysql --user=mysql --initialize-insecure

ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'W6vSN.6j:OT8';
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'W6vSN.6j:OT8';

```









mysql=2

es =2

redis=1





局部变量也需要考虑作用域，否则他的变化很难估测

局部的变量声明和使用，应该在一起，不建议声明与操作中存在其他代码，可能会对变量做出其他操作



第三方接口不可信，肯定会出现异常，必须包裹起来，避免影响程序正确性



浅拷贝和深拷贝

- 浅拷贝对于基础数据类型拷贝的是值，引用类型拷贝的是对象的引用
- 深拷贝对于基础数据类型拷贝的是值，引用类型拷贝的是对象本身，被拷贝对象和拷贝对象是两个对象

总结：浅拷贝和深拷贝对于基础数据类型的操作是一样的，区别是引用类型上有点出入