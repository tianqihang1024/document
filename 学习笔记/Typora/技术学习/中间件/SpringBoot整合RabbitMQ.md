# `SpringBoot`整合`RabbitMQ`

**以下内容为刚需，额外内容自行探索**



## 生产者

依赖使用的都是`SpringBoot`整合版，不需要指定`version`。

```xml
<!-- springboot整合amqp -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

配置项

```properties
# MQ
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
# 开启 confirm 函数回调
spring.rabbitmq.publisher-confirms=true
# 开启 CorrelationData 对象传输
spring.rabbitmq.publisher-confirm-type=correlated
# 开启 returnedMessage 函数回调，发送失败退回（消息有没有找到合适的队列）
spring.rabbitmq.publisher-returns=true
```



```java
package com.config;

import com.alibaba.fastjson.JSON;
import com.baomidou.mybatisplus.core.toolkit.IdWorker;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessageProperties;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.nio.charset.StandardCharsets;

/**
 * @author 田奇杭
 * @Description 消息发送类
 * @Date 2022/5/9 16:01
 */
@Slf4j
@Component("rabbitSender")
public class RabbitSender implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 将发送消息封装成Message
     *
     * @param message 带封装对象
     * @return message
     */
    public static Message convertMessage(Object message) {
        MessageProperties mp = new MessageProperties();
        byte[] src = JSON.toJSONString(message).getBytes(StandardCharsets.UTF_8);
        mp.setContentType("application/json");
        mp.setContentEncoding("UTF-8");
        return new Message(src, mp);
    }

    @PostConstruct
    public void init() {
        // 赋值自定义 confirm
        this.rabbitTemplate.setConfirmCallback(this);
        // 赋值自定义 returnedMessage
        this.rabbitTemplate.setReturnCallback(this);
        // 数据转换为json存入消息队列
        this.rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
    }

    /**
     * 发送消息到指定队列
     *
     * @param exchange   交换机
     * @param routingKey 路由key
     * @param msg        消息体
     */
    public void directExchangeSender(String exchange, String routingKey, Object msg) {
        Message message = convertMessage(msg);
        log.info("即将投递的消息体为：{}", JSON.toJSONString(msg));
        CorrelationData correlationData = new CorrelationData(IdWorker.get32UUID());
        rabbitTemplate.convertAndSend(exchange, routingKey, message, correlationData);
    }

    /**
     * RabbitTemplate.ConfirmCallback接口用于实现消息发送到RabbitMQ交换器exchange的回调
     *
     * @param correlationData
     * @param ack             消息是否被投递到RabbitMQ交换器exchange
     * @param cause           失败原因
     */
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        String correlationDataId = correlationData.getId();
        if (ack) {
            log.info("消息correlationDataId:{}投递到exchange成功", correlationDataId);
        } else {
            log.error("消息correlationDataId:{}投递到exchange失败cause:{}", correlationDataId, cause);
        }
    }

    /**
     * RabbitTemplate.ReturnCallback接口用于实现消息发送到RabbitMQ交换器，但无相应队列与交换器绑定时的回调
     * 只有消息从Exchange路由到Queue失败才会回调这个方法
     *
     * @param message
     * @param replyCode
     * @param replyText
     * @param exchange
     * @param routingKey
     */
    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.error("消息路由到队列失败message:{} replyCode:{} replayText:{}exchange:{} routingKey={}", message, replyCode, replyText, exchange, routingKey);
    }

}

```



```java
package com.tasks;

import com.bean.Course;
import com.config.RabbitSender;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.concurrent.ThreadLocalRandom;

/**
 * @author 田奇杭
 * @Description 店铺客流MQ定时器（用于测试）
 * @Date 2022/5/8 23:32
 */
@Component
public class StoreFlowMessageSendTask {

    /**
     * 交换机
     */
    private static final String STORE_FLOW_EXCHANGE_NAME = "store_flow_topic_exchange";

    /**
     * 路由KEY
     */
    private static final String STORE_FLOW_ROUTING_KEY_NAME = "store_flow_topic_routing_key";

    /**
     * 随机函数
     */
    private static final ThreadLocalRandom RANDOM = ThreadLocalRandom.current();

    /**
     * 消息发送类
     */
    @Resource(name = "rabbitSender")
    private RabbitSender rabbitSender;

    /**
     * 测试 幂等操作，手动应答
     * TODO 启动类添加 @EnableScheduling
     */
    @Scheduled(cron = "0/10 * * * * ?")
    public void testHttpClient() {
        // 消息内容
        Course message = new Course(RANDOM.nextLong(9999999), "", 1L, "");
        rabbitSender.directExchangeSender(STORE_FLOW_EXCHANGE_NAME, STORE_FLOW_ROUTING_KEY_NAME, message);
    }
}

```





## 消费者

依赖使用的都是`SpringBoot`整合版，不需要指定`version`。

```xml
<dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
</dependencies>
```

配置项，只贴出来必选项，其他请自行扩展

```yaml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    client-name: root
    password: root
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        acknowledge-mode: manual
```

`RabbitMQ`配置类

```java
package com.config;

import org.springframework.amqp.core.*;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author 田奇杭
 * @Description 消息监听类
 * @Date 2022/5/8 22:32
 */
@Configuration
public class StoreFlowConsumerConfig {

    /**
     * 交换机 
     * TODO 为了规范，起三个接口来存放交换机、路由KEY、队列的名称，你们可以随意起名。
     * 例：private static final String STORE_FLOW_EXCHANGE_NAME = "我真的很帅，换成字母";
     */
    private static final String STORE_FLOW_EXCHANGE_NAME = RabbitExchangeInterface.STORE_FLOW_EXCHANGE_NAME;

    /**
     * 路由KEY
     */
    private static final String STORE_FLOW_ROUTING_KEY_NAME = RabbitRoutingKeyInterface.STORE_FLOW_ROUTING_KEY_NAME;

    /**
     * 队列
     */
    private static final String STORE_FLOW_QUEUE_NAME = RabbitQueueInterface.STORE_FLOW_QUEUE_NAME;

    /**
     * 创建交换机
     */
    @Bean("storeFlowExchange")
    public Exchange storeFlowExchange() {
        return ExchangeBuilder.topicExchange(STORE_FLOW_EXCHANGE_NAME).durable(true).build();
    }

    /**
     * 创建队列
     */
    @Bean("storeFlowQueue")
    public Queue storeFlowQueue() {
        return QueueBuilder.durable(STORE_FLOW_QUEUE_NAME).build();
    }

    /**
     * 通过路由键，将队列和交换机绑定
     */
    @Bean
    public Binding getBinding(@Qualifier("storeFlowQueue") Queue queue, @Qualifier("storeFlowExchange") Exchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with(STORE_FLOW_ROUTING_KEY_NAME).noargs();
    }

}
```

消息监听类

```java
package com.consumer;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.config.RabbitQueueInterface;
import com.rabbitmq.client.Channel;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.concurrent.TimeUnit;

/**
 * @author 田奇杭
 * @Description 消息消费类
 * @Date 2022/5/8 22:53
 */
@Slf4j
@Component
public class StoreFlowConsumer {

    /**
     * redis客户端，用于消息幂等性
     */
    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 简单测试（别说些的不合格，我还的面试呢/(ㄒoㄒ)/~~）
     *
     * @param message 消息体
     * @param channel 通道
     */
    @RabbitListener(queues = RabbitQueueInterface.STORE_FLOW_QUEUE_NAME)
    public void consumer(Message message, Channel channel) throws IOException {

        try {
            // 获取 messageId，在发送消息时赋予的全局ID，以便做消费记录查询，避免重复消费消息
            String messageId = message.getMessageProperties().getHeader("spring_returned_message_correlation");

            // 获取消息内容
            JSONObject jsonObject = JSON.parseObject(new String(message.getBody()));

            // 输出接收的参数
            log.info("消费者拿到的messageId：{}，获得内容信息为：{}", messageId, jsonObject.toString());
            // 引入第三方介质，做消费验证，判断 key 为 messageId 的消费记录是否存在，不存在说明消息未被消费
            if (Boolean.FALSE.equals(redisTemplate.hasKey(messageId))) {
                // 将消息全局ID作为 key，消息体作为 value 存入 redis，数据格式自己决定
                redisTemplate.opsForValue().set(messageId, "", 1, TimeUnit.HOURS);
            }
        } catch (Exception e) {
            log.error("客流数据处理失败，e：", e);
        } finally {
            // 程序执行完毕，手动返回ack
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        }
    }
}
```

