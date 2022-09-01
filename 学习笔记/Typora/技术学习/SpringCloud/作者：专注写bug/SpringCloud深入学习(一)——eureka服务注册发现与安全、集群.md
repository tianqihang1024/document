# SpringCloud深入学习(一)——eureka服务注册发现与安全、集群

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[专注写bug](https://me.csdn.net/qq_38322527) 2020-01-03 18:09:31 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 218 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 5



分类专栏： [springcloud](https://blog.csdn.net/qq_38322527/category_8898018.html)

版权

## 一、前言

### 1.1、普通项目集群配置的问题

我们在实际的开发中，通常会面对高请求、高负载的情况，此时此刻我们首先想到的就是对我们的项目进行分布式部署操作，再加上一个nginx做反向代理，使用ip_hash或者weight等做负载均衡操作，但是大家可能会想到一些问题。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200102161822713.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

> 问题描述：
> 1、当我需求变更，是不是每个机器上部署的整个代码都需要做变更和重新部署操作？
> 2、我的所有业务逻辑都是在同一个项目中进行，nginx只是负责进行了请求分发操作而已，我要是其中一个机器宕机了，我nginx不知道哪个项目出问题了怎么分析问题呢？

### 1.2、springboot技术

在nginx中，我们集群部署的是一整个项目，项目和项目之间一般不会进行通信操作，毕竟集群部署的项目都基本上是一样的。

在一个项目中，上线后我们会发现，有些服务请求量很大，而有些服务却请求量很少或者基本上没有太大的请求和使用。

我们为了减少计算机上资源的浪费，我们是否可以进行针对项目而言进行项目的拆分，将项目拆分为多个微笑的子项目(服务)，拆到不能拆除为止。

> 将每个细小的服务进行部署，如果出现某个小服务请求量大时，才对那个服务进行分布式部署，是不是更能够节省资源？

### 1.3、服务之间的调用

springboot中，对服务与服务之间的调用，有一个很好用的技术叫：RestTemplate。
我们每次只需要在服务创建时，在其容器中申明就可以使用。

```java
	@Bean
	@LoadBalanced
	RestTemplate restTemplate() {
		return new RestTemplate();
	}
12345
```

所以最初的操作中，我们可能会这么写

```java
String memberUrl = "http://xxx:8080/getMember";
String result = restTemplate.getForObject(memberUrl, String.class);
12
```

但这么写会有一个问题，哪天我把端口号变更了，是不是也需要修改硬编码，每次进行修改后重新clean package再上线再部署，很麻烦呀。

大家可能会想到，我将这个地址写在配置文件中，到时候修改那个配置文件就可以了啊。就像下面这样：

1、在yml中增加代码

```yml
member:
  uer: http://xxx:8080/getMember
12
```

2、在java中调用

```java
@Value("${member.uer}")
private String memberUrl;
----
String result = restTemplate.getForObject(memberUrl, String.class);
1234
```

但是此时此刻，又会出现一个问题：

> 当我的目标自服务进行了集群化配置，我的配置文件中如何写多个服务的member.uer，又如何保证自动的随着我部署的个数去动态添加配置、动态的去请求处理？------注册中心的使用

### 1.4、注册中心的使用

说到服务注册中心，我们其实主要关注的还是CAP理论。

> 我们使用表格形式，大致说明下CAP定理中的各项组成的意思。

#### 1.1、什么是CAP

| 名称            | 英语名              | 含义                                                         |
| --------------- | ------------------- | ------------------------------------------------------------ |
| 高可用性（A）   | Availability        | 当出现高负载、网络波动情况下，保证服务能及时相应请求。       |
| 一致性（C）     | Consistency         | 在分布式系统中，所有节点在同一时间内，必须保证数据完全一致，节点越多，需要同步数据耗时越多 |
| 分区容错性（P） | Partition tolerance | 相同服务集群部署后，其中一个宕机，整个系统不会受到影响。     |

#### 1.2、CAP中为什么不能同时满足

| 组合 | 含义                  | 描述                                                         |
| ---- | --------------------- | ------------------------------------------------------------ |
| AP   | 高可用、 分区容错     | 为了满足分区容错，则需要某个服务尽量多的集群部署（P）； 为了能够及时相应就必须减少数据处理中的耗时时间（A）； 数据同步需要时间，机器越多（P），时间越长，时间越长，就越不能满足及时响应请求（A） |
| CP   | 数据一致性、 分区容错 | 为了满足分区容错，则需要某个服务尽量多的集群部署（P）； 机器数目多了，数据一致性（C）就越耗时； 相对能及时响应请求就越难，所以很难保证高可用性（A） |
| AC   | 高可用性、 数据一致性 | 为了达到高可用，则必须要求及时处理请求，及时响应请求； 为了保持数据一致性，则需要每个机器中某个时刻数据保持一致； 一致性是耗时的操作，高可用又要保证及时响应，机器多了同步数据需要更多的时间，所以需要机器越少才能满足 |

但是，在分布式服务架构中，必须保证某些服务集群部署，所以分区容错性必须能够得到保证。

------

因此，在分布式开发中，结合CAP定理的要求，我们必须要保证P的要求，所以C和A的选择方面只需要保证服务支持AP或CP即可。

## 二、服务注册中心

在微服务分布式开发中，我们需要保证AP和CP，那支持这两种的有哪些架构呢？

### 2.1、服务发现组件对比

| 名称      | CAP  | 概述                                                         |
| --------- | ---- | ------------------------------------------------------------ |
| Zookeeper | CP   | 主要做服务的协同。亮点在于一致性比较好。 不接受服务直接宕掉不可用。 当某个节点不可用时，会采取选举策略，踢出当前服务，在这个时间段内整个ZK服务不可用（注册服务短时间内瘫痪） |
| Eureka    | AP   | 每个节点都是平等的，只要有一个eureka在，服务就可以保证(可能有少许延迟) |

### 2.2、eureka相关

#### 2.2.1、源码

《[eureka-github官方源码](https://github.com/Netflix/eureka)》

#### 2.2.2、eureka理解

在eureka架构中，参见：《[High level architecture](https://github.com/Netflix/eureka/wiki/Eureka-at-a-glance#high-level-architecture)》有一个架构图形，我们现在去围观这个架构的格局。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200102154910102.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
我们主要研究这一部分的操作
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200102155031607.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
①：register，注册。每个服务都必须先注册至指定的注册中心中。
②：get，获取。当服务和服务之间需要进行通信时，当前这个服务需要从注册中心上获取(定时获取)指定服务所在机器的标识信息(ip+port等)，才能进行rest访问。
③：Make Rempte Call，做请求获取结果。
④：Replicate，互相注册。做服务注册中心集群配置时，需要让各个eureka注册中心互相注册。

## 三、eureka注册中心的简单配置

说起eureka注册中心的配置问题，我们首当其冲的应该去了解他的结构问题。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200102164003279.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
①：服务A和服务B都需要在Eureka上进行注册。
②：当服务A需要调用服务B的接口时，则需要从Eureka上拿到服务B注册时的相关信息。
③：服务A再请求服务B，实现业务的处理操作。

只有了解了结构问题，我们才知道如何去配置服务注册中心，如何让服务进行注册、注册到哪里等问题。

### 3.1、注册中心的搭建

本次搭建采取 Spring Boot 2.X实现，随便2.X之前的版本也能用，但是2.X前后变更挺大的。

#### 3.1.1、依赖的引入

```xml
<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.4.RELEASE</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>
	<properties>
		<java.version>1.8</java.version>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
	</properties>

	<!-- 管理依赖 -->
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Finchley.M7</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<dependencies>
		<!--SpringCloud eureka-server -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>
	</dependencies>
	<!-- 注意： 这里必须要添加， 否者各种依赖有问题 -->
	<repositories>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/libs-milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
	</repositories>
	
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051
```

#### 3.1.2、配置文件的编写

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
    
  server:
  # 测试时关闭自我保护机制，保证不可用服务及时踢出
    enable-self-preservation: false #关闭自我防护机制
    eviction-interval-timer-in-ms: 2000

123456789101112131415161718192021
```

#### 3.1.3、启动类增加注解

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaSercice10000{
    public static void main(String[] args) {
		SpringApplication.run(EurekaSercice10000.class, args);
	}
}
1234567891011
```

#### 3.1.4、启动项目

启动项目，访问连接：localhost:10000，出现如下所示界面，表示配置、启动成功：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200102174124987.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

### 3.2、服务的创建和注册

创建、配置好了注册中心后，我们也启动了注册中心，但是此时的注册中心上没有服务，我们需要编写一个服务的提供者和服务的消费者，并注册至指定的注册中心上。

这里我就只写一个服务的配置操作，毕竟两个服务的配置信息都是差不多的，唯一的区别在于端口信息不同和注册服务中心时名称的不同。

#### 3.2.1、依赖的引入

```xml
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.4.RELEASE</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<java.version>1.8</java.version>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
	</properties>

	<!-- 管理依赖 -->
	<dependencyManagement>
		<dependencies>

			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Finchley.M7</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>

		</dependencies>
	</dependencyManagement>
	<dependencies>
		<!-- SpringBoot整合Web组件 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<!-- SpringBoot整合eureka客户端 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
	</dependencies>
	<!-- 注意： 这里必须要添加， 否者各种依赖有问题 -->
	<repositories>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/libs-milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
	</repositories>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859
```

#### 3.2.2、服务消费者配置文件的编写

```yml
###服务启动端口号
server:
  port: 10001
###服务名称(服务注册到eureka名称)  
spring:
    application:
        name: app-bunana-consumer-10001
###服务注册到eureka地址
eureka:
  client:
    service-url:
           defaultZone: http://localhost:10000/eureka
       
	#自己是服务，需要注册至服务注册中心上
    register-with-eureka: true
	#如果需要通信，必须运行此服务可以拉去别的服务的注册信息
    fetch-registry: true
1234567891011121314151617
```

#### 3.2.3、启动类的编写

需要在springboot启动类上增加注解 @EnableEurekaClient；
如果存在rest通信，则需要增加配置

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableEurekaClient
public class Consumer10001 {
   public static void main(String[] args) {
	SpringApplication.run(Consumer10001.class, args);
   }
   
   @Bean
   @LoadBalanced //开启负载均衡
   public RestTemplate restTemplate(){
	   return new RestTemplate();
   }
}
1234567891011121314151617181920
```

> @LoadBalanced 注解，不仅可以开启负载均衡效果，还能在分布式架构中，让服务和服务之间采取别名的方式进行请求操作。

#### 3.2.4、编写一个开放的接口

我们需要编写一个开放的请求接口，让其能够进行请求操作。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020010315242772.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

#### 3.2.5、创建一个服务的提供者

我们上面创建了服务的消费者，消费者的作用是当我们请求指定的接口时，传递数据，并调用服务生产者提供的服务，进行业务处理操作。
我们已经创建了消费者了，所以此时需要创建一个服务的提供者。

服务生产者的配置和消费者配置差不多，依赖引入也没什么区别，此处我们省略pom依赖引入问题，只写几个需要区别的地方。

##### 3.2.5.1、配置文件编写

配置文件方式区别在于注册至服务注册中心时的别名和端口不一样，我们修改下。

```yml
###服务启动端口号
server:
  port: 10010
###服务名称(服务注册到eureka名称)  
spring:
    application:
        name: app-bunana-product
###服务注册到eureka地址
eureka:
  client:
    service-url:
           defaultZone: http://localhost:10000/eureka
       
  #自己是服务，需要注册至服务注册中心上
    register-with-eureka: true
  #如果需要通信，必须运行此服务可以拉去别的服务的注册信息
    fetch-registry: true
1234567891011121314151617
```

##### 3.2.5.2、对外提供消费接口

```java
@RestController
@RequestMapping("/product")
public class TestController {
	
	@Value("${server.port}")
	private String port;
	
	@RequestMapping("/getProduct")
	public String getTest1(String name){
		return "this is product project getProduct name = "+String.valueOf(name)+",port="+port;
	}
}
123456789101112
```

#### 3.2.6、修改服务消费者信息

我们此时做出了服务提供者的接口，服务消费者需要请求调用提供者给的接口信息，所以我们修改下测试controller。

> 因为有了服务注册中心，所以我们只需要进行使用别名请求的方式进行调用即可。

```java
@RestController
@RequestMapping("/test")
public class TestController {
	
	@Autowired
	private RestTemplate restTemplate;
	
	@RequestMapping("/test1")
	public String getTest1(String name){
		String memberUrl = "http://app-bunana-product/product/getProduct?name="+String.valueOf(name);
		String result = restTemplate.getForObject(memberUrl, String.class);
		return "this is consumer project ,get product result = "+String.valueOf(result);
	}
}
1234567891011121314
```

#### 3.2.7、测试

1、启动服务注册中心。
2、分别启动 服务提供者、服务消费者。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103154723628.png)
3、请求接口测试，查看返回数据。

> localhost:10001/test/test1?name=66666
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103161027491.png)

代码参考：《[SpringCloud2.0详细配置——简单注册中心和服务配置](https://github.com/765199214/springcloud2.0-detailConfig/tree/master/1 简单注册中心和服务注册)》

#### 3.2.8、问题分析

问题一：当我请求出现异常，如下异常所示：

> java.net.UnknownHostException:

此时你需要注意，在我将RestTemplate注入至容器中时，是否添加有 @LoadBalanced 注解。
@LoadBalanced的作用：

> 1、让服务和服务之间，采取别名的方式进行请求。
> 2、当相同的服务提供者有多个时，可以保证服务消费者负载均衡式消费服务。

#### 3.2.9、服务——负载均衡配置

上面既然说到了负载均衡，那我们就测试下服务提供者集群部署。
我们也不需要再创建另外的新项目，只需要将服务提供者的端口号变更一个再运行一次(之前的不要close)就行了。

既然这样，我们修改服务提供者端口号，由 10010 变更为 10011。

```yml
server:
  port: 10011
12
```

启动项目后，查看注册中心。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103164708274.png)
集群部署时，该服务的名称绝对要求一样!

我们再请求上述接口：

> localhost:10001/test/test1?name=66666
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103164827766.png)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103164844423.png)

### 3.3、探究服务的另一个注解

```yml
###服务启动端口号
server:
  port: 10001
###服务名称(服务注册到eureka名称)  
spring:
    application:
        name: app-bunana-consumer-10001
###服务注册到eureka地址
eureka:
  client:
    service-url:
           defaultZone: http://localhost:10000/eureka
       
  #自己是服务，需要注册至服务注册中心上
    register-with-eureka: true
  #如果需要通信，必须运行此服务可以拉去别的服务的注册信息
    fetch-registry: true
    
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.application.instance_id:${server.port}}
123456789101112131415161718192021
```

对比上面的注解配置，可以发现，我此处新增了一个新的eureka注解配置信息：

```yml
eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.application.instance_id:${server.port}}
1234
```

不加此注解时：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103162802832.png)
加了后，显示如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103162958114.png)
由此可见，此处两个注解，可以在服务注册至注册中心时，简化显示别名信息。

### 3.4、eureka和服务之间的心跳、超时踢出设置

> 1、当我们的Product服务注册至服务中心上后，eureka会采取心跳机制，判断服务的在线情况。
> 2、当某一时刻，该服务宕机了！
> 3、eureka监测到该服务最后一次上报心跳到现在有30秒了，会再等待60秒时间，保证服务不会被踢出注册中心。
> 4、如果这60秒内服务依旧无法建立心跳，eureka则会将服务踢出注册中心。
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103180642946.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

问题：那么心跳时间或者服务保护开关可以设置吗？或者如何设置？
1、服务注册中心的设置(service)：

```yml
eureka:
  server:
  # 测试时关闭自我保护机制，保证不可用服务及时踢出
    enable-self-preservation: false #关闭自我防护机制
    eviction-interval-timer-in-ms: 2000 #服务宕机后，从发现到踢出之间的间隔时间(默认60*1000)
12345
```

2、服务中的配置(client)：

```yml
eureka:
  instance:
    #Eureka客户端向服务端发送心跳的时间间隔，单位为秒（客户端告诉服务端自己会按照该规则），默认30
    lease-renewal-interval-in-seconds: 5
    #Eureka服务端在收到最后一次心跳之后等待的时间上限，单位为秒，超过则剔除（客户端告诉服务端按照此规则等待自己），默认90
    lease-expiration-duration-in-seconds: 7
123456
```

------

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103181854894.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
启动服务注册中心后再启动子服务，保证子服务能都注册到注册中心中。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103173345715.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
测试1：然后我们将这个子服务宕掉，看多少时间会自动踢出。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103173457204.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

> 2秒将宕掉的服务踢出注册中心，和我们配置的一样(子服务配置)！

此处配置项，参考文章：《[Eureka 开发时快速剔除失效服务](https://blog.csdn.net/haveqing/article/details/88406592)》

测试2：当服务重启后，能否再注册上去呢？
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020010317383879.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

> 项目重启之后，依旧可以重新注册至注册中心上！

注意：注意一个问题：

原则上的配置：设置定时清理时间,这个时间可以自己定义,但是也要给服务之间相互预留重连时间,不能说一旦宕机立马移除,这样容易错杀！
上面我们只是采取测试操作，所以eureka对子服务掉线踢出时间设置为2秒，子服务心跳周期上报间隔为5秒一次；
eureka收到子服务中的最后一次心跳上报等待的时间上限设置7秒(默认90秒)；

### 3.5、存在的安全隐患

上面的配置，我们直接采取localhost:10000就能访问我的eureka注册中心。
当我的项目上线之后，你们知道了我的域名和我的端口号信息，如果我的服务器没有做端口访问策略，岂不是很危险？
所以我们接下来来说一个eureka的安全配置问题。

#### 3.5.1、注册中心pom依赖引入

```xml
<!-- 为Eureka 增加安全访问 -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-security</artifactId>
	</dependency>
12345
```

#### 3.5.2、注册中心配置文件的修改

完整配置为：

```yml
###服务端口号
server:
  port: 10000
#eureka安全模块
spring:
  security:
    basic:
      enabled: true
    user:
      name: xiangjiao
      password: bunana
    
###eureka 基本信息配置
eureka:
  instance:
    ###注册到eurekaip地址
    hostname: 127.0.0.1
  client:
    serviceUrl:
      defaultZone: http://${spring.security.user.name}:${spring.security.user.password}@${eureka.instance.hostname}:${server.port}/eureka/
      #defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
    #取消向eureka注册中心注册(自己本身就是注册中心，自己不需要注册自己)
    register-with-eureka: false
    #取消从注册中心上获取注册信息
    fetch-registry: false
    
  server:
  # 测试时关闭自我保护机制，保证不可用服务及时踢出
    enable-self-preservation: false #关闭自我防护机制
    eviction-interval-timer-in-ms: 2000 #服务宕机后，从发现到踢出之间的间隔时间(默认60*1000)
123456789101112131415161718192021222324252627282930
```

启动服务注册中心，访问eureka后，则会出现如下所示的界面，输入账号密码即可访问。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103191902253.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

#### 3.5.3、子服务依赖和配置修改

当注册中心增加了安全策略，服务需要注册至注册中心中，此时必须讲究服务也能通过配置的安全策略。

> 子服务无需引入security依赖，只需要修改eureka注册中心地址即可！！

```yml
eureka:
  client:
    service-url:
           defaultZone: http://xiangjiao:bunana@localhost:10000/eureka
1234
```

当然，这样启动eureka注册中心可以进去，也需要密码登录才能正常进入，但子服务却不无法注册至服务中心上！！！！
重点：eureka安全security影藏bug！！

> 为什么会出现 com.netflix.discovery.shared.transport.TransportException: Cannot execute request on any known server ？
> 还有这是什么鬼？？？

让我耐心简单的做说明！！

> eureka开启安全策略后，在新版本的security中，添加了csrf过滤,csrf将微服务的注册也给过滤了！

那怎么解决这个问题呢？
既然是注册中心引入了security依赖导致的问题，我们只需要在eureka注册中心中对csrf过滤进行屏蔽就可以了！！

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.csrf().disable(); // 关闭csrf
		super.configure(http);
	}

}
12345678910111213141516
```

重新启动eureka注册中心和各个服务项，查看是否注册成功。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200103201810600.png)
参考代码：《[springcloud2.0详细配置——eureka安全配置和服务注册](https://github.com/765199214/springcloud2.0-detailConfig/tree/master/2 eureka安全配置和服务注册安全配置)》

### 3.6、eureka集群化配置

正常的项目开发中，我们不可能只使用一台注册中心，当我们只有一台时，注册中心宕机了，整个项目就崩了，不切实际，所以我们也需要对注册中心进行集群化配置。

既然服务可以集群化配置，那eureka注册中心理应可以集群化配置。

我们之前在注册中心中，配置了服务注册中心两个关注点：

> 1、不注册自己：eureka.client.register-with-eureka=false
> 2、不拉取注册信息：eureka.client.fetch-registry=false

#### 3.6.1、eureka服务注册中心的配置

既然要集群化配置，我们就必须保证由多个eureka注册中心，并保证他们能够互相注册，互相拉取信息。

所以我们如果配置eureka集群，只需要更改配置文件中几个位置就行了。
1、增加集群时，eureka别名称

```yml
spring:
  application: #给定集群时的别名称
    name: eureka-server
```

2、集群，必须要将备用的eureka注册至已使用的eureka上，并能拉取服务（此处eureka端口为10000，所以集群配置端口为11000和12000）

```yml
eureka:
  client:
  	serviceUrl:
  	  #分别注册至其他的注册中心上，所以这里的端口不是自己的端口信息！！
  	  #如果集群有多个(假如3个，此处连接采取","拼接)
      defaultZone: http://xiangjiao:bunana@localhost:11000/eureka/,http://xiangjiao:bunana@localhost:12000/eureka/
###因为自己是为注册中心，不需要自己注册自己(集群时开启)
    register-with-eureka: true
###因为自己是为注册中心，不需要检索服务(集群时开启)
    fetch-registry: true
12345678910
```

关于多个eureka构成集群配置的可以参考我下面的代码，在测试完成后，本小节末会给出git代码地址。

注册中心这么配置就可以了，启动eureka注册中心，看效果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200104134741688.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
后面几个注册中心的登录进去的效果图就不展示了。
我们配置好了集群化的注册中心，如何让我们的服务能够注册至注册中心上呢？所以，我们接下来配置子服务注册至注册中心

#### 3.6.2、子服务的配置和注册

针对各个服务，能够注册至注册中集群上，我们只需要变更一项配置即可
修改配置文件信息：

```yml
eureka:
  client:
    service-url:
           defaultZone: http://xiangjiao:bunana@localhost:10000/eureka/,http://xiangjiao:bunana@localhost:11000/eureka/,http://xiangjiao:bunana@localhost:12000/eureka/

12345
```

增加注册到各项服务注册中心的地址即可！
启动子服务，我们查看注册情况

只是学习，无需要进行复杂的加密安全的话，集群配置可以参考我的这篇文章：《[Spring Cloud 学习(四)——Eureka集群配置](https://blog.csdn.net/qq_38322527/article/details/100188545)》

关于本次添加安全策略的基础上，新增了eureka集群部署和子服务服务注册配置的代码，可以查看：《[github代码地址](https://github.com/765199214/springcloud2.0-detailConfig/tree/master/3 eureka集群安全配置和子服务配置)》