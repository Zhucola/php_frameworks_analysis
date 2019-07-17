## 目录
* [PHP的多继承概述](#PHP的多继承概述)
* [Behavior行为](#Behavior行为)

# PHP的多继承
我们知道PHP是不支持多继承的，一个子类只能继承一个父类
```
class A{}
class B extends A{}
```
但是我们可以使用traits来模拟多继承，很多框架内部都是涉及到了triats,比如yii2的底层base\Model类
```
class Model extends Component implements StaticInstanceInterface, IteratorAggregate, ArrayAccess, Arrayable
{
    use ArrayableTrait;
    use StaticInstanceTrait;
```
traits就是一个类可以附加多个traits，模拟了多继承，其实traits有很多事项需要注意，比如trait和class都有相同的属性需要特殊处理等等，详细的知识点可以参考[PHP手册](https://www.php.net/manual/en/language.oop5.traits.php)  
```
<?php
trait t{
	public $name = 1;
	public function sing(){
		echo 2;
	}
}
class a{
	use t;
	public function test(){
		echo $this->name;
	}

	public function sing(){
		echo 3;
	}
}
$a = new a;
$a->test(); //1
$a->sing(); //3
```
traits不能再继承了，一旦注册进类就无法移除，所以yii2的行为更加灵活
# Behavior行为
在项目目录创建behavior目录，创建Test.php文件
```
<?php
namespace app\behaviors;

use yii\base\Behavior;

class Test extends Behavior
{
    public $name = 1;
    protected $age = 2;

    public function events()
    {
        return [
        	"gogo" => "eventsgogo",
        ];
    }

    public function sing(){
	return __FUNCTION__;
    }

    public function getage(){
	return $this->age;
    }

    public function eventsgogo($event){
	echo "events";
    }
}
```
然后在控制器中将行为注册进去
```
use app\behaviors\Test;

class TestController extends Controller{
	public function behaviors(){
		return [
		    'myBehavior4' => [
			'class' => Test::className(),
			"name" => 21313131,
		    ]
		];
    	}
	
	public function actionX(){
		$this->name;   //行为的属性
		$this->sing(); //行为的方法
		$this->age; //行为的属性
	}
}
```
因为控制器Controller继承于Component，Component是实现行为的底层架构，使用的就是魔术方法，当调用一个不属于本类的方法会被魔术get捕获到  
```
//如果调用$this->age;

public function __get($name)
{
    //会判断本类是否有getage方法，大小写不冲突    
    $getter = 'get' . $name;
    if (method_exists($this, $getter)) {
        // read property, e.g. getName()
        return $this->$getter();
    }

    //开始去找行为的age属性
    $this->ensureBehaviors();
    foreach ($this->_behaviors as $behavior) {
        if ($behavior->canGetProperty($name)) {
            return $behavior->$name;
        }
    }
    //如果找不到并且本类有setage方法，报错
    if (method_exists($this, 'set' . $name)) {
        throw new InvalidCallException('Getting write-only property: ' . get_class($this) . '::' . $name);
    }
    //找不到，将异常抛出去
    throw new UnknownPropertyException('Getting unknown property: ' . get_class($this) . '::' . $name);
}
```
创建行为的方法如下，需要注意的是，如果在控制器中调用，这里的$this是控制器的
```
public function ensureBehaviors()
{
    if ($this->_behaviors === null) {
        //初始化数组
        $this->_behaviors = [];
	//遍历behaviors
        foreach ($this->behaviors() as $name => $behavior) {
	    //注册行为
            $this->attachBehaviorInternal($name, $behavior);
        }
    }
}
```
默认的底层behaviors是返回一个空数组，所以我们需要重写这个方法
```
public function behaviors()
{
    return [];
}
```
注册的方法如下
```
private function attachBehaviorInternal($name, $behavior)
{
    if (!($behavior instanceof Behavior)) {
        //实例化行为对象
        $behavior = Yii::createObject($behavior);
    }
    if (is_int($name)) {
        $behavior->attach($this);
	//追加行为
        $this->_behaviors[] = $behavior;
    } else {
        //判断是否应该覆盖已注册的行为
        if (isset($this->_behaviors[$name])) {
            $this->_behaviors[$name]->detach();
        }
        $behavior->attach($this);
        $this->_behaviors[$name] = $behavior;
    }

    return $behavior;
}
```
attach和detach方法涉及到了yii2的事件概念，这里先不展开讨论，可以先理解为将行为中的事件注册或者移除
```
public function attach($owner)
{
    $this->owner = $owner;
    foreach ($this->events() as $event => $handler) {
        $owner->on($event, is_string($handler) ? [$this, $handler] : $handler);
    }
}
public function detach()
{
    if ($this->owner) {
        foreach ($this->events() as $event => $handler) {
            $this->owner->off($event, is_string($handler) ? [$this, $handler] : $handler);
        }
        $this->owner = null;
    }
}
```
这里需要注意，行为底层有一个共有方法owner，不要在注册行为的类中声明owner属性，这样行为的owner会被覆盖
```
class Behavior extends BaseObject{
	public $owner;
```
魔术get还调用了canGetProperty方法，这个方法是yii属性注入的核心方法，在base\BaseObj.php里面
```
public function canGetProperty($name, $checkVars = true)
{
    return method_exists($this, 'get' . $name) || $checkVars && property_exists($this, $name);
}
```
所以如果行为类里面有非公有非静态属性，可以声明get前缀的方法
```
class Obj extends Behavior{
	private $incr = 0;
	public function getIncr(){
		return $sex++;
	}
}
```
除了可以覆盖behaviors方法来声明行为，还可以调用attachBehavior和attachBehaviors来注册行为
```
public function attachBehavior($name, $behavior)
{
    $this->ensureBehaviors();
    //注册行为
    return $this->attachBehaviorInternal($name, $behavior);
}
public function attachBehaviors($behaviors)
{
    $this->ensureBehaviors();
    foreach ($behaviors as $name => $behavior) {
        //注册行为
        $this->attachBehaviorInternal($name, $behavior);
    }
}
```
如果需要移除行为，可以使用detachBehavior或者detachBehaviors方法
```
public function detachBehavior($name)
{
    $this->ensureBehaviors();
    if (isset($this->_behaviors[$name])) {
        $behavior = $this->_behaviors[$name];
        unset($this->_behaviors[$name]);
        $behavior->detach();
        return $behavior;
    }

    return null;
}
public function detachBehaviors()
{
    $this->ensureBehaviors();
    foreach ($this->_behaviors as $name => $behavior) {
    	$this->detachBehavior($name);
    }
}
```
获取行为可以使用getBehavior或者getBehaviors方法，因为有属性注入，所以直接调用behaviors属性也是可以的
```
public function getBehavior($name)
{
    $this->ensureBehaviors();
    return isset($this->_behaviors[$name]) ? $this->_behaviors[$name] : null;
}
public function getBehaviors()
{
    $this->ensureBehaviors();
    return $this->_behaviors;
}
```
