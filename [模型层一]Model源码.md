## 目录
* [Model类结构](#Model类架构)
* [创建Model](#创建Model)
* [属性](#属性)
* [属性标签](#属性标签)
* [场景](#场景)
* [验证](#验证)
* [不注册直接从认证组件获取用户信息源码](#不注册直接从认证组件获取用户信息源码)

# Model类架构
Model类源码在verdor/yiisoft/yii2/base/Model.php，该类和传统框架的模型层不一样，比如CI3的模型层Model是专门和数据库交互的，而Yii2的Model是用来处理业务数据、逻辑和验证的  
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
