# rabbitmq监听队列中的消息数量

https://blog.csdn.net/java_chegnxuyuan/article/details/105228711



分类专栏： [Java](https://blog.csdn.net/java_chegnxuyuan/category_9352332.html)

版权

今天有个需求就是根据队列中消息的数量来执行不同的代码。

代码如下：

获取MQ连接

```java
private Channel  getMqConnection(){

        ConnectionFactory factory = new ConnectionFactory();

        //设置MabbitMQ所在主机ip或者主机名
        factory.setHost(host);

        factory.setPort(port);

        factory.setUsername(userName);

        factory.setPassword(password);

        Connection connection = null;

        Channel channel = null;

        try {

            connection = factory.newConnection();
            channel = connection.createChannel();
        } catch (IOException e) {

            e.printStackTrace();
        } catch (TimeoutException e) {

            e.printStackTrace();
        }
        return channel;
    }
Channel channel = getMqConnection();

        AMQP.Queue.DeclareOk declareOk = channel.queueDeclarePassive(queue);
        int num = declareOk.getMessageCount();
```

然后就可以根据num的值来判断执行