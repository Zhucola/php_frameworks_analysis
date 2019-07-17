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
