# SpringCloud深入学习(五)——Hystrix的简介以及降级、限流、超时、熔断机制

https://blog.csdn.net/qq_38322527/article/details/104253116

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[专注写bug](https://me.csdn.net/qq_38322527) 2020-02-28 16:34:04 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 251 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 1



分类专栏： [springcloud](https://blog.csdn.net/qq_38322527/category_8898018.html)

版权

## 一、了解雪崩

在分布式项目中，往往出现最多的情况是服务的雪崩现象，导致单独微服务的宕机或者整个项目的奔溃。
[问：]什么是服务的雪崩效应？

> 在微服务与微服务之间，通信操作如下所示：
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200210192759632.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
> 正常的服务A的操作，需要服务B和服务C的共同完成。
>
> 但是在请求操作处理过程中，假如 服务B 在业务上处理过于繁琐，导致响应至服务A不够及时。
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200210192953424.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
> 此时将会导致整个服务A的响应变得迟缓。如果此时是大流量请求访问，就像双11的抢购环境，大量的请求将会堆积在服务A中，导致整个服务的服务器资源耗尽(内存、cpu、线程 等)，从而引发服务的宕机。

此思想源于《[浅谈服务雪崩 --记忆挽留(简书)](https://www.jianshu.com/p/acfb4ac2b124)》

雪崩的出现，会逐渐导致整个微服务的不可用现象，严重者会导致整个项目的瘫痪。
[问：]那么，出现微服务不可用的原因有哪些呢？
1、程序端

> a、大流量的请求操作。
> b、子服务所在的服务器宕机，导致这个服务不可用，但别的服务却又不断地请求。
> c、程序地业务逻辑过于复杂，或存在bug。
> d、缓存击穿—电脑重启，redis中地信息还未成功重新加载时。

2、用户端

> a、当用户使用网址时，如果半天加载不出，通常会选择手动刷新页面。
> b、或者是开发者请求指定服务回调中，写了别的代码逻辑造成相对的“死循环”(比如ajax成功/失败回调 继续重试请求等)

[总结：]

> 线程是在服务器中占用资源的。
> 大量的请求线程同步等待造成的服务器资源耗尽。

[问：]针对出现某个子服务不可用会造成整个项目的雪崩现象，我们应该如何面对和处理？

> 接下来我们一起看服务雪崩的解决思路。

## 二、服务雪崩的解决思路

- 超时机制
	服务之间的调用，设置请求超时时间。达到限制时间依旧未请求得到数据，就给前台页面一个正在刷新的显示图标或提示。
- 服务降级
	和超时时间设置相类似，超过请求响应时间，给予一个默认的提示信息fallback 回调，返回一个缺省值。(备用接口、缓存、默认数据等)。
- 服务限流
	设置某个容易出现问题的服务的请求线程数，当满足当前的请求线程数后，新的请求线程访问就直接进行 fallback ，及时释放线程。避免整体的资源不会被出问题的服务消耗殆尽。
- 服务熔断
	当出现网络波动或者服务不稳定时，暂时关闭问题服务。
	当设置了该服务的响应时间为3秒，某个线程请求后得不到这个服务的回调信息，可以暂定这个服务出现了异常，此时就没有必要让后面的请求继续去请求访问这个服务了。
	当服务稳定后，在开启该服务即可。

## 三、Hystrix 服务降级 配置

上面我们大致说明了服务发生雪崩的情况，以及相关的雪崩解决思路，但我们依旧只是停留在理论理解的层面上，接下来我们需要进行实际的代码测试。

在一般的微服务中，整体的项目调用结构如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200224180323490.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
[问：]那么我们服务熔断降级的逻辑代码放于何处？是在 Consumer 中还是 Product 中呢？

> 答案显而易见，我们需要在Consumer中制定服务保护的相关逻辑。
>
> > 若我们逻辑写在了Product中，Consuner调取Product时，当此时Product服务宕机，我们写的逻辑根本就不会用上。

### 3.1、product子服务提供(服务提供方)

我们此时需要创建一个微服务，注册至注册中心中。当前的微服务需要满足的条件：

> 1、提供可供消费者调用的接口。
> 2、注册至指定的eureka注册中心上。

我们只需要随便创建一个服务即可，当然也需要添加pom依赖，配置文件编写等，如下所示。

##### 1、pom依赖的引入

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

从依赖文件中我们可以发现，我们此时只引入了eureka-client和springboot 2.x的pom。

##### 2、配置文件的编写

我们在配置文件中需要提供注册至注册中心的配置信息、当前的微服务别名信息、当前服务的端口信息等。

```yml
###服务启动端口号
server:
  port: 10011
###服务名称(服务注册到eureka名称)  
spring:
    application:
        name: app-bunana-product
###服务注册到eureka地址
eureka:
  client:
    service-url:
           defaultZone: http://localhost:10000/eureka/
       
  #自己是服务，需要注册至服务注册中心上
    register-with-eureka: true
  #如果需要通信，必须运行此服务可以拉去别的服务的注册信息
    fetch-registry: true
    
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.application.instance_id:${server.port}}
    #Eureka客户端向服务端发送心跳的时间间隔，单位为秒（客户端告诉服务端自己会按照该规则），默认30
    lease-renewal-interval-in-seconds: 5
    #Eureka服务端在收到最后一次心跳之后等待的时间上限，单位为秒，超过则剔除（客户端告诉服务端按照此规则等待自己），默认90
    lease-expiration-duration-in-seconds: 7

1234567891011121314151617181920212223242526
```

##### 3、编写启动类，添加注解

由于需要注册至注册中心，那么此时的项目就是一个eureka-client的项目。

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableEurekaClient
public class Product10010 {
   public static void main(String[] args) {
	SpringApplication.run(Product10010.class, args);
   }
}
1234567891011121314
```

##### 4、提供可供消费的接口

我们只需要制定一个可供请求调用的处理接口即可。

```java
import javax.servlet.http.HttpServletRequest;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

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
1234567891011121314151617
```

此时启动该项目，注册至注册中心中，我们就实现了app-bunana-product 服务的注册。接下来我们需要编写一个能够消费该服务接口的调用者(Consumer子服务)。

### 3.2、Consumer 子服务(服务消费者)

在上面的分析中，我们已经说明了 服务保护 的逻辑代码必须写在Consumer微服务中，所以我们此时的Consumer项目就需要使用到Hystrix。

所以本次项目中需要采用的框架有：

> Feign：做服务之间的请求、调用
> Hystrix：做服务保护熔断操作

我们想要使用Hystrix，就需要使用指定的pom依赖文件。

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
1234
```

光引入还不够，我们需要在配置文件中添加相关的配置文件。
在Feign框架中使用Hystrix时，Feign默认是关闭了Hystrix的，所以我们需要在配置中开启这方面的配置信息。

```yml
###服务启动端口号
server:
  port: 10001
###服务名称(服务注册到eureka名称)  
spring:
    application:
        name: app-bunana-consumer-hystrix-feign
        
##feign中使用断路器，默认是没有开启的，需要在配置文件中开启        
feign:
    hystrix:
        enable: true
###服务注册到eureka地址
eureka:
  client:
    #以下两种方式都可以
    #registry-fetch-interval-seconds: 20
    registry:
        fetch:
            interval:
                seconds: 20
    #registryFetchIntervalSeconds: 20
    service-url:
       defaultZone: http://localhost:10000/eureka/
       
  #自己是服务，需要注册至服务注册中心上
    register-with-eureka: true
  #如果需要通信，必须运行此服务可以拉去别的服务的注册信息
    fetch-registry: true
    
    
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.application.instance_id:${server.port}}
    #Eureka客户端向服务端发送心跳的时间间隔，单位为秒（客户端告诉服务端自己会按照该规则），默认30
    lease-renewal-interval-in-seconds: 5
    #Eureka服务端在收到最后一次心跳之后等待的时间上限，单位为秒，超过则剔除（客户端告诉服务端按照此规则等待自己），默认90
    lease-expiration-duration-in-seconds: 7
  
123456789101112131415161718192021222324252627282930313233343536373839
```

其中的这个配置表示开启Hystrix的设置。

```yml
##feign中使用断路器，默认是没有开启的，需要在配置文件中开启        
feign:
    hystrix:
        enable: true
1234
```

引入了依赖和修改了配置文件，依旧无法使用Hystrix实现标题的功能需求，我们还应该在该项目的启动类上添加开启Hystrix注解信息。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.SpringCloudApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.openfeign.EnableFeignClients;

@EnableEurekaClient		//使用Eureka
@EnableFeignClients	 //开启Feign
@EnableHystrix  //开启hystrix
//@SpringCloudApplication  //上述的三个注解，可以使用这一个注解代替
@SpringBootApplication
public class ConsumerFeignHystrix10001 {
	private static Logger log = LoggerFactory.getLogger(ConsumerFeignHystrix10001.class);
	
	public static void main(String[] args) {
		try {
			SpringApplication.run(ConsumerFeignHystrix10001.class, args);
		} catch (Exception e) {
			log.info("ConsumerFeignHystrix10001 Application exception-->{}",e);
		}
	}
}
12345678910111213141516171819202122232425
```

引入依赖，修改启动类和配置文件之后，我们需要增加一个测试类来完成标题的功能需求，由于我们这里采取的是Feign的通信方式，所以我们还需要定义Feign的服务业务。

```java
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

//添加feignClient注解并引用对应的别名
@FeignClient("app-bunana-product")
public interface ConsumerFeignClient {
	
	@RequestMapping("/product/getProduct")
	public String getTest1ByFeign(@RequestParam("name") String name);
}
1234567891011
```

关于Feign的文件编写，可以查看我专栏中的相关博客。
Feign的学习笔记地址：《[SpringCloud深入学习(四)——Feign的介绍和使用](https://blog.csdn.net/qq_38322527/article/details/104189889)》

编写完成Feign后，我们定义一个外围可以访问的接口。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import cn.linkpower.feignService.ConsumerFeignClient;

@RestController
public class TestController {
	
	@Autowired
	private ConsumerFeignClient consumerFeignClient;
	
	//采用feign调用方式，进行服务和服务之间的请求
	@RequestMapping("/test1")
	public String test1(String name){
		return consumerFeignClient.getTest1ByFeign(name);
	}
}
123456789101112131415161718
```

此时的我们还没有写 服务熔断保护 的逻辑程序，当我们运行代码时，也能注册至注册中心中，同时正常请求时，也有相应的回执信息。

> http://localhost:10001/test1?name=66666
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020022418245813.png)

但是，如果我们忘了提供相应的Product服务，或者Product服务宕机了，我们模拟测试–杀掉Product项目的进程。
此时的erueka中的注册子服务信息由最开始的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200224182654879.png)
变换为
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200224182708662.png)
上面的请求我们再刷新一遍，查看返回信息：

> http://localhost:10001/test1?name=77777
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200224182756619.png)

从请求结果返回中，我们可以看出，当服务的提供方宕机了，服务的消费方并不能拿到对应的请求接口，就会出现 500 的报错信息，如果返回给客户这样的页面显示情况，很不美观和友好，所以我们可以采取默认返回一个其他的信息作为替代品。

### 3.3、添加服务熔断保护机制

由于此时使用的是 Feign 做请求访问操作，我们需要在feign接口中定义一个 fallback 函数。指明对应的回执信息。

#### 1、修改feign 接口

```java
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;


@FeignClient(value="app-bunana-product",fallback=FallbackHystrix.class)
public interface ConsumerFeignClient {
	@RequestMapping("/product/getProduct")
	public String getTest1ByFeign(@RequestParam("name") String name);
}
12345678910
```

#### 2、编写fallback回执的实现类

```java
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;
import cn.linkpower.feignService.ConsumerFeignClient;

@Component 
public class FallbackHystrix implements ConsumerFeignClient {

	@Override
	public String getTest1ByFeign(String name) {
		return "服务错误，请联系开发者";
	}

}
12345678910111213
```

关闭服务并进行重启，继续访问。

> http://localhost:10001/test1?name=88888
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/202002241911209.png)

### 3.4、feign添加Hystrix的错误总结

#### 1、ClientException

这种问题，一般都是引入Hystrix失败，或者Feign的版本过高，或其他配置的问题。
我这里使用的 Springcloud 的版本为 2.1.4.RELEASE 。
其中相关的依赖文件信息为：
feign为：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
1234
```

Hystrix为：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
1234
```

还有最重要的一点，Feign 默认未开启Hystrix的熔断支持，需要在配置文件中开启相关的配置信息。

```yml
feign:
  hystrix:
    enabled: true
123
```

问题总结参考文献：
《[SpringCloud组件：Feign整合Hystrix实现熔断机制](https://www.jianshu.com/p/6f6ac6433d5e)》
《[com.netflix.client.ClientException: Load balancer does not have available server for client xxxx(注：参考评论，不是文章)](https://blog.csdn.net/jiadajing267/article/details/81047335)》

#### 2、容易出现请求失败

本次项目设置的心跳间隔和心跳超时时间比较短，只限于测试使用，如果容易出现请求失败，此时需要考虑设计好合适的时间值！

### 3.5、无Feign时Hystrix的配置

我们上面所述的是在项目中，需要使用到Feifn时，对需要使用Hystrix时的简单配置，但是有些公司还是喜欢最简单的RestTemplate的方式进行请求操作，此时，我们又该如何整合Hystrix实现服务的保护呢？

#### 1、Product 接口提供方

接口提供方，我们依旧可以使用上面的那个Product项目，无需做更改。

#### 2、Consumer 接口消费方

由于本次我们不采用Feign做请求操作，采用Springcloud中本身自带的Ribbon的RestTemplate，所以只需要在上述Consumer项目中移除Feign的依赖和配置。
当然，为了让RestTemplate也具有采取别名方式请求和负载均衡操作，我们也需要进行下相关的配置。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableEurekaClient
public class ConsumerHystrix10001Application {
	private static Logger log = LoggerFactory.getLogger(ConsumerHystrix10001Application.class);
	public static void main(String[] args) {
		try {
			SpringApplication.run(ConsumerHystrix10001Application.class, args);
		} catch (Exception e) {
			log.info("ConsumerHystrix10001Application start exception--->{}", e);
		}
	}
	
	@Bean
	@LoadBalanced
	public RestTemplate getRestTemplate(){
		return new RestTemplate();
	}
}
123456789101112131415161718192021222324252627
```

然后修改请求控制器即可。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
public class TestController {
	
	@Autowired
	private RestTemplate restTemplate;
	
	@RequestMapping("/test1")
	public String test1(@RequestParam String name){
		String memberUrl = "http://app-bunana-product/product/getProduct?name="+String.valueOf(name);
		String result = restTemplate.getForObject(memberUrl, String.class);
		return "this is Consumer,get result is ==="+String.valueOf(result);
	}
}
12345678910111213141516171819
```

我们上面依旧暂未配置Hystrix，启动项目、注册中心，分别再测试正常和异常情况下的访问操作。

> http://localhost:10001/test1?name=77777
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226204147593.png)

上面是正常情况下，必然会出现的情况，但是我们依旧需要考虑 Consumer 调取 Product 接口时，若出现超时或者Product宕机情况下，如何友好提示用户的操作。

#### 3、配置Hystrix

我们需要使用Hystrix实现服务的熔断保护操作，就需要再启动类上新增相关的开启注解 @EnableHystrix 。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableEurekaClient
@EnableHystrix
public class ConsumerHystrix10001Application {
	private static Logger log = LoggerFactory.getLogger(ConsumerHystrix10001Application.class);
	public static void main(String[] args) {
		try {
			SpringApplication.run(ConsumerHystrix10001Application.class, args);
		} catch (Exception e) {
			log.info("ConsumerHystrix10001Application start exception--->{}", e);
		}
	}
	
	@Bean
	@LoadBalanced
	public RestTemplate getRestTemplate(){
		return new RestTemplate();
	}
}
1234567891011121314151617181920212223242526272829
```

同时，也需要针对接口请求调用超时或异常时，做配置操作。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226205744370.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)
重启项目，访问即可
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226205808478.png)

> http://localhost:10001/test1?name=88888
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200226205823187.png)

@RequestParam 在fallback函数中的参数前也可以不加这个注解

[注意：]此处有坑！
当我们的接口设置了请求参数时，如果Hystrix的fallback方法中，未定义和接口相同的参数类型，此时则会出现一个报错信息。

> com.netflix.hystrix.contrib.javanica.exception.FallbackDefinitionException:

所以此处的fallbackmethod方法中，参数需要和原接口的参数类型数目等保持一致。

### 3.6、无feign时的hystrix的另一种操作

最近查看各种资料了解学习Hystrix服务保护，了解到还有另外一种比较繁琐的实现方式，这里也记录下这种方式吧，万一哪天有需要用到呢。

我们修改Consumer类的实现方式，采取 Hystrix 的一种叫 “命令模式”的方案来实现。

> 继承 HystrixCommand 类，来包裹具体的服务调用逻辑(Run())。
> 并在命令模式中添加服务调用失败后的降级逻辑(getFallback)。

我们此时按照上面所述，新增一个处理操作类，继承 HystrixCommand 类，同时重写 Run()和getFallback()；

```java
import org.springframework.web.client.RestTemplate;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;

public class ServiceCommand extends HystrixCommand<String> {
	
	//请求操作，需要携带的参数信息
	private RestTemplate restTemplate;
	private String name;
	
	public ServiceCommand(String key, RestTemplate restTemplate, String name) {
		super(HystrixCommandGroupKey.Factory.asKey(key));
		this.restTemplate = restTemplate;
		this.name = name;
	}
	
	@Override
	protected String run() throws Exception {
		String memberUrl = "http://app-bunana-product/product/getProduct?name="+String.valueOf(name);
		return restTemplate.getForObject(memberUrl, String.class);
	}
	
	@Override
	protected String getFallback() {
		//返回默认信息，或者失败的提示信息
		return "请求调用失败";
	}
}
1234567891011121314151617181920212223242526272829
```

再添加一个可供请求测试的接口，实现测试操作。

```java
	@RequestMapping("/test2")
	public String test2(@RequestParam String name){
		//这里的key可以任意取名，只是这种方式必须传递这个参数而已
		ServiceCommand serviceCommand = new ServiceCommand("random", restTemplate, name);
		return serviceCommand.execute();
	}
123456
```

启动项目，进行访问，此时的注册中心的情况为：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227133411468.png)
请求测试：

> http://localhost:10001/test2?name=88888
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227133429133.png)

上面的测试只是异常测试，我们测试正常情况下，是否会有相关的请求回调信息。
启动Product项目，并注册至注册中心中。此时的注册中心情况如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227133606294.png)
请求测试：

> http://localhost:10001/test2?name=99999
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227133708998.png)

## 四、Hystrix 超时配置

Hystrix 调用接口默认超时时间为 1秒，当接口请求超时了，也会触发降级。
[问：]我们如何进行超时的测试呢？

> 既然是接口调用接口，1s后无返回信息，定义为超时。
> 我们就可以在Product 子服务中新增一个 延迟等待操作。

接下来我们修改Product服务的接口处理逻辑，添加一个等待的随机时间。
Product：

```java
import java.util.Random;

import javax.servlet.http.HttpServletRequest;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
@RequestMapping("/product")
public class TestController {
	
	private static Logger log = LoggerFactory.getLogger(TestController.class);
	
	@Value("${server.port}")
	private String port;
	
	@RequestMapping("/getProduct")
	public String getTest1(String name) throws InterruptedException{
		int timeout = new Random().nextInt(3000);
		log.info("此时的超时时间--->{}",String.valueOf(timeout));
		Thread.sleep(timeout);
		return "this is product project getProduct name = "+String.valueOf(name)+",port="+port;
	}
}
1234567891011121314151617181920212223242526272829
```

启动项目，注册至注册中心中，我们访问不断刷新查看情况。

> http://localhost:10001/test2?name=66666666666
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227142609251.png)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227142624255.png)

> http://localhost:10001/test2?name=66666666666
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227142652555.png)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227142704428.png)

[问：]既然有默认超时1s的设置，那我们能否设置其他的时间呢？
在[SpringCloud](https://cloud.spring.io/spring-cloud-static/Greenwich.SR5/single/spring-cloud.html#_circuit_breaker_hystrix_clients)中，针对Hystrix有一个超时时间的配置参数：

> hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 60000

我们添加至 Consumer中，再次启动项目并注册至注册中心中，请求接口，查看返回状态信息。

> http://localhost:10001/test2?name=66666666666
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227144047261.png)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227144106414.png)

[问：]除了超时时间配置外，还有别的相关配置信息吗？

> hystrix.comand.default.executlop.timeout.enabled 执行是否启用超时，默认启用true,
> hystrix.command.default.executipn.isolation.thread.interruptonTimecut发生超时是否中断，默认true
> hystrix.command.default.execution.isolation.strategy 隔离策略，默认是Thread，可选 Thread | Semaphore

相关demo在文章末尾有github下载地址。

## 五、Hystrix 熔断机制

我们假设Product服务出现逻辑bug。

> 当我们发送 name=“1” 时，表示出错，
> 如果此时Consumer的这个接口访问Product时，出现的错误过多，
> 我们如何去在下次请求时，直接进行fallback，而不是继续执行其逻辑？

我们此时修改 Product 提供的接口的处理逻辑。

```java
import java.util.Random;

import javax.servlet.http.HttpServletRequest;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
@RequestMapping("/product")
public class TestController {
	
	private static Logger log = LoggerFactory.getLogger(TestController.class);
	
	@Value("${server.port}")
	private String port;
	
	@RequestMapping("/getProduct")
	public String getTest1(String name) throws InterruptedException{
		//int timeout = new Random().nextInt(3000);
		//log.info("此时的超时时间--->{}",String.valueOf(timeout));
		//Thread.sleep(timeout);
		
		//此处 我们模拟 当 name = "1" 时，出现业务报错
		if("1".equalsIgnoreCase(name)){
			throw new NullPointerException();
		}
		return "this is product project getProduct name = "+String.valueOf(name)+",port="+port;
	}
}
12345678910111213141516171819202122232425262728293031323334
```

同时，修改Consumer类中的接口，达到的需求：

> 当熔断后，不会进入接口中，直接进入fallback回调函数中。

Consumer 中相比原来的接口逻辑，只是添加了一条日志打印信息，达到熔断成功与否的判断。

```java
	@RequestMapping("/test1")
	@HystrixCommand(fallbackMethod="testfallback")
	public String test1(@RequestParam String name){
		log.info("请求调用 --- test1 --- name ===={}",String.valueOf(name));
		String memberUrl = "http://app-bunana-product/product/getProduct?name="+String.valueOf(name);
		String result = restTemplate.getForObject(memberUrl, String.class);
		return "this is Consumer,get result is ==="+String.valueOf(result);
	}
12345678
```

启动项目，注册至注册中心上，测试查看返回信息。

> http://localhost:10001/test1?name=777
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227155707180.png)

> http://localhost:10001/test1?name=1
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227155729972.png)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227170222365.png)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227155750492.png)

此时的Product中，已经具备了当请求参数为 “1” 时，会出现错误信息的逻辑，我们接下来就是去让hystrix在请求中，Product出现错误达到指定次数时，进行服务的熔断。

设定超过多少次进行熔断
我们在Consumer的配置文件中，新增如下配置：

```yml
#hystrix相关配置
hystrix:
  command:
    default:
      execution:  ##超时配置
        isolation:
          thread:
            timeoutInMilliseconds: 60000   #设置请求超时时间，默认1秒，超过指定的时间后，触发降级
      circuitBreaker:  ##熔断配置
        requestVolumeThreshold: 3 #达到指定的次数后熔断服务(默认 20次---连续出现20次的请求报错就会进入熔断隔离)
        sleepWindowInMilliseconds: 10000 #熔断后，多长时间进行恢复(默认5000)
1234567891011
```

添加了上面的配置信息后，我们重新启动项目，在进行访问测试(不断地请求)：

> http://localhost:10001/test1?name=1
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200227170657519.png)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200228142551234.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

[总结：] 通过测试我们发现：

> 测试正常和报错的情况。
> 1、我们发现当出错到一定次数(默认20，这里设置的3)，正常和有问题的请求都会被隔断。
> 2、当限定时间过去后(默认5000，这里设置的10000)，又能继续请求了。

[问：]触发熔断是连续失败触发？还是说规定时间内触发？

> 取决于设置的 requestVolumeThreshold(默认20次)和sleepWindowInMilliseconds(默认5000)这两个参数值。
> 当5000ms内，若连续出现 20次失败，才会出现熔断。
> 即使在5s内出现19次也不会触发熔断隔离。

[注：]此处还有个重点：

> 1、当连续3次出现异常，需要等待10秒后才能隔离释放；若隔离期间(隔离时间段内)内继续请求，则会重新计时！
> 2、隔离成功后释放，若释放后的那一次继续是出问题的，则会继续重新计时！

## 六、Hystrix的限流机制

之前就有说到服务限流的简单原理。这里还是继续再说一下吧，博客上面也有，懒得去翻找。

> 设置某个容易出现问题的服务的请求线程数，当满足当前的请求线程数后，新的请求线程访问就直接进行 fallback ，及时释放线程。避免整体的资源不会被出问题的服务消耗殆尽。

围绕这个思路，我们可以让请求处理在逻辑中阻塞一段时间，达到线程资源的消耗。

### 1、修改Product子服务的处理流程，新增线程逻辑等待延迟

```java
import java.util.Random;

import javax.servlet.http.HttpServletRequest;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
@RequestMapping("/product")
public class TestController {
	
	private static Logger log = LoggerFactory.getLogger(TestController.class);
	
	@Value("${server.port}")
	private String port;
	
	@RequestMapping("/getProduct")
	public String getTest1(String name) throws InterruptedException{
		//int timeout = new Random().nextInt(3000);
		//log.info("此时的超时时间--->{}",String.valueOf(timeout));
		//Thread.sleep(timeout);
		
		//此处 我们模拟 当 name = "1" 时，出现业务报错
//		if("1".equalsIgnoreCase(name)){
//			throw new NullPointerException();
//		}
		//测试限流，让每个请求阻塞3秒
		Thread.sleep(3000);
		return "this is product project getProduct name = "+String.valueOf(name)+",port="+port;
	}
}
123456789101112131415161718192021222324252627282930313233343536
```

可以很直白的看到，我在这个接口中，每次请求来后，都会延迟一段时间后，在做处理。

```java
//测试限流，让每个请求阻塞3秒
Thread.sleep(3000);
12
```

### 2、除此之外，我们也需要修改Consumer服务

我们先设置超时机制的时间，由于默认1秒，为了测试限流机制，这个时间太短了，必须要更改其超时机制的时间。我们将默认的1秒更改为 20秒

```yml
#hystrix相关配置
hystrix:
  command:
    default:
      execution: ## 超时配置
        isolation:
          thread:
            timeoutInMilliseconds: 20000   #设置请求超时时间，默认1秒，超过指定的时间后，触发降级
12345678
```

当我们设置完超时时间后，我们还需要设置Hystrix针对限流机制触发的线程数限制等。

```java
	/**
	 * 用于限流操作的配置
	 * @param name
	 * @return
	 */
	@RequestMapping("/test3")
	@HystrixCommand(fallbackMethod="testfallback3",
					groupKey="test3",
					threadPoolKey="test3ThreadPool", //配置全局唯一标识线程池的名称
					threadPoolProperties={	//配置线程池的参数信息
							@HystrixProperty(name="coreSize",value="2") //coreSize配置核心线程池的大小和线程池最大大小(默认10，超过这个数后直接降级)
							//,@HystrixProperty(name="keepAliveTimeMinutes",value="60000") //keepAliveTimeMinutes配置线程池中空闲线程的生存时间，不配置则无作用
							//,@HystrixProperty(name="maxQueueSize",value="1") //maxQueueSize线程池队列最大大小
							//,@HystrixProperty(name="queueSizeRejectionThreshold",value="6") //限定当前队列大小
					}
				)
	public String test3(@RequestParam String name){
		log.info("请求调用 --- test3 --- name ===={}",String.valueOf(name));
		String memberUrl = "http://app-bunana-product/product/getProduct?name="+String.valueOf(name);
		String result = restTemplate.getForObject(memberUrl, String.class);
		return "this is Consumer,get result is ==="+String.valueOf(result);
	}
	
	//Hystrix的失败回调函数
	public String testfallback3(String name){
		return "异常了。。。。";
	}
123456789101112131415161718192021222324252627
```

### 3、启动并注册至注册中心上，并进行相关的测试操作

> 此处可以使用Jmeter进行压池测试显示。

[总结：]发现：
1、当我只配置了 fallbackMethod=“testfallback3”

> 压池操作，设置多个线程数(大于10)，且都是同一时间请求时，只有10个线程有成功的结果返回，其他的结果直接进入了降级。

> 默认 coreSize 为 10

2、当我的配置中加入了 groupKey 和 threadPoolKey 的配置后，当前的线程组和线程池的关系如下所示：

- 当多个接口中这两个参数配置一致时：
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200228160115771.png)
	会出现多个接口公用一个线程组中的线程池！此时的总默认线程数为 10
- 当多个接口中 groupKey 一致，但 threadPoolKey 名称不同时：
	![在这里插入图片描述](https://img-blog.csdnimg.cn/2020022816034173.png)
	此时的总默认线程数为 20

3、当我们在接口中配置了 threadPoolProperties 参数信息，同时 coreSize(最大线程核心数) 为 15 时，如果此时同一时间有20个请求，他的返回结果为：

> 先会返回 5 个降级的信息，后返回15个完整正常的信息。
> (顺序是因为我的代码中加了延迟)
> (有可能不止15个成功，还要看是否配置了其他参数)

4、maxQueueSize 和 queueSizeRejectionThreshold 的设置问题：

| 名称                        | 含义                                       |
| --------------------------- | ------------------------------------------ |
| maxQueueSize                | 表示线程池队列的最大大小                   |
| queueSizeRejectionThreshold | 表示限定当前队列大小(线程池中有效的队列数) |

注意：这两个设置的参数信息，Hystrix取的是两者中最小的那个！

5、重点总结：
[问1：]有人曾问过我，我设置了 coreSize 为 5 ，也设置了 maxQueueSize 和 queueSizeRejectionThreshold ，但这两个最小的值为 2 ，我压测时，为什么会出现 7个正常，其他失败？然道不是 5 个正常，其他失败么？

[答1：]

> 1、在请求来的时候，Hystrix会根据 coreSize 给足 能够接受的请求数！
> 2、当 coreSize 请求数使用完后，会根据 maxQueueSize 和 queueSizeRejectionThreshold 两者中的最小值(假设n)，扣留指定数目(n)的请求。
> 3、当 coreSize 数的请求处理完成后，再请求 剩下的扣留的n的数目的请求。
> 4、其他的没有允许的全部进入了降级逻辑中。
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200228162938765.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MzIyNTI3,size_16,color_FFFFFF,t_70)

## 七、本节博客源码地址

《[使用Feign+Hystrix实现服务的请求和保护demo](https://github.com/765199214/springcloud2.0-detailConfig/tree/master/10 feign%2Bhystrix实现服务的熔断保护机制)》
《[使用RestTemplate+Hystrix的降级、超时配置demo](https://github.com/765199214/springcloud2.0-detailConfig/tree/master/11 resttemplate%2Bhystrix实现服务的熔断保护)》
《[使用RestTemplate+Hystrix实现服务的熔断保护配置demo](https://github.com/765199214/springcloud2.0-detailConfig/tree/master/12 resttemplate%2Bhystrix实现服务的熔断保护)》

《[使用RestTemplate+Hystrix实现服务限流保护配置demo](https://github.com/765199214/springcloud2.0-detailConfig/tree/master/13 resttemplate%2Bhystrix实现服务的限流保护)》

《[Hystrix的官方配置文档](https://github.com/Netflix/Hystrix/wiki/Configuration)》