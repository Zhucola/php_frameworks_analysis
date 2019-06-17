先说从库连接的总体流程，再说细节
1.随机打乱从库配置
2.遍历从库配置
	(a)如果该从库的配置在serverStatusCache缓存中生效则说明过期时间内该配置不可用，直接continue
	(b)如果缓存无值则去实例化PDO
3.实例化PDO
	(a)记录info日志
	(b)记录实例化性能分析日志
	(c)new PDO
	(d)设置PDO的ATTR_ERRMODE、ATTR_EMULATE_PREPARES、字符集属性
	(e)执行afterOpen事件
4.实例化失败
	(a)记录serverStatusCache缓存，标识该配置600秒(默认)内不可用
	
	




yii2中可以配置一主多从配置，在连接从库方面数据库配置如下(控制器中连接配置)
```
<?php
namespace app\controllers;
use Yii;
use yii\web\Controller;
use PDO;
class TestController extends Controller
{
	public function actionD(){
		$db = new \yii\db\Connection([
		    'dsn' => 'mysql:host=192.168.124.10;dbname=test',
		    'username' => 'root',
		    'password' => '',
		    'charset' => 'utf8',
		    'enableSlaves'=>true, //可以使用从库
		    'serverRetryInterval'=>600, //其中一个从库配置不可用，将缓存不可用状态600秒
		    'enableProfiling'=>true, //默认配置，将记录连接数据库、执行语句等的性能分析日志
		    'emulatePrepare'=>true,  //true为开启本地模拟prepare
		    'slaveConfig'=>[ //从库slaves属性通用配置
		    	'username' => 'root',
		    	'password' => '',
		    	'attributes' => [
            		PDO::ATTR_TIMEOUT => 10,
        		],
		    ],
		    'slaves'=>[  //从库列表
		    	["dsn"=>"mysql:host=192.168.124.11;dbname=test"],
		    	["dsn"=>"mysql:host=192.168.124.12;dbname=test"],
		    	[
		    		"dsn"=>"mysql:host=192.168.124.13;dbname=test",
		    		'username' => 'main',
		    		'password' => '123456',
		    	],
		    ]
		]);
		$slave = $db->getSlavePdo();
		$slave = $db->getSlave();
		return 123;
	}
}
```
可以看到数据库操作的类是\yii\db\Connection，该类继承Component类，可见可以使用属性注入、行为和事件

针对Connection的属性注入，只有以下属性是私有的，以下属性一般不会在外部进行操作
```
private $_transaction;
private $_schema;
private $_driverName;
private $_master = false;
private $_slave = false;
private $_queryCacheInfo = [];
```

针对Connection的事件，可以注册以下事件
```
const EVENT_AFTER_OPEN = 'afterOpen';   //连接数据库后的事件
const EVENT_BEGIN_TRANSACTION = 'beginTransaction';  //开启事务的事件
const EVENT_COMMIT_TRANSACTION = 'commitTransaction';  //提交事务的事件
const EVENT_ROLLBACK_TRANSACTION = 'rollbackTransaction';  //回滚的事件
```

Connection类使用的mysql操作对象是PDO，涉及方法有
```
public function getSlavePdo($fallbackToMaster = true)
public function getSlave($fallbackToMaster = true)
```
追进在getSlavePdo方法，可见当slave连接不可用时候，会默认连接主库($fallbackToMaster=true)
```
public function getSlavePdo($fallbackToMaster = true)
{
	$db = $this->getSlave(false);  //进行slave连接
	if ($db === null) { 
    		return $fallbackToMaster ? $this->getMasterPdo() : null;  //当slave不可用时候，是否连接主库
	}

	return $db->pdo;  //返回数据库连接资源，从库和主库都连接不上的话会返回null
}
```
追进getSlave方法
```
public function getSlave($fallbackToMaster = true)
{
	if (!$this->enableSlaves) {//判断是否可以使用slave
	    return $fallbackToMaster ? $this : null; 
	}

	if ($this->_slave === false) {  //如果还没有连接过slave库，就进行连接
	    $this->_slave = $this->openFromPool($this->slaves, $this->slaveConfig);  //将slave配置信息给openFromPool方法
	}
	return $this->_slave === null && $fallbackToMaster ? $this : $this->_slave;
}
```
追进openFromPool方法，可见该方法就是将$this->slaves从库dsn配置打乱，让第一次连接slave随机化
```
protected function openFromPool(array $pool, array $sharedConfig)
{
	shuffle($pool); //打乱从库配置
	return $this->openFromPoolSequentially($pool, $sharedConfig);
}
```
openFromPoolSequentially方法
```
protected function openFromPoolSequentially(array $pool, array $sharedConfig)
{
	if (empty($pool)) {  //是否有slave配置池，如果没有的话就是最后返回给$this->_slave为null
	    return null;
	}

	if (!isset($sharedConfig['class'])) {  //判断$this->slaveConfig属性是否有class，可以设置class将从库的连接配置成自己重新的类
	    $sharedConfig['class'] = get_class($this);
	}
	//服务状态缓存，使用依赖注入获取cache缓存类
	$cache = is_string($this->serverStatusCache) ? Yii::$app->get($this->serverStatusCache, false) : $this->serverStatusCache;
	//遍历slave配置池
	foreach ($pool as $config) {
	    //合并配置
	    $config = array_merge($sharedConfig, $config);
	    if (empty($config['dsn'])) {
		throw new InvalidConfigException('The "dsn" option must be specified.');
	    }
	    $key = [__METHOD__, $config['dsn']];
	    //这里就是判断缓存是否有值，如果有的话说明在过期时间内该配置的slave不可用
	    if ($cache instanceof CacheInterface && $cache->get($key)) {
		// should not try this dead server now
		continue;
	    }
	    //通过依赖注入创建了一个类，该类专门是这个slave的
	    $db = Yii::createObject($config);
	    try {
		$db->open();
		return $db;
	    } catch (\Exception $e) {
	    	//记录日志
		Yii::warning("Connection ({$config['dsn']}) failed: " . $e->getMessage(), __METHOD__);
		if ($cache instanceof CacheInterface) {
		    //将该配置的slave服务不可用状态存缓存，值是1，过期时间的$this->serverRetryInterval秒
		    $cache->set($key, 1, $this->serverRetryInterval);
		}
	    }
	}
	return null;
}
```
在TestController控制器的配置中，可见会随机打乱slaves属性，如果有任何一个从库连接上了就是直接返回，如果有连接不上的就会将不可用状态存缓存，然后继续循环

slaveConfig属性是一个从库的通用配置，会循环的去array_merge()属性slaves

所以配置
```
'slaveConfig'=>[ //从库slaves属性通用配置
	'username' => 'root',
	'password' => '',
	'attributes' => [
		PDO::ATTR_TIMEOUT => 10,
	],
],
'slaves'=>[  //从库列表
	["dsn"=>"mysql:host=192.168.124.11;dbname=test"],
	["dsn"=>"mysql:host=192.168.124.12;dbname=test"],
	[
		"dsn"=>"mysql:host=192.168.124.13;dbname=test",
		'username' => 'main',
		'password' => '123456',
		'class'=> yii\overload\myDB
	],
]
```
最后生成的配置为(这个配置会被shuffle函数打乱顺序)
```
'slaves'=>[  //从库列表
	[
		"dsn"=>"mysql:host=192.168.124.11;dbname=test",
		'username' => 'root',
		'password' => '',
		'attributes' => [
			PDO::ATTR_TIMEOUT => 10,
		],
		'class' => 'yii\db\Connection',
	],
	[
		"dsn"=>"mysql:host=192.168.124.12;dbname=test"
		'username' => 'root',
		'password' => '',
		'attributes' => [
			PDO::ATTR_TIMEOUT => 10,
		],
		'class' => 'yii\db\Connection',
	],
	[
		"dsn"=>"mysql:host=192.168.124.13;dbname=test",
		'username' => 'main',
		'password' => '123456',
		'class'=> yii\overload\myDB
	],
]
```
在open方法中，因为是重新new，所以$this->pdo和$this->master都是null
```
public function open()
{
	//因为是重新new，所以$this->pdo和$this->master都是null
	if ($this->pdo !== null) {
	    return;
	}
	//因为是重新new，所以$this->pdo和$this->master都是null
	if (!empty($this->masters)) {
	    $db = $this->getMaster();
	    if ($db !== null) {
		$this->pdo = $db->pdo;
		return;
	    }

	    throw new InvalidConfigException('None of the master DB servers is available.');
	}
	
	if (empty($this->dsn)) {
	    throw new InvalidConfigException('Connection::dsn cannot be empty.');
	}

	$token = 'Opening DB connection: ' . $this->dsn;
	$enableProfiling = $this->enableProfiling;
	try {
	    //记录日志
	    Yii::info($token, __METHOD__);
	    //如果开启了性能分析，则记录性能分析日志(性能分析开启)
	    if ($enableProfiling) {
		Yii::beginProfile($token, __METHOD__);
	    }
		
	    $this->pdo = $this->createPdoInstance();
	    $this->initConnection();
	    //如果开启了性能分析，则记录性能分析日志(性能分析关闭)
	    if ($enableProfiling) {
		Yii::endProfile($token, __METHOD__);
	    }
	} catch (\PDOException $e) {
	    if ($enableProfiling) {
		Yii::endProfile($token, __METHOD__);
	    }

	    throw new Exception($e->getMessage(), $e->errorInfo, (int) $e->getCode(), $e);
	}
}
```
在createPdoInstance方法中，这个没什么好说的，就是执行new PDO
```
protected function createPdoInstance()
{
	$pdoClass = $this->pdoClass;
	if ($pdoClass === null) {
	    $pdoClass = 'PDO';
	    if ($this->_driverName !== null) {
		$driver = $this->_driverName;
	    } elseif (($pos = strpos($this->dsn, ':')) !== false) {
		$driver = strtolower(substr($this->dsn, 0, $pos));
	    }
	    if (isset($driver)) {
		if ($driver === 'mssql' || $driver === 'dblib') {
		    $pdoClass = 'yii\db\mssql\PDO';
		} elseif ($driver === 'sqlsrv') {
		    $pdoClass = 'yii\db\mssql\SqlsrvPDO';
		}
	    }
	}

	$dsn = $this->dsn;
	if (strncmp('sqlite:@', $dsn, 8) === 0) {
	    $dsn = 'sqlite:' . Yii::getAlias(substr($dsn, 7));
	}

	return new $pdoClass($dsn, $this->username, $this->password, $this->attributes);
}
```
在initConnection方法中，这个也没什么好说的，就是去设置PDO属性和执行afterOpen事件
```
protected function initConnection()
{
    $this->pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    if ($this->emulatePrepare !== null && constant('PDO::ATTR_EMULATE_PREPARES')) {
        $this->pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, $this->emulatePrepare);
    }
    if ($this->charset !== null && in_array($this->getDriverName(), ['pgsql', 'mysql', 'mysqli', 'cubrid'], true)) {
        $this->pdo->exec('SET NAMES ' . $this->pdo->quote($this->charset));
    }
    $this->trigger(self::EVENT_AFTER_OPEN);
}
```
