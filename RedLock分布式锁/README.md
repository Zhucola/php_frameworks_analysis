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
  - 轮流请求5个redis节点，4个节点锁成功，但是一共用时5秒
  - 认为拿锁成功，其他客户端同样可以拿到锁
 
