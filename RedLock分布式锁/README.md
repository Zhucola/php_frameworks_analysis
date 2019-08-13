# 单机模式下的redis锁
||session A|session B|
|:---|:---|:---|
|1|set lock 1234 nx px 3000获得锁成功||
|2||3000毫秒内set lock 1234 nx px 3000获得锁失败|
