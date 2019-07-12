## 目录
* [Model类结构](#Model类架构)
* [创建Model](#创建Model)
* [属性](#属性)
* [属性标签与属性hint](#属性标签与属性hint)
* [场景](#场景)
* [验证](#验证)
* [不注册直接从认证组件获取用户信息源码](#不注册直接从认证组件获取用户信息源码)

# Model类架构
Model类源码在verdor/yiisoft/yii2/base/Model.php，该类和传统框架的模型层不一样，比如CI3的模型层Model是专门和数据库交互的，而Yii2的Model是用来处理业务数据、逻辑、验证、获取验证错误信息的  
单纯的Model类无法和数据库交互，需要配合活动记录ActiveRecord  
```
class Model extends Component implements StaticInstanceInterface, IteratorAggregate, ArrayAccess, Arrayable
{
    use ArrayableTrait;   //多继承trait
    use StaticInstanceTrait;  //多继承trait
    const SCENARIO_DEFAULT = 'default';  //默认场景default
    const EVENT_BEFORE_VALIDATE = 'beforeValidate';  //开始验证事件
    const EVENT_AFTER_VALIDATE = 'afterValidate';  //结束验证事件
    private $_errors;   //验证错误的错误信息
    private $_validators;  //验证类
    private $_scenario = self::SCENARIO_DEFAULT;  //场景
```
Model可以使用类迭代器Iterator、数组式访问，还实现yii底层封装的Arrayable接口
```

```
# 创建Model
可以在控制器里面直接new一个Model
```
use app\models\User;
class TestController extends Controller{
    public function actionModel(){
        $user = new User();
    }
}
```
也可以用静态方法实例化
```
use app\models\User;
class TestController extends Controller{
    public function actionModel(){
        $user1 = User:instance();
        $user2 = User:instance(true);
    }
}
```
因为base\Model使用了StaticInstanceTrait，StaticInstanceTrait是一个trait，里面只有一个静态方法instance  
参数refresh为false可以防止多次调用造成的多次实例化，同时true可以强制重新实例化
```
trait StaticInstanceTrait
{
    private static $_instances = [];
    public static function instance($refresh = false)
    {
        $className = get_called_class();
        if ($refresh || !isset(self::$_instances[$className])) {
            self::$_instances[$className] = Yii::createObject($className);
        }
        return self::$_instances[$className];
    }
}
```
由于是trait，所以可以在自己的Model里面重写instance方法  
```
class User extends Model
{
    public static function instance(){
        echo "instance rewrite";
    }
}
```
可以使用formName方法来获取去除命名空间的类名，formName方法是base\Model的底层方法，在需要记录日志的情况下非常有用  
```
public function formName()
{
    //反射本类
    $reflector = new ReflectionClass($this);
    //php版本在7.0.0以上并且不是匿名类
    if (PHP_VERSION_ID >= 70000 && $reflector->isAnonymous()) {
        throw new InvalidConfigException('The "formName()" method should be explicitly defined for anonymous models');
    }
    //获取shortName
    return $reflector->getShortName();
}
```
# 属性
模型通过属性来代表业务数据，属性在Model里面需要定义成非静态共有方法  
涉及属性的方法操作非常多，而且还有一个trait ArrayableTrait来多继承增加操作属性的方法    
```
public function attributes()   //获取全部所有非静态共有属性
public function attributeLabels()  //获取全部属性标签
public function attributeHints()  //获取全部属性hint
public function isAttributeRequired($attribute)  //判断$attribute是否是必填
public function safeAttributes()  //获取该场景下的所有safe属性，safe属性就是rule中定义的非！规则属性
public function isAttributeActive($attribute)  //判断$attribute是否在该场景下有校验规则
public function getAttributeLabel($attribute) //获取$attribute的属性标签
public function generateAttributeLabel($name)  //获取$name的generate属性标签，就是mb_ucwords($name)
public function getAttributes($names = null, $except = [])  //获取属性，并且排除except数组里面的属性
public function setAttributes($values, $safeOnly = true)  //设置属性值，如果safeOnly为true则只能设置在该场景下rule中的属性
public function fields()  //获取属性数组
public function getIterator()  //属性迭代器，为IteratorAggregate接口需要继承的方法
```
