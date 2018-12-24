#### Redis实现分布式限流
用来限制API在一定时间内的访问次数
```java
private Object limitRequest(Object connection) {
        Object result = null;
        //请求的key
        String key = String.valueOf(System.currentTimeMillis() / 1000);
        if (connection instanceof Jedis){
            result = ((Jedis)connection).eval(script, Collections.singletonList(key), Collections.singletonList(String.valueOf(limit)));
            ((Jedis) connection).close();
        }else {
            result = ((JedisCluster) connection).eval(script, Collections.singletonList(key), Collections.singletonList(String.valueOf(limit)));
            try {
                ((JedisCluster) connection).close();
            } catch (IOException e) {
                logger.error("IOException",e);
            }
        }
        return result;
    }
```
```lua
--lua 下标从 1 开始
-- 限流 key KEYS[1] 指第一个key
local key = KEYS[1]
-- 限流大小 ARGV[1] 指key后面第一个参数
local limit = tonumber(ARGV[1])

-- 获取当前流量大小
local curentLimit = tonumber(redis.call('get', key) or "0")

if curentLimit + 1 > limit then
    -- 达到限流大小 返回
    return 0;
else
    -- 没有达到阈值 value + 1
    redis.call("INCRBY", key, 1)
    -- 重新设定时间
    redis.call("EXPIRE", key, 2)
    return curentLimit + 1
end

```
* 判断Redis里面的key的次数是否达到limit,如果达到，就返回限制错误。否则累加1。