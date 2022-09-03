`Redis`客户端使用`Lua`脚本示例

```lua
EVAL "redis.call('SET', KEYS[1], ARGV[1])" 1 age 16
```



`Lua`脚本

```lua
local result = redis.callp('SETNX', KEYS[1], ARGV[1])
if (result == true) then 

```





```lua
if (redis.call('exists', KEYS[1]) == 0) then 
    redis.call('hincrby', KEYS[1], ARGV[2], 1); 
    redis.call('pexpire', KEYS[1], ARGV[1]); 
    return 1; 
    end; 
return 0;
```

















## 调试中出新的问题

-   抢占锁未保证互斥性，因为参考的`AQS`，他在抢占时会同时判断`AOS`的线程和state从`0`设置为`1`，我这个因为state维护在`redis`，也不方便在每个队列中都塞一个`redis`客户端，就只是检查了`AOS`的线程是否为空，从而没有达到互斥的预期
-   fifo同步队列的node节点形成了死循环，node1.next=node2，node1





```lua
if (redis.call('EXISTS', KEYS[1]) == 0) then 
    redis.call('HINCRBY', KEYS[1], ARGV[2], 1); 
    redis.call('PEXPIRE', KEYS[1], ARGV[1]); 
    return nil; 
    end; 
if (redis.call('HEXISTS', KEYS[1], ARGV[2]) == 1) then
    redis.call('HINCRBY', KEYS[1], ARGV[2], 1); 
    redis.call('PEXPIRE', KEYS[1], ARGV[1]); 
    return nil; 
    end; 
return redis.call('PTTL', KEYS[1]);
```



```lua
if (redis.call('HEXISTS', KEYS[1], ARGV[2]) == 0) then 
    return nil;
end;
if (tonumber(redis.call('HGET', KEYS[1], ARGV[2])) > 1) then 
    redis.call('HINCRBY', KEYS[1], ARGV[2], -1); 
    redis.call('PEXPIRE', KEYS[1], ARGV[1]);
    return 0;
else 
    redis.call('DEL', KEYS[1]); 
    redis.call('PUBLISH', 'UN_LOCK_TOPIC', KEYS[1]); 
    return 1;
end; 
return nil;
```



```lua
redis.call('zrange', KEYS[1], ARGV[1],ARGV[2]) 3 {xxx}lock 0 1
```





















`Lua`脚本多层`if`、`else`怎么确定边界



-   `KEYS[1]`：锁名称
-   `KEYS[2]`：`fifo`对列
-   `ARGV[1]`：过期时间（预计解锁的时间）
-   `ARGV[2]`：线程标识（随机数加线程id）
-   `ARGV[3]`：预计解锁时间



```lua
if(redis.call('EXISTS', KEYS[1]) == 0) then
   redis.call('SETEX', KEYS[1], ARGV[2], ARGV[1]);
   return true;
   end;
if(redis.call('ZCARD', KEYS[2]) == 0) then
   local threadFlag = redis.call('GET', KEYS[1]);
   redis.call('ZADD', KEYS[2], 0,  threadFlag);
   redis.call('ZADD', KEYS[2], ARGV[3],  ARGV[2]);
   redis.call('PEXPIRE', KEYS[2], ARGV[1]);
   return false;
else
   redis.call('ZADD', KEYS[2], ARGV[3],  ARGV[2]);
   redis.call('PEXPIRE', KEYS[2], ARGV[1]);
   return false;
   end;
```



```lua
if(redis.call('GET', KEYS[1]) == ARGV[1]) then
   local threadFlag = redis.call('ZRANGE', KEYS[2], 1, 1);
   redis.call('SET', KEYS[1], threadFlag);
   redis.call('ZREM', KEYS[2], ARGV[1]);
   return true;
   end;
return false;
```



```lua
local  threadFlag = tostring(redis.call('LRANGE', 'age', 1, 1));
return redis.call('SET', 'lock', threadFlag);
```



`list、set、zset`这些类型使用`tostring`转换后都不是字符串而是`table`



### `java`客户端`lua`脚本

```java
package extend.constants;

public interface RedisLockScriptConstant {

    /**
     * 加锁
     * 一行代码一行注释，自行联想
     * key 尚未被抢占
     *      使用 hash 存储锁信息 key 持有锁的线程 重入次数
     *      设置传入的过期时间
     *      返回 true
     *      终止
     * key 的所有者是当前线程
     *      重入次数 +1
     *      设置传入的过期时间
     *      返回 true
     *      终止
     * 抢占锁失败返回 false
     */
    String LOCK_LUA = "if (redis.call('exists', KEYS[1]) == 0) then  " +
            "    redis.call('hincrby', KEYS[1], ARGV[2], 1);  " +
            "    redis.call('pexpire', KEYS[1], ARGV[1]);  " +
            "    return true;  " +
            "    end;  " +
            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
            "    redis.call('hincrby', KEYS[1], ARGV[2], 1);  " +
            "    redis.call('pexpire', KEYS[1], ARGV[1]);  " +
            "    return true;  " +
            "    end;  " +
            "return false;";

    /**
     * 释放锁
     *
     * key 不存在
     *      返回 false
     *      终止
     * key 的重入次数减一后大于 0
     *      设置传入的过期时间
     *      返回 true
     *      不大于 0
     *      删除 key，释放锁
     *      返回 true
     *      终止
     * 释放锁失败返回 false
     */
    String UNLOCK_LUA = "if (redis.call('hexists', KEYS[1], ARGV[2]) == 0) then " +
            "    return false; " +
            "    end; " +
            "if (tonumber(redis.call('hincrby', KEYS[1], ARGV[2], -1)) > 0) then " +
            "    redis.call('pexpire', KEYS[1], ARGV[1]); " +
            "    return true; " +
            "    else " +
            "    redis.call('del', KEYS[1]); " +
            "    return true; " +
            "    end; " +
            "return false;";
}
```

```java

    String UNLOCK_LUA = "if (redis.call('exists', KEYS[1]) == 0) then " +
            "    return false; " +
            "    end; " +
            "if (tonumber(redis.call('hincrby', KEYS[1], ARGV[2], -1)) > 0) then " +
            "    redis.call('pexpire', KEYS[1], ARGV[1]); " +
            "    return true; " +
            "    else " +
            "    redis.call('del', KEYS[1]); " +
            "    return true; " +
            "    end; " +
            "return false;";

```

```lua
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('pexpire', KEYS[1], ARGV[1]);
```



