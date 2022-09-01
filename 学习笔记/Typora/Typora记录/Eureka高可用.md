# eureka高可用

## 高可用介绍

当其中一台的服务发生故障时不影响整体服务状况，不能因为一台服务器的问题导致服务停止，高可用的方法有三种：主从方式、双机双工方式、集群工作方式。而Zookeeper采用的是主从方式、Eureka则采用的是集群方式，当多台服务器相互注册就形成了高可用，这样当其中的一台停止提供服务时，剩余的则会继续提供服务。

## 高可用Eureka服务

和Eureka基本使用类似，需要引入相关的依赖添加到POM中，这里不再赘述。但是与基本使用不一致的是在application.xml中进行修改：

```properties
# 设置端口号
server.port=8100
# 设置host名称
eureka.instance.hostname=localhost
# 设置应用的名称 推荐设置统一名称
spring.application.name=lattice-eureka
# 官方推荐让注册中心相互注册
eureka.client.register-with-eureka=true
eureka.client.fetch-registry=true
# 设置显示真实ip
eureka.instance.prefer-ip-address=true
# 设置注册到其他的注册中心的地址
eureka.client.service-url.defaultZone=http://127.0.0.1:8200/eureka/,http://127.0.0.1:8300/eureka/
# 关闭自我保护
eureka.server.enable-self-preservation=false
# 清理间隔10s
eureka.server.eviction-interval-timer-in-ms=10000
1234567891011121314151617
```

在其他的Eureka服务端只需要修改eureka.client.service-url.defaultZone和server.port就可以了。