# Spring Boot 对多线程支持-提高程序执行效率

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[jieniyimiao](https://me.csdn.net/u013467442) 2019-04-17 22:05:00 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 25195 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 71



分类专栏： [Spring](https://blog.csdn.net/u013467442/category_8846319.html) [spring核心技术](https://blog.csdn.net/u013467442/category_9287294.html) 文章标签： [@EnableAsync](https://so.csdn.net/so/search/s.do?q=@EnableAsync&t=blog&o=vip&s=&l=&f=&viparticle=)[Async](https://www.csdn.net/gather_27/MtTaEg0sNDIwMjktYmxvZwO0O0OO0O0O.html)[Spring Boot 对多线程支持](https://so.csdn.net/so/search/s.do?q=Spring Boot 对多线程支持&t=blog&o=vip&s=&l=&f=&viparticle=)[线程池](https://www.csdn.net/gather_22/MtTaEg0sMTEwMjYtYmxvZwO0O0OO0O0O.html)[CompletableFuture](https://so.csdn.net/so/search/s.do?q=CompletableFuture&t=blog&o=vip&s=&l=&f=&viparticle=)

版权

## 1.楔子

在我们的系统中，经常会处理一些耗时任务，自然而然的会想到使用多线程，JDK给我们提供了非常方便的操作线程的API,为什么还要使用Spring来实现多线程呢？

> 1.使用Spring比使用JDK原生的并发API更简单。（一个注解`@Async`就搞定）
> 2.我们的应用环境一般都会集成Spring，我们的Bean也都交给Spring来进行管理，那么使用Spring来实现多线程更加简单，更加优雅。

为什么要用异步？当需要调用多个服务时，使用传统的同步调用来执行时，是这样的

> 调用服务A
> 等待服务A的响应
> 调用服务B
> 等待服务B的响应
> 调用服务C
> 等待服务C的响应
> 根据从服务A、服务B和服务C返回的数据完成业务逻辑，然后结束

如果每个服务需要3秒的响应时间，这样顺序执行下来，可能需要9秒以上才能完成业务逻辑，但是如果我们使用异步调用

> 调用服务A
> 调用服务B
> 调用服务C
> 然后等待从服务A、B和C的响应
> 根据从服务A、服务B和服务C返回的数据完成业务逻辑，然后结束

理论上 3秒左右即可完成同样的业务逻辑

## 2. spring boot 如何使用多线程

Spring中实现多线程，其实非常简单，只需要在配置类中添加`@EnableAsync`就可以使用多线程。在希望执行的并发方法中使用`@Async`就可以定义一个线程任务。通过spring给我们提供的`ThreadPoolTaskExecutor`就可以使用线程池。

#### 2.1第一步，先在Spring Boot主类中定义一个线程池，比如：

```java
@Configuration
@EnableAsync  // 启用异步任务
public class AsyncConfiguration {

    // 声明一个线程池(并指定线程池的名字)
    @Bean("taskExecutor")
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //核心线程数5：线程池创建时候初始化的线程数
        executor.setCorePoolSize(5);
        //最大线程数5：线程池最大的线程数，只有在缓冲队列满了之后才会申请超过核心线程数的线程
        executor.setMaxPoolSize(5);
        //缓冲队列500：用来缓冲执行任务的队列
        executor.setQueueCapacity(500);
        //允许线程的空闲时间60秒：当超过了核心线程出之外的线程在空闲时间到达之后会被销毁
        executor.setKeepAliveSeconds(60);
        //线程池名的前缀：设置好了之后可以方便我们定位处理任务所在的线程池
        executor.setThreadNamePrefix("DailyAsync-");
        executor.initialize();
        return executor;
    }
}
12345678910111213141516171819202122
```

有很多你可以配置的东西。默认情况下，使用`SimpleAsyncTaskExecutor`。

#### 2.2第二步，使用线程池

在定义了线程池之后，我们如何让异步调用的执行任务使用这个线程池中的资源来运行呢？方法非常简单，我们只需要在`@Async`注解中指定线程池名即可，比如：

```java
@Service
public class GitHubLookupService {

    private static final Logger logger = LoggerFactory.getLogger(GitHubLookupService.class);

    @Autowired
    private RestTemplate restTemplate;

    // 这里进行标注为异步任务，在执行此方法的时候，会单独开启线程来执行(并指定线程池的名字)
    @Async("taskExecutor")
    public CompletableFuture<String> findUser(String user) throws InterruptedException {
        logger.info("Looking up " + user);
        String url = String.format("https://api.github.com/users/%s", user);
        String results = restTemplate.getForObject(url, String.class);
        // Artificial delay of 3s for demonstration purposes
        Thread.sleep(3000L);
        return CompletableFuture.completedFuture(results);
    }
}
12345678910111213141516171819
```

findUser 方法被标记为Spring的 @Async 注解，表示它将在一个单独的线程上运行。该方法的返回类型是 CompleetableFuture 而不是 String，这是任何异步服务的要求。

#### 2.3第三步，单元测试

最后，我们来写个单元测试来验证一下

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
public class AsyncTests {
    private static final Logger logger = LoggerFactory.getLogger(AsyncTests.class);

    @Autowired
    private GitHubLookupService gitHubLookupService;

    @Test
    public void asyncTest() throws InterruptedException, ExecutionException {
        // Start the clock
        long start = System.currentTimeMillis();

        // Kick of multiple, asynchronous lookups
        CompletableFuture<String> page1 = gitHubLookupService.findUser("PivotalSoftware");
        CompletableFuture<String> page2 = gitHubLookupService.findUser("CloudFoundry");
        CompletableFuture<String> page3 = gitHubLookupService.findUser("Spring-Projects");

        // Wait until they are all done
        //join() 的作用：让“主线程”等待“子线程”结束之后才能继续运行
        CompletableFuture.allOf(page1,page2,page3).join();

        // Print results, including elapsed time
        float exc = (float)(System.currentTimeMillis() - start)/1000;
        logger.info("Elapsed time: " + exc + " seconds");
        logger.info("--> " + page1.get());
        logger.info("--> " + page2.get());
        logger.info("--> " + page3.get());
    }

}

1234567891011121314151617181920212223242526272829303132
```

执行上面的单元测试，我们可以在控制台中看到所有输出的线程名前都是之前我们定义的线程池前缀名开始的，并且执行时间小于9秒，说明我们使用线程池来执行异步任务的试验成功了！
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190417215220732.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9qaWVuaXlpbWlhby5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

## 3. 注意事项

在使用spring的异步多线程时经常回碰到多线程失效的问题，解决方式为：
异步方法和调用方法一定要`写在不同的类中` ,如果写在一个类中,是没有效果的！
原因：

> spring对@Transactional注解时也有类似问题，spring扫描时具有@Transactional注解方法的类时，是生成一个代理类，由代理类去开启关闭事务，而在同一个类中，方法调用是在类体内执行的，spring无法截获这个方法调用。