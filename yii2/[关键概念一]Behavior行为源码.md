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
traits就是一个类可以附加多个traits，模拟了多继承，其实traits有很多事项需要注意，比如trait和class都有相同的属性需要特殊处理等等，详细的知识点可以参考PHP手册  
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
