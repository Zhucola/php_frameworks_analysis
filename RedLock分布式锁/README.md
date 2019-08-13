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
|5||set lock 1234 nx px 3000 3000毫秒内，获得锁成功|
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

