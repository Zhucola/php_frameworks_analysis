## 目录
* [事务与事务嵌套](#事务与事务嵌套)
* [db异常类](#db异常类)

# 事务与事务嵌套  
  yii有两种事务模式，一种是需要自己commit和捕获异常rollback，还有一种令是使用回调的方式，如控制器代码如下:
```
public function actionTest()
{
    //db组件配置
    $db = new \yii\db\Connection([
	    'dsn' => 'mysql:host=192.168.0.10;dbname=test',
	    'username' => 'root',
	    'password' => '',
	    'charset' => 'utf8',
	    'enableSlaves'=>true, //可以使用从库
	    'serverRetryInterval'=>600, //其中一个从库配置不可用，将缓存不可用状态600秒
	    'enableProfiling'=>true, //默认配置，将记录连接数据库、执行语句等的性能分析日志
	    'emulatePrepare'=>true,  //true为开启本地模拟prepare
	    'shuffleMasters'=>false,
	    'serverStatusCache'=>false,
	    'slaveConfig'=>[ //从库slaves属性通用配置
	    	'username' => 'root',
	    	'password' => '',
	    	'attributes' => [
        		PDO::ATTR_TIMEOUT => 1,
    		],
	    ],
	    'slaves'=>[  //从库列表
	    	["dsn"=>"mysql:host=192.168.0.10;dbname=test"],
	    ],
	    'masters'=>[  //主库列表
	    	["dsn"=>"mysql:host=192.168.0.10;dbname=test"],
	    ],
	    'masterConfig'=>[ //主库master属性通用配置
	    	'username' => 'root',
	    	'password' => '',
	    	'attributes' => [
        		PDO::ATTR_TIMEOUT => 1,
    		],
	    ],
	]);
  //第一种事务模式，需要自己去commit和捕获异常rollback
	$transaction = $db->beginTransaction();
	try {
		$command1 = $db->createCommand("insert into a([[age]]) value(:age)");
		$res1 = $command1->bindValue(":age",111)->execute();
		$command2 = $db->createCommand("update a set age = 1");
		$res2 = $command2->execute();
		$transaction->commit();
	} catch (\Exception $e) {
		$transaction->rollback();
	}
  //第二种事务模式，不用自己去捕获异常然后去rollback
	$db->transaction(function($db){
		$command1 = $db->createCommand("insert into a([[age]]) value(:age)");
		$res1 = $command1->bindValue(":age",111)->execute();
		$command2 = $db->createCommand("update a set age = 1");
		$res2 = $command2->execute();
	});
	return 123;
}
```
我们先从第一种模式的源码开始看，第一个方法是beginTransaction  
从源码分析可知，事务一定会去连接master，具体master和slave连接源码分析请看  
```
public function beginTransaction($isolationLevel = null)
{
    //事务一定会去连接master
    $this->open();
    //已经开启了事务就不用再去实例化Transaction类了
    if (($transaction = $this->getTransaction()) === null) {
        $transaction = $this->_transaction = new Transaction(['db' => $this]);
    }
    $transaction->begin($isolationLevel);
    //返回Transaction对象
    return $transaction;
}
```
可见事务的核心是一个单独的Transaction类，该类继承与BaseObject所以只有属性注入，无行为behavior和事件Events  
```
class Transaction extends \yii\base\BaseObject
{
    //以下几个常量都是隔离级别
    const READ_UNCOMMITTED = 'READ UNCOMMITTED';
    const READ_COMMITTED = 'READ COMMITTED';
    const REPEATABLE_READ = 'REPEATABLE READ';
    const SERIALIZABLE = 'SERIALIZABLE';
    public $db;
    //事务层级，用来模拟多层级事务
    private $_level = 0;
```
其实yii的事务是有几个Events事件的，只不过事件标识是挂在Connection类的
```
class Connection extends Component
{
    const EVENT_AFTER_OPEN = 'afterOpen';
    //事务开启事件标识
    const EVENT_BEGIN_TRANSACTION = 'beginTransaction';
    //事务提交事件标识
    const EVENT_COMMIT_TRANSACTION = 'commitTransaction';
    //事务回滚事件标识
    const EVENT_ROLLBACK_TRANSACTION = 'rollbackTransaction';
```
回到Transaction类的begin方法  
我们知道mysql是不支持多事务层级的，在第一次begin开启事务之后，再次begin会commit第一次的事务，yii根据事务层级level和savepoint来模拟多事务层级  
单事务源码逻辑:  
1.第一次开启事务，事务层级level是1(自增)    
2.提交或者回滚，事务层级level为0(自减)，会真正的commit或者rollback    
  
  
  
多事务源码逻辑  
1.第一次开启事务，事务层级level是1(自增)    
2.再次开启事务，事务层级level是2(自增)，savepoint名字是LEVEL2  
3.再次开启事务，事务层级level是3(自增)，savepoint名字是LEVEL3  
4.提交，事务层级level是2(自减)，删除名字是LEVEL3的savepint，不会真正的commit  
5.回滚，事务层级level是1(自减)，回滚名字是LEVEL2的savepoint，会真实rollback  
6.提交或者回滚，因为事务层级level是1自减为0，所以表示是最外层事务了，会真正的commit或者rollback    
```
public function begin($isolationLevel = null)
{
    if ($this->db === null) {
        throw new InvalidConfigException('Transaction::db must be set.');
    }
    //连接master
    $this->db->open();
    //如果是第一次开启事务
    if ($this->_level === 0) {
        if ($isolationLevel !== null) {
            //设置事务隔离级别
            $this->db->getSchema()->setTransactionIsolationLevel($isolationLevel);
        }
        //记录debug级别日志
        Yii::debug('Begin transaction' . ($isolationLevel ? ' with isolation level ' . $isolationLevel : ''), __METHOD__);
        //触发开启事件事件
        $this->db->trigger(Connection::EVENT_BEGIN_TRANSACTION);
        //pdo开启事务
        $this->db->pdo->beginTransaction();
        //事务层级为1
        $this->_level = 1;

        return;
    }
    //如果不是第一次开启事务(多层级事务)
    $schema = $this->db->getSchema();
    //判断db组件配置是否支持使用多层级事务
    if ($schema->supportsSavepoint()) {
        //记录debug级别日志
        Yii::debug('Set savepoint ' . $this->_level, __METHOD__);
        //打一个savepoint点，注意savepoint点的名字是和事务层级有关
        $schema->createSavepoint('LEVEL' . $this->_level);
    } else {
        Yii::info('Transaction not started: nested transaction not supported', __METHOD__);
        throw new NotSupportedException('Transaction not started: nested transaction not supported.');
    }
    //多层级事务，自增事务层级
    $this->_level++;
}
```
