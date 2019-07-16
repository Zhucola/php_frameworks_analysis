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
