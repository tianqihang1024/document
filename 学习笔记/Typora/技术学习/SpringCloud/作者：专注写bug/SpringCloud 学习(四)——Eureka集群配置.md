# Spring Cloud 学习(四)——Eureka集群配置

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[专注写bug](https://me.csdn.net/qq_38322527) 2019-09-01 22:27:56 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 47 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏



分类专栏： [springcloud](https://blog.csdn.net/qq_38322527/category_8898018.html)

版权

## 一、配置要点

### 单独的注册中心配置文件

```yml
###服务端口号
server:
  port: 10000

###eureka 基本信息配置
eureka:
  instance:
    ###注册到eurekaip地址
    hostname: 127.0.0.1
  client:
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
###因为自己是为注册中心，不需要自己注册自己
    register-with-eureka: false
###因为自己是为注册中心，不需要检索服务
    fetch-registry: false
```

一台注册中心不安全，如果注册中心宕机了，此时则需要进行注册中心的集群部署操作，多设置几台注册中心，一台宕了，另外一台启动顶替(上一台宕机到下一台启动默认时间30秒)

## 二、修改配置文件，实现集群操作

### 1、如果要保证多台注册中心，此时必须指定注册中心的别名，**且同一个注册中心的别名必须一致**

```yml
spring: 
 application: 
  name: eureka-server
```

### 2、注册中心1的配置

```yml
###服务端口号
server:
  port: 9000

###eureka 基本信息配置
spring: 
 application: 
  name: eureka-server

eureka:
  instance:
    ###注册到eurekaip地址
    hostname: 127.0.0.1
  client:
    serviceUrl:
        #注册至别的eureka注册中心上，有多个采取","区别
      defaultZone: http://127.0.0.1:10000/eureka/
###此时有两个注册中心，采取集群配置必须打到我中有你，你中有我的境界，所以必须彼此进行注册操作
    register-with-eureka: true
###此时有两个注册中心，采取集群配置必须打到我中有你，你中有我的境界，所以必须彼此开放服务
    fetch-registry: true
```

### 3、注册中心2的配置

```yml
###服务端口号
server:
  port: 10000

###eureka 基本信息配置（别名一致）
spring: 
 application: 
  name: eureka-server

eureka:
  instance:
    ###注册到eurekaip地址
    hostname: 127.0.0.1
  client:
    serviceUrl:
        #注册至别的eureka注册中心上，有多个采取","区别
      defaultZone: http://127.0.0.1:9000/eureka/
###此时有两个注册中心，采取集群配置必须打到我中有你，你中有我的境界，所以必须彼此进行注册操作
    register-with-eureka: true
###此时有两个注册中心，采取集群配置必须打到我中有你，你中有我的境界，所以必须彼此开放服务
    fetch-registry: true
```

### 4、多个注册中心呢？如何操作？

只需要在  defaultZone  后添加","，后进行追加即可，如：

```yml
eureka:
  instance:
    ###注册到eurekaip地址
    hostname: 127.0.0.1
  client:
    serviceUrl:
      defaultZone: http://127.0.0.1:9000/eureka/,http://127.0.0.1:11000/eureka/,...
```

### 5、修改各项服务的注册地址

member中的地址修改

```yml
###服务注册到eureka地址
eureka:
  client:
    service-url:
            #各个微服务，需要将自己的服务上报到所有的eureka集群上
           defaultZone: http://localhost:10000/eureka,http://localhost:9000/eureka
```

同理，orders服务中也应修改如上所示

### 三、测试

### 1、当未指定别名且注册上报为true时

 

![img](https://img-blog.csdnimg.cn/20190901221615145.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20190901221654133.png)

 

### 2、当未指定 别名且注册上报为false时，出现什么现象

```yml
spring: 
 application: 
  name: eureka-server

eureka:
  instance:
    ###注册到eurekaip地址
    hostname: 127.0.0.1
  client:
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:10000/eureka/
    register-with-eureka: false #集群应该为true，此处为异常测试
    fetch-registry: false#集群应该为true，此处为异常测试
```

无上述配置时，分别启动项目，项目不会进行互相注册操作

![img](https://img-blog.csdnimg.cn/20190901221010842.png)

![img](https://img-blog.csdnimg.cn/20190901221037421.png)

关闭两个注册中心，添加注册别名信息(必须保证别名一致)后，重新启动

### 3、正常启动结果

![img](https://img-blog.csdnimg.cn/20190901222028379.png)

![img](https://img-blog.csdnimg.cn/2019090122204495.png)

### 4、修改服务项的配置信息

member / orders

```yml
eureka:
  client:
    service-url:
           defaultZone: http://localhost:10000/eureka,http://localhost:9000/eureka
```

启动测试

![img](https://img-blog.csdnimg.cn/20190901222452264.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20190901222520722.png)

接下来我们关闭 10000的注册中心！

![img](https://img-blog.csdnimg.cn/20190901222740438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

作为备用注册中心 9000 ，只有在主注册中心10000宕机30秒后，才能起效。

 

本文代码：

https://download.csdn.net/download/qq_38322527/11663245