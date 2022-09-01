# `CompletableFuture`是什么

`CompletableFuture`是`jdk1.8`引入的实现类。扩展了`Future`和`CompletionStage`，是一个可以在任务完成阶段触发一些操作`Future`。简单的来讲就是可以实现异步回调。

## 













### `CompletableFuture`自定义线程池，每个请求都自定义还是所有请求共用一个

请求自定义的话，相当于每个异步请求都创建一个线程，用完还得关闭不如直接`new Thread`。还是共用一个线程池比较好，把拒绝策略设置为`CallerRunsPolicy`，任务无法使用线程池，就自己去执行异步任务。





### join和get的区别

-   `join`会抛出未经检查的异常（不会显示要求处理异常，所以异常未知）。
-   `get`会抛出经检查的异常，可被捕获，自定义处理或者直接抛出（执行异常、中断异常、超时异常）。





自定义线程池，避免因为使用共享线程池，导致任务不能即使消费。

```java
package com.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;

/**
 * @author 田奇杭
 * @Description CompletableFuture自定义线程池配置类
 * @Date 2022/5/10 12:59
 */
@Slf4j
@Configuration
public class CompletableFuturePoolConfig {

    @Bean("storeFlowPool")
    public Executor storeFlowPool() {
        ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
        // 核心线程数
        threadPoolTaskExecutor.setCorePoolSize(Runtime.getRuntime().availableProcessors());
        // 允许核心线程关闭
        threadPoolTaskExecutor.setAllowCoreThreadTimeOut(true);
        // 最大线程数
        threadPoolTaskExecutor.setMaxPoolSize(Runtime.getRuntime().availableProcessors() * 2);
        // 队列大小
        threadPoolTaskExecutor.setQueueCapacity(1000);
        // 线程池中线程的名称前缀
        threadPoolTaskExecutor.setThreadNamePrefix("store_flow：");
        // HelloWorldServiceImpl     rejection-policy: 当pool已经达到max size时，如何处理新任务：
        // CallerRunsPolicy: 不在新线程中执行任务，而是由调用者所在的线程来执行；
        // AbortPolicy: 拒绝执行新任务，并抛出RejectedExecutionException异常；
        // DiscardPolicy：丢弃当前将要加入队列的任务；
        // DiscardOldestPolicy：丢弃任务队列中最旧的任务；
        threadPoolTaskExecutor.setRejectedExecutionHandler(
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
        threadPoolTaskExecutor.initialize();
        return threadPoolTaskExecutor;
    }
}

```

```java
package com.test.future;

import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Executor;

/**
 * @author 田奇杭
 * @Description
 * @Date 2022/5/9 22:44
 */
@Slf4j
@Component
public class CompletableFutureTest {

    /**
     * 引入自定义线程池
     */
    @Resource(name = "storeFlowPool")
    private Executor executor;

    /**
     * 测试代码
     */
    @Scheduled(cron = "0/2 * * * * ?")
    public void get() throws ExecutionException, InterruptedException {

        // TODO CompletableFuture 不指定线程池的话，使用默认的ForkJoinPool.commonPool线程池，
        //  它是当前JVM里所有平行流共享，可能会导致异步任务不能正常执行
        // 异步起一个线程，无返回值
        CompletableFuture.runAsync(this::compute, executor);
        // 异步起一个线程，有返回值
        CompletableFuture<Integer> supplyAsync = CompletableFuture.supplyAsync(this::compute, executor);

        log.info("supplyAsync 计算结果为：{}", supplyAsync.join());
        log.info("supplyAsync 计算结果为：{}", supplyAsync.get());
        log.info("supplyAsync 计算结果为：{}", supplyAsync.get(3, TimeUnit.SECONDS));
    }

    public Integer compute() {
        log.info("当前线程为：{}", Thread.currentThread().getName());
        return 666;
    }
}
```





