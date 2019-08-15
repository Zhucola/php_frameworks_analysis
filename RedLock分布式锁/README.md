
## 目录
* [单机模式下的redis锁](#单机模式下的redis锁)
* [主从模式下的redis锁](#主从模式下的redis锁)
* [集群模式下的redis锁](#集群模式下的redis锁)
* [删除锁操作](#删除锁操作)
* [删除锁操作的改进](#删除锁操作的改进)
* [多节点轮流请求的超时时间](#多节点轮流请求的超时时间)
* [php版本redlock源码](#php版本redlock源码)

# 单机模式下的redis锁
||client A|client B|
|:---|:---|:---|
|1|set lock 1234 nx px 3000获得锁成功||
|2||set lock 1234 nx px 3000 3000毫秒内，获得锁失败|
|3||set lock 1234 nx px 3000 3000毫秒外，获得锁成功|
  
可见单机模式下的redis锁是可以做到互斥性，也不会发生死锁  
但是最大的问题是如果这个redis单机宕机，会造成整个服务不可用
# 主从模式下的redis锁
||client A|client B|
|:---|:---|:---|
|1|set lock 1234 nx px 3000获得锁成功，master同步给slave||
|2||set lock 1234 nx px 3000 3000毫秒内，获得锁失败|
|3||set lock 1234 nx px 3000 3000毫秒外，获得锁成功，master同步给slave|
|4|set lock 1234 nx px 3000 3000毫秒外，获得锁成功，master宕机，lock未同步给slave，slave升级为master||
|5||set lock 1234 nx px 3000 3000毫秒内，获得锁成功，因为新的master没有lock锁|
  
由于redis主从是异步复制  
- 客户端发送写命令给master
- master执行写命令，将结果返回给客户端
- 执行成功则master将命令同步给slave
  
可见在主从模式下锁会造成互斥性失效

# 集群模式下的redis锁
因为cluster模式，需要根据键做crc32，然后去判断键分配到那个槽里面，所以集群模式可以理解为单机模式

# 删除锁操作
在客户端拿到锁并进行业务逻辑后，需要进行锁删除操作，以便尽快解锁
  
||client A|client B|
|:---|:---|:---|
|1|set lock 1234 nx px 3000获得锁成功||
|2||set lock 1234 nx px 3000 第1500毫秒，获得锁失败|
|3|第2000毫秒，业务逻辑处理完毕，del lock||
|4||第2200毫秒，set lock 1234 nx px 3000 获得锁成功|
  
以上情况开起来很美好，但是考虑如下情况  
  
||client A|client B|
|:---|:---|:---|
|1|set lock 1234 nx px 3000获得锁成功||
|2||set lock 1234 nx px 3000 第1500毫秒，获得锁失败|
|3|第2000毫秒，业务逻辑处理完毕，del lock，命令在网络上发生阻塞，没有传递给redis客户端||
|4||第2200毫秒，set lock 1234 nx px 3000 获得锁失败，因为redis没有收到client A的del操作|
|5||第4000毫秒，set lock 1234 nx px 3000获得锁成功|
|6|del命令被redis接受|lock未超时，client B未处理完业务逻辑|
|7|set lock 1234 nx px 3000获得锁成功||
  
可见这种删除操作无法做到互斥性，client A将client B拿到的锁给删除了
# 删除锁操作的改进
每次生成lock的值都是随机的，然后使用lua脚本来删除，这个删除操作是一个原子性操作
```
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```
使用redis监视器可以查看到这个lua脚本的执行过程
```
127.0.0.1:6380> monitor
ok
1565688900.609818 [0 127.0.0.1:34344] "EVAL" "\r\n    if redis.call(\"GET\", KEYS[1]) == ARGV[1] then\r\n        return redis.call(\"DEL\", KEYS[1])\r\n    else\r\n        return 0\r\n    end" "1" "aaa" "bbb"
1565688900.609912 [0 lua] "GET" "aaa"
1565688900.609918 [0 lua] "DEL" "aaa"
```
流程改进如下  
  
||client A|client B|
|:---|:---|:---|
|1|随机生成锁的值为rand1，set lock rand1 nx px 3000获得锁成功||
|2||随机生成锁的值为rand2，set lock rand2 nx px abcd 第1500毫秒，获得锁失败|
|3|第2000毫秒，业务逻辑处理完毕，执行删除lua脚本，命令在网络上发生阻塞，没有传递给redis客户端||
|4||第2200毫秒，随机生成锁的值为rand3，set lock rand3 nx px 3000 获得锁失败，因为redis没有收到client A的del操作|
|5||第4000毫秒，随机生成锁的值为rand4，set lock rand4 nx px 3000获得锁成功|
|6|lua脚本被redis接受，因为值不相同所以删除失败|lock未超时，client B未处理完业务逻辑|
|7|随机生成锁的值为rand5，set lock rand5 nx px 3000获得锁失败||

# 多节点轮流请求的超时时间
  因为不能使用单机、主从和cluster，所以我们使用多节点，每个节点都不能有主从，当请求有count(servers_map) / 2 + 1个数量成功则认为是获取锁成功  
    
  如有5个redis节点，4个节点都成功set了，就认为获取锁成功  
  
||redis A|redis B|redis C|redis D|redis E|
|:---|:---|:---|:---|:---|:---|
|1|set lock 1234 nx px 3000获得锁成功|set lock 1234 nx px 3000获得锁成功|set lock rand nx px 3000获得锁成功|set lock 1234 nx px 3000获得锁失败|set lock 1234 nx px 3000获得锁成功|
  
  因为是轮流请求redis节点的，会造成请求拿锁的时机超过锁的超时时间情况
  - 锁超时时间为2秒
  - 轮流请求5个redis节点，前4个节点锁成功，共用时1秒，第5个节点用时3秒
  - 认为拿锁成功，其他客户端同样可以拿到锁，但是前4个节点的锁已经过期
    
  解决方案是记录拿锁请求开始时间和拿锁结束时间，对比超时时间  
  - 记录请求开始时间  
  - 锁超时时间为2秒
  - 轮流请求5个redis节点，前4个节点锁成功，共用时1秒，第5个节点用时3秒，共用时4秒
  - 记录请求结束时间
  - 因为总请求5秒大于锁超时2秒时间，所以认为拿锁失败
  - 轮流放锁
 
 # php版本redlock源码
 客户端demo如下
 ```
 <?php

require_once __DIR__ . './src/RedLock.php';

$servers = [
    ['192.168.124.10', 6379,1],
    ['192.168.124.10', 6379,1],
    ['192.168.124.10', 6379,1],
];

$redLock = new RedLock($servers);

while (true) {
    $lock = $redLock->lock('test', 10000);
    if ($lock) {
        print_r($lock);
    } else {
        print "Lock not acquired\n";
    }
    sleep(1);
}

```
构造函数如下
```
function __construct(array $servers, $retryDelay = 200, $retryCount = 3)
{
    //拿锁的redis节点列表
    $this->servers = $servers;
    //拿锁失败重试的间隔时间
    $this->retryDelay = $retryDelay;
    //拿锁失败的重试次数
    $this->retryCount = $retryCount;
    //多少个服务器拿锁成功就认为拿锁成功
    $this->quorum  = min(count($servers), (count($servers) / 2 + 1));
}
```
拿锁操作
```
public function lock($resource, $ttl)
{
    //实例化redis
    $this->initInstances();
    //生成随机的值
    $token = uniqid();

    $retry = $this->retryCount;

    do {
        $n = 0;

        $startTime = microtime(true) * 1000;
        //循环拿锁
        foreach ($this->instances as $instance) {
            if ($this->lockInstance($instance, $resource, $token, $ttl)) {
                $n++;
            }
        }

        //锁超时时间的一个偏移量
        $drift = ($ttl * $this->clockDriftFactor) + 2;
        //拿锁请求的时间间隔
        $validityTime = $ttl - (microtime(true) * 1000 - $startTime) - $drift;
        //判断是否拿锁成功
        if ($n >= $this->quorum && $validityTime > 0) {
            return [
                'validity' => $validityTime,
                'resource' => $resource,
                'token'    => $token,
            ];

        } else {
            //放锁操作
            foreach ($this->instances as $instance) {
                $this->unlockInstance($instance, $resource, $token);
            }
        }

        //重试时间间隔
        $delay = mt_rand(floor($this->retryDelay / 2), $this->retryDelay);
        usleep($delay * 1000);

        $retry--;

    } while ($retry > 0);

    return false;
}
```
锁命令
```
private function lockInstance($instance, $resource, $token, $ttl)
{
    return $instance->set($resource, $token, ['NX', 'PX' => $ttl]);
}
```
放锁操作
```
public function unlock(array $lock)
{
    $this->initInstances();
    $resource = $lock['resource'];
    $token    = $lock['token'];

    foreach ($this->instances as $instance) {
        $this->unlockInstance($instance, $resource, $token);
    }
}
private function unlockInstance($instance, $resource, $token)
{
    $script = '
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    ';
    return $instance->eval($script, [$resource, $token], 1);
}
```
