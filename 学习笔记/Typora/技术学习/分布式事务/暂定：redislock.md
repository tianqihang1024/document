# 基于LUA脚本的Redis分布式锁（SpringBoot实现）

[![img](https://upload.jianshu.io/users/upload_avatars/1627704/05b8394e-4a53-4860-b5eb-eab2c7ee5c05.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/2f53da5bf844)

[姜蒜儿](https://www.jianshu.com/u/2f53da5bf844)关注																==百度：		redis分布式锁lua脚本==			==分布式锁lua脚本==

22018.11.16 13:53:04字数 1,450阅读 3,452

### 1.前言

Redis实现分布式锁，本身比较简单，就是Redis中一个简单的KEY。一般都利用setnx（set if not exists）指令可以非常简单的实现加锁，锁用完后，再调用del指令释放锁。要确保锁可用，一般需要解决几个问题：

1. 不能出现死锁情况，一个获得锁的客户端宕机或者异常后，要保障其他客户端也能获得锁。
2. 应用程序通过网络与Redis交互，为避免网络延迟以及获取锁线程与其他线程不冲突，需要保障锁操作的原子性，既同一时间只有一个客户端可用获取到锁。
3. 加锁和解锁的客户端必须是同一个，不能把其他客户端加的锁给解了。
4. 考虑容错性，如果一个客户端加锁成功后，Redis集群Master宕掉并没有及时同步，另外一个客户端加锁会立即成功，避免同一把锁被两个客户端持有。

### 2.解决思路

1. 死锁问题，通常是在拿到锁后给锁设置一个过期时间（expire指令），即使出现异常，在过期时间后，锁也会自动释放

2. 原子性问题通常的两个解决方式：

	- 通过redis2.8版本后加入的set扩展参数，将加锁和设置过期时间作为一个原子操作，目前发现不是所有Java的Redis客户端都支持这样的set指令

	

	```redis
	set lock:test true ex 5 nx
	```

	- LUA脚本，Redis Lua脚本可以保证多条指令的原子性执行

3. 释放其他客户端锁，通过在加锁的时候指定随机值，在解锁的时候用这个随机值去匹配，匹配成功则解锁，匹配失败就不能解锁，因为锁可能已经过期或者已经被其他客户端占用

4. Redis集群宕掉的极端情况下，可以考虑redlock算法，但是算法本身复杂，而且带来一些性能损耗，可以根据实际场景判断，是否非常在乎这样的高可用

### 3.SpringBoot实现

##### 3.1 LUA脚本

本实现基于SpringBoot2x，考虑SpringBoot2x中Redis的默认连接是由lettuce提供，不是常用的Jedis，同时考虑不同版本的Redis，加锁和解锁都采用LUA脚本。



```lua
 -- 加锁脚本，其中KEYS[]为外部传入参数
 -- KEYS[1]表示key
 -- KEYS[2]表示value
 -- KEYS[3]表示过期时间
 if redis.call("setnx", KEYS[1], KEYS[2]) == 1 then
     return redis.call("pexpire", KEYS[1], KEYS[3])
 else
     return 0
```



```lua
 -- 解锁脚本
 -- KEYS[1]表示key
 -- KEYS[2]表示value
 -- return -1 表示未能获取到key或者key的值与传入的值不相等
 if redis.call("get",KEYS[1]) == KEYS[2] then
     return redis.call("del",KEYS[1])
 else
     return -1
```

##### 3.2 加锁代码

依赖SpringBoot的RedisTemplate执行LUA脚本，同时考虑重试机制



```java
    /**
     * 加锁
     * @param key Key
     * @param timeout 过期时间
     * @param retryTimes 重试次数
     * @return
     */
    public boolean lock(String key, long timeout, int retryTimes) {
        try {
            final String redisKey = this.getRedisKey(key);
            final String requestId = this.getRequestId();
            logger.debug("lock :::: redisKey = " + redisKey + " requestid = " + requestId);
            //组装lua脚本参数
            List<String> keys = Arrays.asList(redisKey, requestId, String.valueOf(timeout));
            //执行脚本
            Long result = redisTemplate.execute(LOCK_LUA_SCRIPT, keys);
            //存储本地变量
            if(!StringUtils.isEmpty(result) && result == LOCK_SUCCESS) {
                localRequestIds.set(requestId);
                localKeys.set(redisKey);
                logger.info("success to acquire lock:" + Thread.currentThread().getName() + ", Status code reply:" + result);
                return true;
            } else if (retryTimes == 0) {
                //重试次数为0直接返回失败
                return false;
            } else {
                //重试获取锁
                logger.info("retry to acquire lock:" + Thread.currentThread().getName() + ", Status code reply:" + result);
                int count = 0;
                while(true) {
                    try {
                        //休眠一定时间后再获取锁，这里时间可以通过外部设置
                        Thread.sleep(100);
                        result = redisTemplate.execute(LOCK_LUA_SCRIPT, keys);
                        if(!StringUtils.isEmpty(result) && result == LOCK_SUCCESS) {
                            localRequestIds.set(requestId);
                            localKeys.set(redisKey);
                            logger.info("success to acquire lock:" + Thread.currentThread().getName() + ", Status code reply:" + result);
                            return true;
                        } else {
                            count++;
                            if (retryTimes == count) {
                                logger.info("fail to acquire lock for " + Thread.currentThread().getName() + ", Status code reply:" + result);
                                return false;
                            } else {
                                logger.warn(count + " times try to acquire lock for " + Thread.currentThread().getName() + ", Status code reply:" + result);
                                continue;
                            }
                        }
                    } catch (Exception e) {
                        logger.error("acquire redis occured an exception:" + Thread.currentThread().getName(), e);
                        break;
                    }
                }
            }
        } catch (Exception e1) {
            logger.error("acquire redis occured an exception:" + Thread.currentThread().getName(), e1);
        }
        return false;
    }
```

1. getRedisKey根据传入的key加上一个前缀生成锁的key

	

	```java
	 /**
	  * 获取RedisKey
	  * @param key 原始KEY，如果为空，自动生成随机KEY
	  * @return
	  */
	 private String getRedisKey(String key) {
	     //如果Key为空且线程已经保存，直接用，异常保护
	     if (StringUtils.isEmpty(key) && !StringUtils.isEmpty(localKeys.get())) {
	         return localKeys.get();
	     }
	     //如果都是空那就抛出异常
	     if (StringUtils.isEmpty(key) && StringUtils.isEmpty(localKeys.get())) {
	         throw new RuntimeException("key is null");
	     }
	     return LOCK_PREFIX + key;
	 }
	```

2. getRequestId用于为每一个加锁请求生成请求ID，内部方法

	

	```java
	 /**
	  * 获取随机请求ID
	  * @return
	  */
	 private String getRequestId() {
	     return UUID.randomUUID().toString();
	 }
	```

3. redisTemplate.execute(LOCK_LUA_SCRIPT, keys)，execute最终调用的RedisConnection的eval方法将LUA脚本交给Redis服务端执行，可兼容springboot中不同redis客户端实现（Jedis、Lettuce等）。这个操作通过setnx设置锁key，成功后设置锁的有效期，成功返回1，失败返回0，其中LOCK_LUA_SCRIPT为常量定义

	

	```java
	 //定义获取锁的lua脚本
	 private final static DefaultRedisScript<Long> LOCK_LUA_SCRIPT = new DefaultRedisScript<>(
	         "if redis.call(\"setnx\", KEYS[1], KEYS[2]) == 1 then return redis.call(\"pexpire\", KEYS[1], KEYS[3]) else return 0 end"
	         , Long.class
	 );
	```

4. 根据脚本执行情况，将锁的key和requestId分别存入线程本地变量localKeys和localRequestIds中，两个都是ThreadLocal变量，通过两个变量在释放锁的时候避免释放其他客户端占用的锁。

5. 根据重试次数retryTimes值进行重试判断，如果为0则不重试，否则进入重试逻辑。

##### 3.3 解锁代码



```java
    /**
     * 释放KEY
     * @param key
     * @return
     */
    public boolean unlock(String key) {
        try {
            String localKey = localKeys.get();
            //如果本地线程没有KEY，说明还没加锁，不能释放
            if(StringUtils.isEmpty(localKey)) {
                logger.error("release lock occured an error: lock key not found");
                return false;
            }
            String redisKey = getRedisKey(key);
            //判断KEY是否正确，不能释放其他线程的KEY
            if(!StringUtils.isEmpty(localKey) && !localKey.equals(redisKey)) {
                logger.error("release lock occured an error: illegal key:" + key);
                return false;
            }
            //组装lua脚本参数
            List<String> keys = Arrays.asList(redisKey, localRequestIds.get());
            logger.debug("unlock :::: redisKey = " + redisKey + " requestid = " + localRequestIds.get());
            // 使用lua脚本删除redis中匹配value的key，可以避免由于方法执行时间过长而redis锁自动过期失效的时候误删其他线程的锁
            Long result = redisTemplate.execute(UNLOCK_LUA_SCRIPT, keys);
            //如果这里抛异常，后续锁无法释放
            if (!StringUtils.isEmpty(result) && result == RELEASE_SUCCESS) {
                logger.info("release lock success:" + Thread.currentThread().getName() + ", Status code reply=" + result);
                return true;
            } else if (!StringUtils.isEmpty(result) && result == LOCK_EXPIRED) {
                //返回-1说明获取到的KEY值与requestId不一致或者KEY不存在，可能已经过期或被其他线程加锁
                // 一般发生在key的过期时间短于业务处理时间，属于正常可接受情况
                logger.warn("release lock exception:" + Thread.currentThread().getName() + ", key has expired or released. Status code reply=" + result);
            } else {
                //其他情况，一般是删除KEY失败，返回0
                logger.error("release lock failed:" + Thread.currentThread().getName() + ", del key failed. Status code reply=" + result);
            }
        } catch (Exception e) {
            logger.error("release lock occured an exception", e);
        } finally {
            //清除本地变量
            this.clean();
        }
        return false;
    }
```

1. 如果本地线程localKeys中无法获取到key，或者获取到的key与传入的不一致，解锁失败

2. redisTemplate.execute(UNLOCK_LUA_SCRIPT, keys) 将LUA脚本交给Redis服务端执行。UNLOCK_LUA_SCRIPT常量定义，先判断key值是否与传入的requestId一致，如果一致则删除key，如果不一致返回-1表示key可能已经过期或被其他客户端占用，避免误删。

	

	```java
	 //定义释放锁的lua脚本
	 private final static DefaultRedisScript<Long> UNLOCK_LUA_SCRIPT = new DefaultRedisScript<>(
	         "if redis.call(\"get\",KEYS[1]) == KEYS[2] then return redis.call(\"del\",KEYS[1]) else return -1 end"
	         , Long.class
	 );
	```

3. 最后通过clean做清理工作

	

	```java
	 /**
	  * 清除本地线程变量，防止内存泄露
	  */
	 private void clean() {
	     localRequestIds.remove();
	     localKeys.remove();
	 }
	```

### 4.后记

1. 可将锁改成注解方式，通过AOP降低锁使用的复杂度
2. 重试机制可以根据业务情况进行优化
3. 可以更进一步借助ThreadLocal保存锁计数器可实现类似ReentrantLock可重入锁机制
4. 释放锁失败后可以加入回调方法进行一些业务处理
5. 如果业务挂起或者执行时间过长，超过了锁的超时时间，另外的客户端可能提前获取到锁，导致临界区代码不能严格的串行执行。除了合理设置锁超时时间外，尽量不要把分布式锁用于执行时间长的任务

### 5.补充

##### 5.1 RedisTemplate加载



```java
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.io.Serializable;

/**
 * @Description Redis配置类，替代SpringBoot自动配置的RedisTemplate，参加RedisAutoConfiguration
 * 这个类没有设置序列化方式等
 * @Author Gazza Jiang
 * @Date 2018/11/12 9:30
 * @Version 1.0
 */
@Configuration
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Serializable> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Serializable> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        //Jackson序列化器
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        //字符串序列化器
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        //普通Key设置为字符串序列化器
        template.setKeySerializer(stringRedisSerializer);
        //Hash结构的key设置为字符串序列化器
        template.setHashKeySerializer(stringRedisSerializer);
        //普通值和hash的值都设置为jackson序列化器
        template.setValueSerializer(jackson2JsonRedisSerializer);
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }
}
```

##### 5.2 简单测试类



```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import xyz.gazza.demo.redis.Application;
import xyz.gazza.demo.redis.lock.RedisLock;

import java.util.ArrayList;

/**
 * @Description 测试类
 * @Author Gazza Jiang
 * @Date 2018/11/12 13:29
 * @Version 1.0
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {Application.class})
public class ApplicationTest {
    private final Logger logger = LoggerFactory.getLogger(ApplicationTest.class);
    @Autowired
    RedisLock redisLock;
    @Test
    public void testRedisLock() throws InterruptedException {
        ArrayList<Thread> list = new ArrayList<>();
        for(int i =0; i<10; i++) {
            //logger.info("线程开始");
            Thread t = new Thread() {
                @Override
                public void run() {
                    if (redisLock.lock("suaner")) {
                        try {
                            //成功获取锁
                            logger.info("获取锁成功，继续执行任务" + Thread.currentThread().getName());
                            try {
                                Thread.sleep(10);
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }catch (Exception e) {
                            logger.error("excepiotn ", e);
                        } finally {
                            redisLock.unlock("suaner");
                        }
                    }
                }
            };
            list.add(t);
            t.start();
        }
        for(Thread t : list) {
            t.join();
        }
        Thread.sleep(10000);
    }
}
```

##### 5.3 pom依赖



```maven
 <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```