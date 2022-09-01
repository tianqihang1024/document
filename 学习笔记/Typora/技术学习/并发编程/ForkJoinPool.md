

# `ForkJoinPool`是什么

`ForkJoinPool`是`JDK7`引入的线程池，核心思想是将大的任务拆分成多个小任务（即`fork`），然后在将多个小任务处理汇总到一个结果（即`join`）。提供基本的线程池功能，支持设置最大并发线程数，支持任务排队，支持线程池停止，支持线程池使用情况监控，也是`AbstractExecutorService`的子类，并且引入了“工作窃取”机制。

![图片](C:/Users/22489/OneDrive/%E7%94%B0%E5%A5%87%E6%9D%AD/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/TyporaImg/640.png)





# `ForkJoinPool`怎么使用

1.   `ForkJoinTask`有`RecursiveTask、RecursiveAction（rɪˈkɜːsɪv：递归）`两个子类，前者有返回值，后者没有。
2.   创建一个执行类，继承`ForkJoinTask`上述任意子类，重写`compute（kəmˈpjuːt：计算）`方法。
3.   创建一个`ForkJoinPool`对象，使用`summit`方法入参为上面创建的`ForkJoinTask`对象。接收返回值，调用返回值的`get`方法，阻塞的形式获取结果值。



# `ForkJoinPool`注意事项

1.   `ForkJoinPool`是CPU密集型，避免在子任务进行`IO`操作。

2.   为什么推荐使用`invoke()`而非`fork()`，对于`fork/join`模式，假如pool里面线程数量是固定的，那么调用子任务的fork方法相当于A先分工给B，然后A当监工不干活，B去完成A交代的任务。所以上面的模式相当于浪费了一个线程。那么如果使用`invokeAll()`相当于A分工给B后，A和B都去完成工作。这样缩短了执行的时间。

     -   `invokeAll()`执行`512`个任务耗时`28917ms、24794ms`

     -   `fork()`执行`512`个任务耗时`29267ms、25939ms`




# `ForkJinPool`原理

基本思想

-   `ForkJoinPool`中，每个工作线程，都维护着自己的双端工作队列，里面存放待处理的任务。
-   每个工作线程在运行中，会产生新的任务（通常是因为调用了 `fork()`）时，会放入工作队列的队尾，并且工作线程在处理自己的工作队列时，使用的是 *LIFO* 方式，也就是说每次从队尾取出任务来执行。
-   每个工作线程在处理自己的任务时，会尝试**窃取**一个任务（或是来自于刚刚提交到 pool 的任务，或是来自于其他工作线程的工作队列），窃取的任务位于其他线程的工作队列的队首，也就是说工作线程在窃取其他工作线程的任务时，使用的是 *FIFO* 方式。
-   在遇到 `join()` 时，如果需要 join 的任务尚未完成，则会先处理其他任务，并等待其完成。
-   在既没有自己的任务，也没有可以窃取的任务时，进入休眠。



# `ForkJoinPool`示例

1.   根据业务需求创建`ForkJoinPool`对象交给`IOC`容器方便统一管理。
2.   ==抽空补全==



```java
package com.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.ForkJoinPool;

/**
 * @author 田奇杭
 * @Description 店铺客流线程池
 * @Date 2022/4/27 22:17
 */
@Configuration
public class StoreFlowForkJoinPoolConfig {

    /**
     * 自定义店铺客流Fork Join Pool线程池
     *
     * @return ForkJoinPool
     */
    @Bean(name = "storeFlowForkJoinPool")
    public ForkJoinPool storeFlowForkJoinPool() {
        // 获取当前服务器CPU核数：Runtime.getRuntime().availableProcessors()
        // 其余参数使用默认值（上面那个就是不指定时的默认值，我也懒的想具体的核心数应该是多少，自己的项目还这么懒 ╮（╯＿╰）╭）
        return new ForkJoinPool(Runtime.getRuntime().availableProcessors());
    }
}

```

```java
package com.bean;

import com.fasterxml.jackson.annotation.JsonFormat;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.format.annotation.DateTimeFormat;

import java.io.Serializable;
import java.time.LocalDateTime;

/**
 * @author 田奇杭
 * @Description 店铺客流实体
 * @Date 2022/4/27 22:17
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class StoreFlow implements Serializable {

    private static final long serialVersionUID = -5449125967256666922L;

    /**
     * 主键
     */
    private Long id;

    /**
     * 租户id
     */
    private Long tenantId;

    /**
     * 店铺id
     */
    private Long storeId;

    /**
     * 总客流
     */
    private Long sumFlow;

    /**
     * 新客流
     */
    private Long newFlow;

    /**
     * 老客流
     */
    private Long oldFlow;

    /**
     * 创建时间
     */
    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime created;


    public StoreFlow(Long tenantId, Long storeId, Long sumFlow, Long oldFlow, Long newFlow, LocalDateTime created) {
        this.tenantId = tenantId;
        this.storeId = storeId;
        this.sumFlow = sumFlow;
        this.oldFlow = oldFlow;
        this.newFlow = newFlow;
        this.created = created;
    }
}

```

```java
package com.tasks;

import com.mapper.StoreFlowMapper;
import com.tasks.config.StoreFlowForkJoin;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.ForkJoinTask;

/**
 * @author 田奇杭
 * @Description 插入店铺客流定时器
 * @Date 2022/4/9 18:10
 */
@Slf4j
@Component
public class StoreFlowTask {

    /**
     * 店铺客流ForkJoin执行类
     */
    @Resource
    private StoreFlowMapper storeFlowMapper;

    /**
     * 店铺客流ForkJoinPool
     */
    @Resource(name = "storeFlowForkJoinPool")
    private ForkJoinPool storeFlowForkJoinPool;

    @Scheduled(cron = "10 14 14 * * ?")
    public void storeFlowInsert() throws ExecutionException, InterruptedException {

        long startNum = System.currentTimeMillis();

        log.info("开始执行插入客流数据定时任务");

        // 执行时间范围
        int start = 0;
        int end = 512;

        // 提交任务给线程池
        ForkJoinTask<Integer> task = this.storeFlowForkJoinPool.submit(new StoreFlowForkJoin(storeFlowMapper, start, end));
        // 阻塞获取结果值
        Integer count = task.get();
        log.info("共插入数据：{} 条", count);

        long endNum = System.currentTimeMillis();
        log.info("店铺客流定时任务耗时：{}", endNum-startNum);
    }
}

```

```java
package com.tasks.config;

import com.bean.StoreFlow;
import com.mapper.StoreFlowMapper;
import lombok.extern.slf4j.Slf4j;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ThreadLocalRandom;

/**
 * @author 田奇杭
 * @Description
 * @Date 2022/4/9 19:15
 */
@Slf4j
public class StoreFlowForkJoin extends RecursiveTask<Integer> {

    /**
     * 最小分割
     */
    private static final Integer MAX_TASK = 1;

    /**
     * 租户Id
     */
    private static final Long TENANT_ID = 1L;

    /**
     * 店铺列表
     */
    private static final List<Long> STORE_LIST = new ArrayList<>();

    /**
     * 初始化店铺id
     */
    static {
        STORE_LIST.add(131231L);
        STORE_LIST.add(231231L);
        STORE_LIST.add(331231L);
        STORE_LIST.add(431231L);
        STORE_LIST.add(531231L);
        STORE_LIST.add(631231L);
        STORE_LIST.add(731231L);
        STORE_LIST.add(831231L);
        STORE_LIST.add(931231L);
    }

    /**
     * 随机函数
     */
    private final ThreadLocalRandom random = ThreadLocalRandom.current();

    /**
     * 店铺客流表持久层
     */
    private final StoreFlowMapper storeFlowMapper;

    /**
     * 开始时间（天）
     */
    private final Integer startDay;

    /**
     * 结束时间（天）
     */
    private final Integer endDay;

    /**
     * @param storeFlowMapper 持久层
     * @param startDay        开始时间
     * @param endDay          结束时间
     */
    public StoreFlowForkJoin(StoreFlowMapper storeFlowMapper, Integer startDay, Integer endDay) {
        this.storeFlowMapper = storeFlowMapper;
        this.startDay = startDay;
        this.endDay = endDay;
    }

    /**
     * 切割任务
     *
     * @return 执行任务数量
     */
    @Override
    protected Integer compute() {

        // 结束时间 - 开始时间大于阈值，继续分割任务
        if ((endDay - startDay) <= MAX_TASK) {

            int insertSum = 0;
            // 循环待插入店铺列表
            for (Long store : STORE_LIST) {

                // 组装数据
                StoreFlow storeFlow = this.assembleStoreFlowData(store);
                // 入库
                insertSum += storeFlowMapper.insert(storeFlow);
                log.info("insertSum:{}", insertSum);
            }

            return insertSum;
        } else {
            // 任务过大，是用二分法分割
            int middle = (startDay + endDay) / 2;

            // 创建两个forkJoinTask对象
            StoreFlowForkJoin firstTask = new StoreFlowForkJoin(storeFlowMapper, startDay, middle);
            StoreFlowForkJoin secondTask = new StoreFlowForkJoin(storeFlowMapper, middle, endDay);

            // 激活firstTask、secondTask
            invokeAll(firstTask, secondTask);
            // 获取并计算激活firstTask、secondTask的加值
            return firstTask.join() + secondTask.join();
        }
    }

    /**
     * 组装店铺客流数据
     *
     * @return
     */
    private StoreFlow assembleStoreFlowData(Long storeId) {

        // 使用随机函数生成客流
        long oldFlow = random.nextInt(5000);
        long newFlow = random.nextInt(2000);
        long sumFlow = oldFlow + newFlow;

        // 获取插入时间
        LocalDateTime now = LocalDateTime.now();
        LocalDateTime created = now.minusDays(startDay);

        return new StoreFlow(TENANT_ID, storeId, sumFlow, oldFlow, newFlow, created);
    }
}
```













