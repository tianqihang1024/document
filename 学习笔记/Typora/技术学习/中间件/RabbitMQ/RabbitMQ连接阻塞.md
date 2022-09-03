# 记一次RabbitMQ连接阻塞，全部队列不消费异常

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[林老师带你学编程](https://me.csdn.net/linzhiqiang0316) 2019-06-02 14:13:40 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 7442 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 5



分类专栏： [MQ](https://blog.csdn.net/linzhiqiang0316/category_7741585.html)

版权

![image](https://www.wolzq.com/20190602/2001.jpg?ynotemdtimestamp=1559455966247)

前几天博主遇到一个很狗屎的bug，RabbitMQ本来运行的好好突然所有的消息队列都不消费了，看了一下 Connections连接，发现全部都发生阻塞了，导致线上的队列堆积如山，情况万分危急。

### 推测一：生产者和消费者问题

刚开始推测是不是生产者和消费者出问题了，然后就检查了一下服务的运行状态，发现都没问题，说明不是这个问题引起的，所以进一步猜测可能是MQ本身出现问题了。

### 推测二：MQ本身出现问题

如果是MQ出现问题，那MQ的日志肯定会有错误的相关信息记载，所以我们进入MQ日志下面，查看日志情况。

```
/var /log/rabbitmq/
```

**日志结果如下所示：**

```java
=ERROR REPORT==== 30-May-2019::19:28:49 ===

connection <0.23331.7132>, channel 4 - soft error:

{amqp_error,precondition_failed,"unknown delivery tag 1",'basic.ack'}

=INFO REPORT==== 30-May-2019::19:28:49 ===

vm_memory_high_watermark clear. Memory used:383597048 allowed:385616281

=WARNING REPORT==== 30-May-2019::19:28:49 ===

memory resource limit alarm cleared on node rabbit@VM_2_12_centos
```

通过错误日志我们可以很明显的看出，是MQ的运行内存超过限制，导致连接阻塞的。

### 问题原因：

因为RabbitMQ服务器在启动时会计算系统内存总大小。然后会根据vm_memory_high_watermark参数指定的百分比，进行控制.可通过命令 rabbitmqctl set_vm_memory_high_watermark fraction 动态设置。默认下，vm_memory_high_watermark的值为0.4,当RabbitMQ使用的内存超过40%时，系统会阻塞所有连接。一旦警报解除（消息被消费者取走，或者消息被写到硬盘中）,系统重新恢复工作。32位系统的单进程内存使用最大为2G,即使系统有10G,那么内存警报线为2G*40%.

### 解决方案：

最快的解决方案时将**vm_memory_high_watermark**值上调，如果短时间内上调不了，可以选择直接升级服务器内存。

但是这样治标不治本，如果队列继续堆积，内存占用率还是会变大，最终的解决方案，应该要**平衡消费者和生产者**，让消息队列尽量不要堆积导致内存过大阻塞。

### 其它阻塞的场景：

**硬盘控制**

当RabbitMQ的磁盘空闲空间小于50M（默认），生产者将被BLOCK，并且阻塞信息发布前，会尝试把内存中的信息输出到磁盘上。持久化信息和非持久化信息都将被写到磁盘（持 久化信息一进入队列就会被写到磁盘）。如果采用集群模式，磁盘节点空闲空间小于50M将导致其他节点的生产者都被block。可以通过disk_free_limit来对进行配置。 如果磁盘的预设值为50%，内存预设值默认是0.4，那么就是当内存使用量达到20%时，队列信息会被写到磁盘上。 可以通过设置 vm_memory_high_watermark_paging_ratio （默认值为0.5）来改变写入磁盘的策略，例如：

```java
[{rabbit, [{vm_memory_high_watermark_paging_ratio, 0.75},          {vm_memory_high_watermark, 0.4}]}].1
```

上面的配置将会在内存全用30%时将队列信息写入到磁盘，阻塞信息发布是在内存使用40%时发生。建议vm_memory_high_watermark 不超过50%.

**消息积压**

在RabbitMQ中，消息可能被存储在多个不同的队列，消息越早被消费，那么消息经过的队列层次越少，则平均每个消息处理的开销就越小。但若发布消息的速率过快，MQ来不及处理，这些消息就可能进入很深层次的队列，大大增加平均每个消息的处理开销，进一步使得处理新消息和发送旧消息的能力减弱，更多的消息会进入很深的队列，循环往复，整个系统的性能就会极大的降低。另外若接收消息的速率过快还会实现某些进程的mailbox过大，可能会产生很严重的后果。为此，RabbitMQ设计了一套流控机制。

RabbitMQ 使用了一种基于 credit 的算法来 限制 message 被 publish 的速率 。Publisher 只有在其从某个 queue 的 全部镜像处收到 credit 之后才被允许继续 publish 。在这个上下文中，Credit 意味着对 publish 行为的允许。如果存在没能成功发送出 credit 的 slaves ，则将导致 publisher 停止 publish 动作。Publisher 会一直保持停止的状态，直到所有 slave 都成功发送了 credit 或者直到剩余的 node 都认为某 slave 已经从 cluster 中断开了。Erlang 会周期性地发送 tick 到所有的 node 上来检测是否出现连接断开。 tick 的间隔时间可以通过配置 net_ticktime 的值来控制。