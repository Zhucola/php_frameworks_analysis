## 目录
* [哨兵的问题](#哨兵的问题)
* [整体运行流程](#整体流程)
* [核心逻辑源码](#核心逻辑源码)

# 哨兵的问题
  哨兵存在的问题  
- 哨兵宕机
- 通过哨兵获取节点，然后操作节点时候节点宕机
- 通过哨兵获取master节点，发送写命令，但是此时master节点已经变成了slave节点，不可写  

对于哨兵宕机问题：  
  可以使用多节点哨兵来监控主从节点，如一个哨兵节点连接异常等，可以使用下一个哨兵节点  
  
对于通过哨兵获取节点，然后操作节点时候节点宕机问题  
  循环从哨兵获取节点信息多次，然后重试发送命令操作，这时候可能节点已宕机，或者哨兵正在做failover操作，如果是failover则通过哨兵会获取新的节点关系信息  

通过哨兵获取master节点，发送写命令，但是此时master节点已经变成了slave节点，不可写问题  
  循环从哨兵获取节点信息多次，然后重试发送命令操作，通过哨兵会获取新的节点关系信息   
  
# 整体运行流程
redis搭建了3个哨兵，分别为26379、26380、26381端口，master节点为6379，两个slave分别为6380和6381  
需要注意的是哨兵和redis节点的bind都不能绑定127.0.0.1，因为客户端通过哨兵获取的主、从节点host会变成127.0.0.1，然后客户端连接不上127.0.0.1  

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
* 初始化predis项目
* 判断是否已经有连接过的redis接点，获取命令是可读还是可写
  1. 如果节点是master则直接使用master节点去读写 
  2. 如果节点是slave，命令是读命令则直接使用slave节点
  3. 如果节点是slave，命令是写命令则通过哨兵获取master节点
- 如果没有连接的redis节点，客户端连接配置列表中的第一个哨兵，并将这个哨兵从配置列表中移除
  1. 需要获取master节点，向哨兵发送sentinel get-master-addr-by-name $service
  2. 需要获取slave节点，向哨兵发送sentinel slaves $service，过滤掉slave的状态为s_down、o_down、disconnected的节点
- 如果连接哨兵异常，则使用配置列表的第一个哨兵继续连接，并将这个哨兵从配置列表中移除
- 向redis节点发送命令
- 如果redis节点异常，则默认停止1秒，继续连接哨兵获取节点，然后使用获取的节点发送命令，重试20秒

