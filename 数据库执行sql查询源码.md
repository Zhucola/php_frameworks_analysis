## 目录
* [从最简单的查询说起](#从最简单的查询说起)

# 从最简单的查询说起
控制器代码如下
```
class TestController extends Controller
{
	public function actionD(){
		$db = new \yii\db\Connection([
		    'dsn' => 'mysql:host=192.168.124.10;dbname=test',
		    'username' => 'root',
		    'password' => '',
		    'charset' => 'utf8',
		    
		]);
		$command = $db->createCommand("select * from a");
		$command->nocache();
		$command->cache(600);
		$res1 = $command->queryAll();
		$res2 = $command->queryOne();
		$res3 = $command->queryColumn();
		return $this->asJson($res1);
	}
}
```
可见简单的查询都要通过createCommand()返回的对象来进行操作，createCommand()就是实例化了yii2\db\Command类
```
public function createCommand($sql = null, $params = [])
{
    //获取数据库的驱动类型
    $driver = $this->getDriverName();
    $config = ['class' => 'yii\db\Command'];
    if ($this->commandClass !== $config['class']) {
        $config['class'] = $this->commandClass;
    } elseif (isset($this->commandMap[$driver])) {
        //可以任意扩展最后实例化的Command类，就是通过将$this->commandMap[$driver]设置为自定义的数组类型
        $config = !is_array($this->commandMap[$driver]) ? ['class' => $this->commandMap[$driver]] : $this->commandMap[$driver];
    }
    $config['db'] = $this;
    $config['sql'] = $sql;
    //实例化
    $command = Yii::createObject($config);
    return $command->bindValues($params);
}
```
获取数据库的驱动类型就是通过$dsn或者$masters、$masterConfig、$slaves、$slaveConfig配置来获取，比如  
```
  $dsn = "mysql:host=127.0.0.1;dbname=test"
```
数据库驱动类型就是mysql，这里涉及到了连接数据库直接从pdo对象获取类型
```
public function getDriverName()
{
    if ($this->_driverName === null) {
        if (($pos = strpos($this->dsn, ':')) !== false) {
            $this->_driverName = strtolower(substr($this->dsn, 0, $pos));
        } else {
            //如果dsn属性没有':'，就去连接slave
            $this->_driverName = strtolower($this->getSlavePdo()->getAttribute(PDO::ATTR_DRIVER_NAME));
        }
    }

    return $this->_driverName;
}
```
Command类就是根据sql命令来实例化的，继承于Component，说明可以使用属性注入、方法和事件  
因为Command的$this->_ sql属性是private，所以走了属性注入setSql()方法
```
public function setSql($sql)
{
    if ($sql !== $this->_sql) {
        $this->cancel();
        $this->reset();
        $this->_sql = $this->db->quoteSql($sql); 
    }
    return $this;
}
```
cancel方法就是重置pdo的prepare操作
```
public function cancel()
{
    $this->pdoStatement = null;
}
```
reset方法就是重置与Command类相关的属性，这里需要注意也会将$this->_ retryHandler重置，所以如果使用异常重试机制，需要在createCommand后再设置一遍retryHandler
```
protected function reset()
{
    $this->_sql = null;
    $this->_pendingParams = [];
    $this->params = [];
    $this->_refreshTableName = null;
    $this->_isolationLevel = false;
    $this->_retryHandler = null;
}
```
重新注册异常重试(控制器部分代码)
```
$command = $db->createCommand("select * from a");
$command->setRetryHandler(function(){
  echo "语句执行发生了异常";
});
```
实例化最后Command类后还要调用Command->bindValues()，因为最简单的select没有使用参数绑定(实参$values为空数据)，所以这个方法直接返回$this
```
public function bindValues($values)
{
    if (empty($values)) {
        return $this;
    }
```
然后执行queryAll、queryOne、queryColumn进行查询，可见其实调用的都是queryInternal
```
public function queryAll($fetchMode = null)
{
    return $this->queryInternal('fetchAll', $fetchMode);
}
public function queryOne($fetchMode = null)
{
    return $this->queryInternal('fetch', $fetchMode);
}
public function queryColumn()
{
    return $this->queryInternal('fetchAll', \PDO::FETCH_COLUMN);
}
```
queryInternal方法如下
```
protected function queryInternal($method, $fetchMode = null)
{
    //拿到真实的参数绑定后的sql语句$rawSql，拿到是否使用性能分析标识$profile
    list($profile, $rawSql) = $this->logQuery('yii\db\Command::query');
    if ($method !== '') {
    	//这里就是去Connection类拿到QueryCache
        $info = $this->db->getQueryCacheInfo($this->queryCacheDuration, $this->queryCacheDependency);
        if (is_array($info)) {
            //cache对象
            $cache = $info[0];
            $rawSql = $rawSql ?: $this->getRawSql();
	    //生成QueryCache键名
            $cacheKey = $this->getCacheKey($method, $fetchMode, $rawSql);
	    //get缓存操作
            $result = $cache->get($cacheKey);
            if (is_array($result) && isset($result[0])) {
                Yii::debug('Query result served from cache', 'yii\db\Command::query');
                return $result[0];
            }
        }
    }
    //就是拿到$pdo->prepare()并且进行参数绑定，会根据sql类型进行判断是用slave的还是master的
    $this->prepare(true);

    try {
    	//开启性能分析日志
        $profile and Yii::beginProfile($rawSql, 'yii\db\Command::query');
	//这里就是真正执行sql了
        $this->internalExecute($rawSql);
	//这里不太清楚，不用太抠这些细节
        if ($method === '') {
            $result = new DataReader($this);
        } else {
            if ($fetchMode === null) {
                $fetchMode = $this->fetchMode;
            }
            $result = call_user_func_array([$this->pdoStatement, $method], (array) $fetchMode);
            $this->pdoStatement->closeCursor();
        }

        $profile and Yii::endProfile($rawSql, 'yii\db\Command::query');
    } catch (Exception $e) {
        $profile and Yii::endProfile($rawSql, 'yii\db\Command::query');
        throw $e;
    }
    //存缓存
    if (isset($cache, $cacheKey, $info)) {
        $cache->set($cacheKey, [$result], $info[1], $info[2]);
        Yii::debug('Saved query result in cache', 'yii\db\Command::query');
    }

    return $result;
}
```
prepare方法如下
```
public function prepare($forRead = null)
{
    if ($this->pdoStatement) {
        $this->bindPendingParams();
        return;
    }

    $sql = $this->getSql();

    if ($this->db->getTransaction()) {
        // master is in a transaction. use the same connection.
        $forRead = false;
    }
    //这里就是根据sql类型来判断是连接slave还是master，从读主写
    if ($forRead || $forRead === null && $this->db->getSchema()->isReadQuery($sql)) {
        $pdo = $this->db->getSlavePdo();
    } else {
        $pdo = $this->db->getMasterPdo();
    }

    try {
        $this->pdoStatement = $pdo->prepare($sql);
	//参数绑定
        $this->bindPendingParams();
    } catch (\Exception $e) {
        $message = $e->getMessage() . "\nFailed to prepare SQL: $sql";
        $errorInfo = $e instanceof \PDOException ? $e->errorInfo : null;
        throw new Exception($message, $errorInfo, (int) $e->getCode(), $e);
    }
}
```
如果不能根据参数来判断是读类型sql还是写类型sql，就是去isReadQuery()，可见select|show|describe都是读操作
```
public function isReadQuery($sql)
{
    $pattern = '/^\s*(SELECT|SHOW|DESCRIBE)\b/i';
    return preg_match($pattern, $sql) > 0;
}
```
internalExecute代码如下
```
protected function internalExecute($rawSql)
{
    $attempt = 0;
    while (true) {
        try {
	    //这里会进行模拟多层级事务，这块会放到事务部分去讲解
            if (
                ++$attempt === 1
                && $this->_isolationLevel !== false
                && $this->db->getTransaction() === null
            ) {
                $this->db->transaction(function () use ($rawSql) {
                    $this->internalExecute($rawSql);
                }, $this->_isolationLevel);
            } else {
	    	//就是$pdo->execute()
                $this->pdoStatement->execute();
            }
            break;
        } catch (\Exception $e) {
            $rawSql = $rawSql ?: $this->getRawSql();
	    //这里返回了一个\yii\db\Exception
            $e = $this->db->getSchema()->convertException($e, $rawSql);
	    //执行异常重试
            if ($this->_retryHandler === null || !call_user_func($this->_retryHandler, $e, $attempt)) {
                throw $e;
            }
        }
    }
}
```
