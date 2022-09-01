```java
package com.lingyun.shop.manager.config;

import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {

    @Bean // 创建消息队列，有的话不生效
    public Queue helloQueue(){
        return new Queue("hello");
    }
}
```