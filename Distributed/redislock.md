#### Redis实现分布式锁
实现分布式锁通常有以下3种方式
1. 基于 DB 的唯一索引。
2. 基于 ZK 的临时有序节点。
3. 基于 Redis 的 NX EX 参数。

通过Redis实现分布式锁的要点：
* 高性能(加、解锁时高性能)
* 可以使用阻塞锁与非阻塞锁。
* 不能出现死锁。
* 可用性(不能出现节点 down 掉后加锁失败)。
##### 获取锁
```java
public boolean lock(String key, String request, int blockTime) throws InterruptedException {

        //get connection
        Object connection = getConnection();
        String result ;
        //阻塞获取锁
        while (blockTime >= 0) {
            if (connection instanceof Jedis){
                result = ((Jedis) connection).set(lockPrefix + key, request, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, 10 * TIME) ;
                if (LOCK_MSG.equals(result)){
                    ((Jedis) connection).close();
                }
            }else {
                result = ((JedisCluster) connection).set(lockPrefix + key, request, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, 10 * TIME) ;
            }
            if (LOCK_MSG.equals(result)) {
                return true;
            }
            blockTime -= sleepTime;
            //防止空轮询消耗CPu
            Thread.sleep(sleepTime);
        }
        return false;
    }
```
NX+EX的组合，能够获取到锁。

#### 释放锁(通过lua脚本)
```java
public boolean unlock(String key, String request) {
        //get connection
        Object connection = getConnection();
        //lua script

        Object result = null;
        if (connection instanceof Jedis) {
            result = ((Jedis) connection).eval(script, Collections.singletonList(lockPrefix + key), Collections.singletonList(request));
            ((Jedis) connection).close();
        } else if (connection instanceof JedisCluster) {
            result = ((JedisCluster) connection).eval(script, Collections.singletonList(lockPrefix + key), Collections.singletonList(request));

        } else {
            //throw new RuntimeException("instance is error") ;
            return false;
        }

        if (UNLOCK_MSG.equals(result)) {
            return true;
        } else {
            return false;
        }
    }
```
>必须同Lua脚本判断释放锁的是加锁的请求，因此需要在Lua里面判断。

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```