



## `ForkJoin`关键词

-   `ForkJoinPool`：`ForkJoin`专用线程池
-   `ForkJoinTask`：`ForkJoinPool`指定的任务类型，有两个子类`RecursiveTask、RecursiveAction`，如果直接提交 `Callable` 或 `Runnable` 也会被自动包装成 `ForkJoinTask`；
-   `WorkQueue`：`ForkJoin`专用工作队列，支持LIFO、FIFO两种类型。
-   `ForkJoinWorkerThread`：`ForkJoin`专用工作线程

**运行关系如下**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201109211849632.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zODM4MDg1OA==,size_16,color_FFFFFF,t_70#pic_center)



## 工作线程状态

在`ForkJoinzPool`中，工作线程有几种状态

-   `running`：正在执行任务，并且没有被阻塞，`getRunningThreadCount`函数获取正在运行的线程数量
-   `active`：活跃状态，`getActiveThreadCount`函数可以获得，细分为两种
    -   `busy`，正在扫描自己的任务队列
    -   `scan`，正在扫描其他队列的任务
-   `inactive`：自己的任务已经处理完毕，并且没有发现其他待执行的任务，最终线程挂起



## 运算符

-   `&`：相同为`1`，不同为`0`
-   `|`：有`1`则`1`，无`1`为`0`
-   `^`：不同为`1`，相同为`0`
-   `~`：
-   `<<`：左移指定的`bit`位，低位补`0`
-   `>>`：右移指定的`bit`位，高位补`0`



## 相关变量

-   `config`：
-   `runs`：
-   `config`：
-   `config`：







如果一个现成的随机数可以改变，怎么确保线程再一次进来能够处理自己刚刚提交的任务





















## `ForkJoinPool`线程池状态

1.   











































