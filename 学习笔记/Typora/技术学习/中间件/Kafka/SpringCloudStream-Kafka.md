







bj-cdb-6r45n79k.sql.tencentcdb.com

60062









绷不住了，使用nacos2.0.3作为注册中心，包路径不能是com，因为会把你写的代码误认成自身的，重新再加载一遍配置，但是你的代码里肯定没有这个类Selector.class，很抱歉，启动失败















## 启动异常



### 异常一：`java.net.UnknownHostException: kafka1`

解决方法：服务所在服务器的`hosts`中追加下列映射。地址为`kafka`服务端所在的服务器地址，`kafka1`为`kafka`服务名。

```
# kafka 关系映射
192.168.71.128	kafka1
192.168.71.128	kafka2
192.168.71.128	kafka3
```

