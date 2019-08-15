## 目录
* [哨兵的问题](#哨兵的问题)
* [predis存在的问题](#predis存在的问题)
* [整体运行流程](#整体流程)
* [核心逻辑源码](#核心逻辑源码)

# 哨兵的问题
  哨兵存在的问题  
- 哨兵宕机
- 通过哨兵获取节点，然后操作节点时候节点宕机
- 通过哨兵获取master节点，发送写命令，但是此时master节点已经变成了slave节点，不可写  

## 对于哨兵宕机问题：  
  可以使用多节点哨兵来监控主从节点，如一个哨兵节点连接异常等，可以使用下一个哨兵节点  
  
## 对于通过哨兵获取节点，然后操作节点时候节点宕机问题  
  可能节点已宕机，哨兵还在主观下线状态，重新从哨兵获取主从节点关系，直到节点进入客观下线状态并且故障转移结束

## 通过哨兵获取master节点，发送写命令，但是此时master节点已经变成了slave节点，不可写问题  
  可能节点已宕机并且重新上线，上线后会变成slave节点，需要重新从哨兵获取主从节点关系
  
以上问题predis框架都进行了处理  
# predis存在的问题
如果配置为如下
```
<?php
set_time_limit(0);
use Predis\Client;
require './vendor/autoload.php';
//服务器列表是哨兵的
$sentinels = ['tcp://192.168.124.10:26379', 'tcp://192.168.124.10:26380', 'tcp://192.168.124.10:26381'];
$options   = [
	'replication' => 'sentinel', 
	'service' => 'mymaster',  //哨兵中监控的主节点名称
	'parameters' => [
        'password' => 123456,   //主从节点的密码
 	],
 ];

$client = new Predis\Client($sentinels, $options);

$client -> set("name",123);
$client -> get("name");
```
每次predis会通过第一个哨兵配置去获取主从节点信息，第一个哨兵压力很大，可以优化为
```
$sentinels = shuffle(['tcp://192.168.124.10:26379', 'tcp://192.168.124.10:26380', 'tcp://192.168.124.10:26381']);
```
# 整体运行流程
  
* 初始化predis项目
* 判断是否已经有连接过的redis接点，获取命令是可读还是可写
  1. 如果节点是master则直接使用master节点去读写 
  2. 如果节点是slave，命令是读命令则直接使用slave节点
  3. 如果节点是slave，命令是写命令则通过哨兵获取master节点
- 如果没有连接的redis节点，客户端连接配置列表中的第一个哨兵，并将这个哨兵从配置列表属性中移除
  1. 需要获取master节点，向哨兵发送sentinel get-master-addr-by-name $service
  2. 需要获取slave节点，向哨兵发送sentinel slaves $service，过滤掉slave的状态为s_down、o_down、disconnected的节点
- 如果连接哨兵异常，则使用配置列表的第一个哨兵继续连接，并将这个哨兵从配置列表中移除，直到哨兵节点可用或者将全部配置列表属性为空
- 向redis节点发送命令
- 如果redis节点异常，则默认停止1秒，继续连接哨兵获取节点，然后使用获取的节点发送命令，重试20次，每次重试停止1秒

# 核心逻辑源码
获取节点发送命令  
```
private function retryCommandOnFailure(CommandInterface $command, $method)
{
    $retries = 0;
    SENTINEL_RETRY: {
        try {
            $response = $this->getConnection($command)->$method($command);
        } catch (CommunicationException $exception) {
	    //这里是redis节点异常重试机制
            $this->wipeServerList();
            $exception->getConnection()->disconnect();
            //默认重试20次
            if ($retries == $this->retryLimit) {
                throw $exception;
            }
            //默认每次重试停止1秒，让哨兵进行故障转移
            usleep($this->retryWait * 1000);

            ++$retries;
            goto SENTINEL_RETRY;
        }
    }

    return $response;
}
```
获取redis节点
```
public function getConnection(CommandInterface $command)
{
    //获取redis节点
    $connection = $this->getConnectionInternal($command);
    if (!$connection->isConnected()) {
        // When we do not have any available slave in the pool we can expect
        // read-only operations to hit the master server.
        $expectedRole = $this->strategy->isReadOperation($command) && $this->slaves ? 'slave' : 'master';
        $this->assertConnectionRole($connection, $expectedRole);
    }

    return $connection;
}
private function getConnectionInternal(CommandInterface $command)
{
    //如果还没有连接过的redis节点
    if (!$this->current) {
        //判断命令是可读还是可写，读命令连slave，写命令连master
        if ($this->strategy->isReadOperation($command) && $slave = $this->pickSlave()) {
            $this->current = $slave;
        } else {
            $this->current = $this->getMaster();
        }
        return $this->current;
    }
    //如果连接过redis节点，并且是master节点
    if ($this->current === $this->master) {
        return $this->current;
    }
    //如果连接的是slave redis节点，并且命令是写操作
    if (!$this->strategy->isReadOperation($command)) {
        //重新连接master节点
        $this->current = $this->getMaster();
    }

    return $this->current;
}
```
通过哨兵获取节点信息
```
public function getSentinelConnection()
{
    if (!$this->sentinelConnection) {
        if (!$this->sentinels) {
            throw new \Predis\ClientException('No sentinel server available for autodiscovery.');
        }
	//从哨兵配置列表中获取第一个哨兵，并且移除
        $sentinel = array_shift($this->sentinels);
        $this->sentinelConnection = $this->createSentinelConnection($sentinel);
    }

    return $this->sentinelConnection;
}
protected function querySentinelForMaster(NodeConnectionInterface $sentinel, $service)
{
    //向哨兵发送sentinel get-master-addr-by-name $service命令，获取master
    节点信息
    $payload = $sentinel->executeCommand(
        RawCommand::create('SENTINEL', 'get-master-addr-by-name', $service)
    );
    if ($payload === null) {
        throw new ServerException('ERR No such master with that name');
    }

    if ($payload instanceof ErrorResponseInterface) {
        $this->handleSentinelErrorResponse($sentinel, $payload);
    }

    return array(
        'host' => $payload[0],
        'port' => $payload[1],
        'alias' => 'master',
    );
}
public function getMaster()
{
    if ($this->master) {
        return $this->master;
    }
    if ($this->updateSentinels) {
        $this->updateSentinels();
    }

    SENTINEL_QUERY: {
        $sentinel = $this->getSentinelConnection();
        try {
	    //获取主节点信息
            $masterParameters = $this->querySentinelForMaster($sentinel, $this->service);
            $masterConnection = $this->connectionFactory->create($masterParameters);
            $this->add($masterConnection);
        } catch (ConnectionException $exception) {
            $this->sentinelConnection = null;

            goto SENTINEL_QUERY;
        }
    }

    return $masterConnection;
}
public function getSlaves()
{
    if ($this->slaves) {
        return array_values($this->slaves);
    }

    if ($this->updateSentinels) {
        $this->updateSentinels();
    }

    SENTINEL_QUERY: {
        $sentinel = $this->getSentinelConnection();

        try {
	    //获取slave节点信息
            $slavesParameters = $this->querySentinelForSlaves($sentinel, $this->service);
            //将所有slave节点信息添加进属性里面
            foreach ($slavesParameters as $slaveParameters) {
                $this->add($this->connectionFactory->create($slaveParameters));
            }
        } catch (ConnectionException $exception) {
            $this->sentinelConnection = null;

            goto SENTINEL_QUERY;
        }
    }

    return array_values($this->slaves ?: array());
}
protected function pickSlave()
{
    if ($slaves = $this->getSlaves()) {
        //会随机拿一个slave节点
        return $slaves[rand(1, count($slaves)) - 1];
    }
}
```
