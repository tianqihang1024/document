# jedis和lettuce连接redis方案性能对比

![img](https://csdnimg.cn/release/phoenix/template/new_img/original.png)

[我行其野&芃芃其麦](https://me.csdn.net/honger_hua) 2020-06-03 17:52:23 ![img](https://csdnimg.cn/release/phoenix/template/new_img/articleReadEyes.png) 737 ![img](https://csdnimg.cn/release/phoenix/template/new_img/tobarCollect.png) 收藏 2



分类专栏： [高并发](https://blog.csdn.net/honger_hua/category_9640891.html)

版权



​     针对redis连接池获取连接出现长时间等待问题虽然现在通过调整线程池大小和去掉调用是否联通号码逻辑暂时起到了一些作用，但是

 却是治标不治本的方案。因此秉着事情必须要有闭环的宗旨。今天花了一天时间做了如下的压力测测试。

​      知识储备：

​      jedis客户端连接方式是基于tcp的阻塞式连接方式。

​      lettuce客户端连接方式是基于netty的多路复用异步非阻塞的连接方案。（目前业界解决高并发大数据的问题的思路）

​    

​      在场景一：

​      最大线程数：10  最大空闲线程10  最小空闲线程5  并发数 100/s   时间  120s

 

​      jedis客户端连接

​      ![img](https://img-blog.csdnimg.cn/20200603175109544.png)

​     lettuce客户端连接

​    ![img](https://img-blog.csdnimg.cn/20200603175117400.png)

 

 

​     在场景二：

​      最大线程数：10  最大空闲线程10  最小空闲线程5  并发数 200/s   时间  120s

 

​      jedis客户端连接

​      ![img](https://img-blog.csdnimg.cn/20200603175128276.png)

​     lettuce客户端连接

​     ![img](https://img-blog.csdnimg.cn/20200603175136157.png)

 

 

​         在场景三：

​      最大线程数：50  最大空闲线程50  最小空闲线程50  并发数 200/s   时间  120s

 

​      jedis客户端连接

​      ![img](https://img-blog.csdnimg.cn/20200603175144153.png)

 

​       lettuce客户端连接

​     ![img](https://img-blog.csdnimg.cn/20200603175152320.png)

 

 

​        在场景四：

​      最大线程数：70  最大空闲线程50  最小空闲线程20  并发数 200/s   时间  120s

 

​      jedis客户端连接

​      ![img](https://img-blog.csdnimg.cn/20200603175200660.png)     

 

 

​        lettuce客户端连接

​       ![img](https://img-blog.csdnimg.cn/20200603175208169.png)

 

​       

​       结论1：

​       所有场景：

​       jedis表现的吞吐量优于lettuce连接方案。

​       lettuce表现的响应时间和稳定性优于jedis连接方案。

​       

​       结论2：

​       场景一  vs  场景二

​       通过增大并发量

​       jedis吞吐量增大，超时错误开始出现，最大响应时间增大。

​       原因分析：连接数达到最大值，出现连接超时拒绝，连接等待出现  。所以最大耗时增加。 

​       lettuce表现都是毫秒级别，但是平均耗时和最大耗时开始增加。

​       原因分析：lettuce采用多路复用原理，因此真正工作的连接受制于CPU核数因此增大连接数反而增加了线程上下文切换时间。因此建议调整为  CPU核实+1.

 

​       场景二  vs 场景三、四

​       通过调大线程池数量

​       jedis吞吐量增大，超时错误开始出现，最大响应时间增大。

​       原因分析：连接数达到最大值，出现连接超时拒绝，连接等待出现  。所以最大耗时增加。 

​       lettuce表现都是毫秒级别，但是平均耗时和最大耗时开始增加。

​       原因分析：lettuce采用多路复用原理，因此真正工作的连接受制于CPU核数因此增大连接数反而增加了线程上下文切换时间。因此建议调整为  CPU核数+1.

 

 

​       最终结论：

​       调大连接池大小能够提高jedis的吞吐量，但是不能避免出现超时错误和长时间等待。

​       jedis连接方式最大连接数和最小、最大空闲连接数设置为一样有利于减少上下文切换时间，提升效率。

​       letturce调大连接池大小反而会影响性能，最佳个数= CPU核数+1.

​       letturce整体稳定性和性能由于jedis方式。