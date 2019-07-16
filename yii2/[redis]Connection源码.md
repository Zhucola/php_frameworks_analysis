yii2安装后不会有yii-redis组件，需要手动安装该组件  
```
composer require yiisoft/yii2-redis
```
然后在组件配置中添加redis配件
```
'redis' => [
    'class' => 'yii\redis\Connection',
    'hostname' => '192.168.124.10',
    'port' => 6379,
    'database' => 0,
],
```
然后就可以在控制器里面使用redis了
```
public function actionRedis(){
    $redis = Yii::$app->get("redis");
    $redis->set("g",1);
    return $redis->get("g");
}
```
yii/redis/Connection是redis的基础组件，底层的会话、Mutex、Cache都是依赖这个组件  
没有redis扩展也可以使用，因为底层使用的是stream_socket_client连接redis客户端  
```
public function open()
{
    if ($this->_socket !== false) {
        return;
    }
    $connection = ($this->unixSocket ?: $this->hostname . ':' . $this->port) . ', database=' . $this->database;
    //记录日志
    \Yii::trace('Opening redis DB connection: ' . $connection, __METHOD__);
    //连接客户端
    $this->_socket = @stream_socket_client(
        $this->unixSocket ? 'unix://' . $this->unixSocket : 'tcp://' . $this->hostname . ':' . $this->port,
        //连接失败的错误号
        $errorNumber,
        //连接失败的错误信息
        $errorDescription,
        //连接超时时间
        $this->connectionTimeout ? $this->connectionTimeout : ini_get('default_socket_timeout'),
        $this->socketClientFlags
    );
    if ($this->_socket) {
        if ($this->dataTimeout !== null) {
            //读、写操作的超时时间
            stream_set_timeout($this->_socket, $timeout = (int) $this->dataTimeout, (int) (($this->dataTimeout - $timeout) * 1000000));
        }
        if ($this->password !== null) {
            //密码
            $this->executeCommand('AUTH', [$this->password]);
        }
        if ($this->database !== null) {
            //redis数据库
            $this->executeCommand('SELECT', [$this->database]);
        }
        //连接成功的事件
        $this->initConnection();
    } else {
        \Yii::error("Failed to open redis DB connection ($connection): $errorNumber - $errorDescription", __CLASS__);
        $message = YII_DEBUG ? "Failed to open redis DB connection ($connection): $errorNumber - $errorDescription" : 'Failed to open DB connection.';
        throw new Exception($message, $errorDescription, $errorNumber);
    }
}
protected function initConnection()
{
    //连接redis成功的事件
    $this->trigger(self::EVENT_AFTER_OPEN);
}
```
如果要使用命令，那么会被__call魔术方法执行  
```
 public function __call($name, $params)
{
    //格式化命令，可以理解为mb_ucwords
    $redisCommand = strtoupper(Inflector::camel2words($name, false));
    //判断命令是否可用
    if (in_array($redisCommand, $this->redisCommands)) {
        //命令可用，执行
        return $this->executeCommand($redisCommand, $params);
    } else {
        //命令不可用，调用父类的魔术方法
        return parent::__call($name, $params);
    }
}   
```
给redis服务发命令的代码十分有意思
```
public function executeCommand($name, $params = [])
{
    $this->open();
    
    //这块是非常有意思的代码
    $params = array_merge(explode(' ', $name), $params);
    $command = '';
    $paramsCount = 0;
    foreach ($params as $arg) {
        if ($arg === null) {
            continue;
        }

        $paramsCount++;
        $command .= '$' . mb_strlen($arg, '8bit') . "\r\n" . $arg . "\r\n";
    }
    $command = '*' . $paramsCount . "\r\n" . $command;
    \Yii::trace("Executing Redis Command: {$name}", __METHOD__);
    //重发机制
    if ($this->retries > 0) {
        $tries = $this->retries;
        while ($tries-- > 0) {
            try {
                //成功了直接return
                return $this->sendCommandInternal($command, $params);
            } catch (SocketException $e) {
                \Yii::error($e, __METHOD__);
                // backup retries, fail on commands that fail inside here
                $retries = $this->retries;
                $this->retries = 0;
                //关闭连接
                $this->close();
                //重新建立连接
                $this->open();
                $this->retries = $retries;
            }
        }
    }
    //非重发机制
    return $this->sendCommandInternal($command, $params);
}
```
核心就是executeCommand方法的那个foreach，如果给redis发一个hmset a a 1 b 2 c 3，php-redis是这样的
```
$redis=new Redis();
$redis->connect(...);
$redis->hmset("a",["a"=>1,"b"=>2,"c"=>3]);
```
但是yii的redis是
```
$redis = Yii::$app->get("redis");
$redis->hmset("a","a",1,"b",2,"c",3);
```
给redis发过去的是这样的格式
```
*8   //代表有8个数据组组成，就是HMSET a a 1 b 2 c 3，一共8个
$5   //代表HMSET一共5个字节
HMSET
$1
a
$1
a
$1
1
$1
b
$1
2
$1
c
$1
3

```
最后将数据发给redis服务
```
private function sendCommandInternal($command, $params)
{
    $written = @fwrite($this->_socket, $command);
    if ($written === false) {
        throw new SocketException("Failed to write to socket.\nRedis command was: " . $command);
    }
    if ($written !== ($len = mb_strlen($command, '8bit'))) {
        throw new SocketException("Failed to write to socket. $written of $len bytes written.\nRedis command was: " . $command);
    }
    return $this->parseResponse(implode(' ', $params));
}
```
然后需要得到redis服务的响应
```
private function parseResponse($command)
{
    if (($line = fgets($this->_socket)) === false) {
        throw new SocketException("Failed to read from socket.\nRedis command was: " . $command);
    }
    $type = $line[0];
    $line = mb_substr($line, 1, -2, '8bit');
    switch ($type) {
        case '+': // Status reply
            if ($line === 'OK' || $line === 'PONG') {
                return true;
            } else {
                return $line;
            }
        case '-': // Error reply
            throw new Exception("Redis error: " . $line . "\nRedis command was: " . $command);
        case ':': // Integer reply
            // no cast to int as it is in the range of a signed 64 bit integer
            return $line;
        case '$': // Bulk replies
            if ($line == '-1') {
                return null;
            }
            $length = (int)$line + 2;
            $data = '';
            while ($length > 0) {
                if (($block = fread($this->_socket, $length)) === false) {
                    throw new SocketException("Failed to read from socket.\nRedis command was: " . $command);
                }
                $data .= $block;
                $length -= mb_strlen($block, '8bit');
            }

            return mb_substr($data, 0, -2, '8bit');
        case '*': // Multi-bulk replies
            $count = (int) $line;
            $data = [];
            for ($i = 0; $i < $count; $i++) {
                $data[] = $this->parseResponse($command);
            }

            return $data;
        default:
            throw new Exception('Received illegal data from redis: ' . $line . "\nRedis command was: " . $command);
    }
}
```
简单来说，给redis发一个hmset命令，redis会返回
```
+OK

```
然后在判断返回的信息是否正确
