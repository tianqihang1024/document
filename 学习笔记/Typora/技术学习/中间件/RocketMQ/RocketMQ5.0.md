





















```shell
nohup sh bin/mqnamesrv &
```





```shell
nohup sh bin/mqbroker -n localhost:9876 --enable-proxy &
```





```shell
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
```





```shell
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```









```shell

```











创建订阅关系

```shell
sh mqadmin updateSubGroup 
```







```shell
$ docker run -d --name dashboard -e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.200.130:9876" -p 8080:8080 -t apacherocketmq/rocketmq-dashboard:latest
```