







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







```lua
if (redis.call('exists', KEYS[1]) == 0) then 
    redis.call('hincrby', KEYS[1], ARGV[2], 1); 
    redis.call('pexpire', KEYS[1], ARGV[1]); 
    return 1; 
    end; 
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1); 
    redis.call('pexpire', KEYS[1], ARGV[1]); 
    return 1; 
    end; 
return 0;
```



```lua
if (redis.call('hexists', KEYS[1], ARGV[2]) == 0) then 
    return false;
    end; 
local counter = redis.call('hincrby', KEYS[1], ARGV[2], -1); 
if (counter > 0) then 
    redis.call('pexpire', KEYS[1], ARGV[1]); 
    else 
    redis.call('del', KEYS[1]); 
    return true; 
    end; 
return false;

```























`Lua`脚本多层`if`、`else`怎么确定边界













