核心逻辑源码部分可以不看，因为predis源码很绕，我就是把代码贴过来而已，有兴趣的童鞋可以自己去追下  

## 目录
* [整体运行流程](#整体流程)
* [CRC16算法源码](#源码分析)
* [核心逻辑源码](#核心逻辑源码)

# 整体运行流程
* 初始化项目(源码很绕，有兴趣的同学可以自己去追一下，就是Client的构造方法)
* 假如客户端执行set操作，会根据key做slot，并且根据slot去获取一个节点
    1. 如果节点配置中指定了slots，就获取指定的slot对应的节点(如50的slot会分配到7000端口，注意这个可能不是真实的redis槽的分配关系，比如redis做了槽的修改而PHP程序没有更新)
    2. 如果节点配置中没有指定slots或者指定的slots不匹配，就去猜节点(guessNode)，猜节点会认为你槽分配是平均的(如有3个节点，配置列表中的第一个节点会认为是0-5469，第二个是5461-10922，第三个是10923-16383，然后用slot去匹配节点)
    
指定slots对节点对应关系的配置如下
```
$redis_list = [
        'redis://192.168.124.10:7000?slots=1-100,500-1000',
        'redis://192.168.124.10:7001?slots=101-499',
        'redis://192.168.124.10:7002'
];
$redis = new Client($redis_list, ['cluster'=>'redis']);
``` 
猜节点算法如下
```
$count = count($this->pool);
$index = min((int) ($slot / (int) (16384 / $count)), $count - 1);
$nodes = array_keys($this->pool);
return $nodes[$index];
```
* 将set命令格式化，如果命令是
```
$redis->set("a",1234);
```
会变成
```
*3
$3
SET
$1
a
$4
1234
```
* 创建与节点的连接，并且发送密码和数据库号(如果配置的话)
    1. 如果连接失败，将该节点从配置列表中删除，然后从配置列表中随机取一个节点重新连接
* 将格式化后的命令发给redis节点
    1. 如果命令发送失败，将该节点从配置列表中删除，然后从配置列表中随机取一个节点重新连接
* 读取redis节点的响应
    1. 有MOVED情况，就是节点与slot的对应关系和真实的redis服务器不一样，处理MOVED信息，获取真实的slot和节点信息(从响应信息中获取)
        * 给MOVED的节点发cluster slots获取集群所有节点的slot信息、主从关系信息，将slot与主节点的对应关系存进变量，再次操作redis就直接用真实的对应关系获取节点了
        * 如果给MOVED节点发cluster slots失败，就配置的节点列表中删除此节点，并且随机返回一个节点去做cluster slots，有5次重试机会
        * 然后会回到初始化后的第二步操作(因为现在已经有真是的slot和节点的对应关系了，所以就不需要再guessNode了)
    2. 响应成功，收到OK
MOVED的响应
```
- MOVED 15495 127.0.0.1:7002 
```
成功的响应
```
+ OK
```
# CRC16算法源码
根据key获取slot的方法在predis\src\Cluster\ClusterStrategy.php里面，getSlot
```
public function getSlot(CommandInterface $command)
{
    $slot = $command->getSlot();
    if (!isset($slot) && isset($this->commands[$cmdID = $command->getId()])) {

        $key = call_user_func($this->commands[$cmdID], $command);
        if (isset($key)) {
            $slot = $this->getSlotByKey($key);
            $command->setSlot($slot);
        }
    }

    return $slot;
}
```
会调用getSlotByKey方法
```
public function getSlotByKey($key)
{
    $key = $this->extractKeyTag($key);  //获取key
    $slot = $this->hashGenerator->hash($key) & 0x3FFF;   //做hash后和16383取余

    return $slot;
}
```
核心的CRC16算法在predis\src\Cluster\Hash\CRC16.php，可以直接拿来用
```
private static $CCITT_16 = array(
    0x0000, 0x1021, 0x2042, 0x3063, 0x4084, 0x50A5, 0x60C6, 0x70E7,
    0x8108, 0x9129, 0xA14A, 0xB16B, 0xC18C, 0xD1AD, 0xE1CE, 0xF1EF,
    0x1231, 0x0210, 0x3273, 0x2252, 0x52B5, 0x4294, 0x72F7, 0x62D6,
    0x9339, 0x8318, 0xB37B, 0xA35A, 0xD3BD, 0xC39C, 0xF3FF, 0xE3DE,
    0x2462, 0x3443, 0x0420, 0x1401, 0x64E6, 0x74C7, 0x44A4, 0x5485,
    0xA56A, 0xB54B, 0x8528, 0x9509, 0xE5EE, 0xF5CF, 0xC5AC, 0xD58D,
    0x3653, 0x2672, 0x1611, 0x0630, 0x76D7, 0x66F6, 0x5695, 0x46B4,
    0xB75B, 0xA77A, 0x9719, 0x8738, 0xF7DF, 0xE7FE, 0xD79D, 0xC7BC,
    0x48C4, 0x58E5, 0x6886, 0x78A7, 0x0840, 0x1861, 0x2802, 0x3823,
    0xC9CC, 0xD9ED, 0xE98E, 0xF9AF, 0x8948, 0x9969, 0xA90A, 0xB92B,
    0x5AF5, 0x4AD4, 0x7AB7, 0x6A96, 0x1A71, 0x0A50, 0x3A33, 0x2A12,
    0xDBFD, 0xCBDC, 0xFBBF, 0xEB9E, 0x9B79, 0x8B58, 0xBB3B, 0xAB1A,
    0x6CA6, 0x7C87, 0x4CE4, 0x5CC5, 0x2C22, 0x3C03, 0x0C60, 0x1C41,
    0xEDAE, 0xFD8F, 0xCDEC, 0xDDCD, 0xAD2A, 0xBD0B, 0x8D68, 0x9D49,
    0x7E97, 0x6EB6, 0x5ED5, 0x4EF4, 0x3E13, 0x2E32, 0x1E51, 0x0E70,
    0xFF9F, 0xEFBE, 0xDFDD, 0xCFFC, 0xBF1B, 0xAF3A, 0x9F59, 0x8F78,
    0x9188, 0x81A9, 0xB1CA, 0xA1EB, 0xD10C, 0xC12D, 0xF14E, 0xE16F,
    0x1080, 0x00A1, 0x30C2, 0x20E3, 0x5004, 0x4025, 0x7046, 0x6067,
    0x83B9, 0x9398, 0xA3FB, 0xB3DA, 0xC33D, 0xD31C, 0xE37F, 0xF35E,
    0x02B1, 0x1290, 0x22F3, 0x32D2, 0x4235, 0x5214, 0x6277, 0x7256,
    0xB5EA, 0xA5CB, 0x95A8, 0x8589, 0xF56E, 0xE54F, 0xD52C, 0xC50D,
    0x34E2, 0x24C3, 0x14A0, 0x0481, 0x7466, 0x6447, 0x5424, 0x4405,
    0xA7DB, 0xB7FA, 0x8799, 0x97B8, 0xE75F, 0xF77E, 0xC71D, 0xD73C,
    0x26D3, 0x36F2, 0x0691, 0x16B0, 0x6657, 0x7676, 0x4615, 0x5634,
    0xD94C, 0xC96D, 0xF90E, 0xE92F, 0x99C8, 0x89E9, 0xB98A, 0xA9AB,
    0x5844, 0x4865, 0x7806, 0x6827, 0x18C0, 0x08E1, 0x3882, 0x28A3,
    0xCB7D, 0xDB5C, 0xEB3F, 0xFB1E, 0x8BF9, 0x9BD8, 0xABBB, 0xBB9A,
    0x4A75, 0x5A54, 0x6A37, 0x7A16, 0x0AF1, 0x1AD0, 0x2AB3, 0x3A92,
    0xFD2E, 0xED0F, 0xDD6C, 0xCD4D, 0xBDAA, 0xAD8B, 0x9DE8, 0x8DC9,
    0x7C26, 0x6C07, 0x5C64, 0x4C45, 0x3CA2, 0x2C83, 0x1CE0, 0x0CC1,
    0xEF1F, 0xFF3E, 0xCF5D, 0xDF7C, 0xAF9B, 0xBFBA, 0x8FD9, 0x9FF8,
    0x6E17, 0x7E36, 0x4E55, 0x5E74, 0x2E93, 0x3EB2, 0x0ED1, 0x1EF0,
);
public function hash($value)
{
    // CRC-CCITT-16 algorithm
    $crc = 0;
    $CCITT_16 = self::$CCITT_16;

    $value = (string) $value;
    $strlen = strlen($value);

    for ($i = 0; $i < $strlen; ++$i) {
        $crc = (($crc << 8) ^ $CCITT_16[($crc >> 8) ^ ord($value[$i])]) & 0xFFFF;
    }

    return $crc;
}
```
这个算法和redis算slot的结果是一样的，a的在PHP中算的slot也是15495
```
127.0.0.1:7001> set a 123
-> Redirected to slot [15495] located at 127.0.0.1:7002
OK
```
# 核心逻辑源码
Cluster执行一条命令走的是predis\src\Connection\Aggregate\PredisCluster.php里面的executeCommand方法
```
public function executeCommand(CommandInterface $command)
{
    $response = $this->retryCommandOnFailure($command, __FUNCTION__);

    if ($response instanceof ErrorResponseInterface) {   //move或者master不可用会执行
        return $this->onErrorResponse($command, $response);
    }

    return $response;
}
```
retryCommandOnFailure方法就是去连接客户端、执行命令、捕获异常方法，这个方法居然用到了goto，这是我第一次在PHP程序里面看到goto
```
private function retryCommandOnFailure(CommandInterface $command, $method)
{
    $failure = false;
    RETRY_COMMAND: {
        try {
            $response = $this->getConnection($command)->$method($command);
        } catch (ConnectionException $exception) {
            $connection = $exception->getConnection();
            $connection->disconnect();

            $this->remove($connection);

            if ($failure) {
                throw $exception;
            } elseif ($this->useClusterSlots) {
                $this->askSlotsMap();
            }

            $failure = true;

            goto RETRY_COMMAND;
        }
    }

    return $response;
}
```
由于php客户端不知道这个slot应该连接哪个redis节点，所以predis需要去猜一个节点
```
public function getConnection(CommandInterface $command)
{
    //获取slot
    $slot = $this->strategy->getSlot($command);
    if (!isset($slot)) {
        throw new NotSupportedException(
            "Cannot use '{$command->getId()}' with redis-cluster."
        );
    }

    if (isset($this->slots[$slot])) {
        //如果这个slot和节点有对应关系
        return $this->slots[$slot];
    } else {
        //根据slot来猜一个节点
        return $this->getConnectionBySlot($slot);
    }
}
public function getConnectionBySlot($slot)
{
    //判断slot是否合法
    if ($slot < 0x0000 || $slot > 0x3FFF) {
        throw new \OutOfBoundsException("Invalid slot [$slot].");
    }

    if (isset($this->slots[$slot])) {
        return $this->slots[$slot];
    }
    //猜一个节点
    $connectionID = $this->guessNode($slot);
    if (!$connection = $this->getConnectionById($connectionID)) {
        $connection = $this->createConnection($connectionID);
        $this->pool[$connectionID] = $connection;
    }
    //先存一个slot和猜出来的节点的对应关系
    return $this->slots[$slot] = $connection;
}
```
