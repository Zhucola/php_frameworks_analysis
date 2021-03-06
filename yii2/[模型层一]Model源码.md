## 目录
* [Model类结构](#Model类架构)
* [创建Model](#创建Model)
* [属性](#属性)
* [场景](#场景)
* [验证](#验证)

# Model类架构
Model类源码在verdor/yiisoft/yii2/base/Model.php，该类和传统框架的模型层不一样，比如CI3的模型层Model是专门和数据库交互的，而Yii2的Model是用来处理业务数据、逻辑、验证、获取验证错误信息的  
个人感觉yii2的Model层比较难理解，注释中作者写道  
```
Model is the base class for data models
You may directly use Model to store model data, or extend it with customization
```
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
attributes方法，获取全部所有非静态共有属性  
其实php的get_class_vars方法也是获取全部共有属性，但是get_class_vars也获取的静态共有属性  
```
public function attributes()
{
    //反射该类
    $class = new ReflectionClass($this);
    $names = [];
    //获取全部共有属性
    foreach ($class->getProperties(\ReflectionProperty::IS_PUBLIC) as $property) {
        if (!$property->isStatic()) {
            //排除静态属性
            $names[] = $property->getName();
        }
    }

    return $names;
}
```
getAttributes方法感觉比较鸡肋，源码比较简单  
```
public function getAttributes($names = null, $except = [])
{
    $values = [];
    if ($names === null) {
        $names = $this->attributes();  //获取全部非静态共有属性
    }
    foreach ($names as $name) {
        $values[$name] = $this->$name;
    }
    foreach ($except as $name) {  //排除属性
        unset($values[$name]);
    }

    return $values;
}
```
fields也是获取全部非静态共有属性，只不过返回结构不同  
```
public function fields()
{
    $fields = $this->attributes();

    return array_combine($fields, $fields);
}
```
base\Model有trait ArrayableTrait，ArrayableTrait里面也有一个fields，区别就是Model的fields不会获取非静态共有属性  
```
public function fields()
{
    $fields = array_keys(Yii::getObjectVars($this));
    return array_combine($fields, $fields);
}
```
attributeLabels和attributeHints方法是获取全部属性标签和属性hint，需要自己去重写这两个方法  
hint其实没理解到底应该怎么用，其实我感觉直接使用标签就够了  
```
public function attributeLabels()
{
    return [];
}
public function attributeHints()
{
    return [];
}
public function getAttributeLabel($attribute)   //获取$attribute属性标签
{
    $labels = $this->attributeLabels();
    return isset($labels[$attribute]) ? $labels[$attribute] : $this->generateAttributeLabel($attribute);
}
public function getAttributeHint($attribute)  //获取属性hints
{
    $hints = $this->attributeHints();
    return isset($hints[$attribute]) ? $hints[$attribute] : '';
}
public function generateAttributeLabel($name)
{
    return Inflector::camel2words($name, true);   //其实就是mb_ucwords
}
```
重写
```
class User extends Model
{
    public function attributeLabels()
    {
        return [
            "name"=>"a name",   //name被注册为一个属性标签
        ];
    }
    
    public function attributeHints()
    {
        return [
            "age"=>"x age",   //age被注册为一个属性hint
        ];
    }
}
```
isAttributeRequired方法会判断参数$attribute是否需要被RequiredValidator类校验  
```
public function isAttributeRequired($attribute)
{
    //获取$attribute在该场景下的全部校验类
    foreach ($this->getActiveValidators($attribute) as $validator) {  
        if ($validator instanceof RequiredValidator && $validator->when === null) {
            return true;
        }
    }

    return false;
}
```
获取safe属性
```
public function safeAttributes()
{
    //获取本场景
    $scenario = $this->getScenario();
    //获取场景与属性的对应关系
    $scenarios = $this->scenarios();
    if (!isset($scenarios[$scenario])) {
        return [];
    }
    $attributes = [];
    foreach ($scenarios[$scenario] as $attribute) {
        if ($attribute[0] !== '!' && !in_array('!' . $attribute, $scenarios[$scenario])) {
            $attributes[] = $attribute;
        }
    }
    return $attributes;
}
```
关于safe属性，如果有如下配置，场景为default，那么safe属性就是name和sex
```
class User extends Model{
    public function rules()
    {
        return [
            [['name'], 'required', 'on' => 'default'],
            [['sex'], 'boolean', 'on' => ['default','login']],
            [['!height'], 'boolean', 'except'=>"unregister"],
        ];
    }
}
```
还有一个属性是active属性，相对于safe属性，就是name、sex、height
```
public function activeAttributes()
{
    $scenario = $this->getScenario();
    $scenarios = $this->scenarios();
    if (!isset($scenarios[$scenario])) {
        return [];
    }
    $attributes = array_keys(array_flip($scenarios[$scenario]));
    foreach ($attributes as $i => $attribute) {
        if ($attribute[0] === '!') {
            $attributes[$i] = substr($attribute, 1);
        }
    }
    return $attributes;
}
```
给属性赋值的操作是setAttributes
```
public function setAttributes($values, $safeOnly = true)
{
    if (is_array($values)) {
        //判断是获取safe属性还是直接获取类的非静态共有方法
        $attributes = array_flip($safeOnly ? $this->safeAttributes() : $this->attributes());
        foreach ($values as $name => $value) {
            if (isset($attributes[$name])) {
                //给类属性赋值
                $this->$name = $value;
            } elseif ($safeOnly) {
                //对不安全的属性记录日志
                $this->onUnsafeAttribute($name, $value);
            }
        }
    }
}
//对不安全的属性记录日志，只在debug模式下会记录日志
public function onUnsafeAttribute($name, $value)
{
    if (YII_DEBUG) {
        Yii::debug("Failed to set unsafe attribute '$name' in '" . get_class($this) . "'.", __METHOD__);
    }
}
```
setAttributes也叫块赋值，只用一行代码将用户所有输入填充到一个模型, 以下两段代码效果是相同的， 都是将终端用户输入的表单数据赋值到Model   
```
$model = new \app\models\ContactForm;
$model->attributes = \Yii::$app->request->post('ContactForm');
```
```
$model = new \app\models\ContactForm;
$data = \Yii::$app->request->post('ContactForm', []);
$model->name = isset($data['name']) ? $data['name'] : null;
$model->email = isset($data['email']) ? $data['email'] : null;
$model->subject = isset($data['subject']) ? $data['subject'] : null;
$model->body = isset($data['body']) ? $data['body'] : null;
```
或者也可以使用load、loadMultiple给属性赋值
```
public function load($data, $formName = null)
{
    $scope = $formName === null ? $this->formName() : $formName;
    if ($scope === '' && !empty($data)) {
        $this->setAttributes($data);

        return true;
    } elseif (isset($data[$scope])) {
        $this->setAttributes($data[$scope]);

        return true;
    }

    return false;
}
public static function loadMultiple($models, $data, $formName = null)
{
    if ($formName === null) {
        /* @var $first Model|false */
        $first = reset($models);
        if ($first === false) {
            return false;
        }
        $formName = $first->formName();
    }

    $success = false;
    foreach ($models as $i => $model) {
        /* @var $model Model */
        if ($formName == '') {
            if (!empty($data[$i]) && $model->load($data[$i], '')) {
                $success = true;
            }
        } elseif (!empty($data[$formName][$i]) && $model->load($data[$formName][$i], '')) {
            $success = true;
        }
    }

    return $success;
}
```
# 场景
一个Model可以用在多个场景下，比如User模型用于登录和注册场景，不同的场景会有不同的校验规则和属性  
默认Model场景为default
```
const SCENARIO_DEFAULT = 'default';
private $_scenario = self::SCENARIO_DEFAULT;
```
因为Model继承于Component，所以可以使用属性注入，控制器代码如下  
```
public function actionUser(){
    $user_model = User::instance();
    $user_model -> scenario = "login";
    return $user_model->scenario;
}
```
会调用Model底层的getScenario和setScenario
```
public function getScenario()
{
    return $this->_scenario;
}
public function setScenario($value)
{
    $this->_scenario = $value;
}
```
也可以在实例化的时候直接给场景赋值，底层走的是BaseYii::configure，然后走属性注入的魔术方法    
```
public function actionUser(){
    $user_model = new User(["scenario"=>"login"]);
}
```
# 验证
验证需要申明验证规则，base\Model的rules方法返回的是空数组
```
public function rules()
{
    return [];
}
```
所以需要重写这个方法
```
class User extends Model
{
    public function rules()
    {
        return [
            [['name'], 'required'],
            [['height','sex'], 'boolean', 'on' => ['default','login']],
            [['age'], 'boolean', 'except'=>"unregister"],
        ];
    }
}
```
rules方法的规则如下  
- 如果没有on或者except标识，则说明本条规则在所有场景下适用
- 如果有on标识，则说明本条规则只在on下适用
- 如果有except标识，则说明本条规则除了except下适用  
所以以上User类的规则如下  
- defaults场景下验证name、height、sex、age
- login场景下验证name、height、sex、age  
- unregister场景下验证name  
获取验证规则可以使用scenarios方法  
```
public function scenarios()
{
    //默认default场景
    $scenarios = [self::SCENARIO_DEFAULT => []];
    //获取验证规则类，并且遍历
    foreach ($this->getValidators() as $validator) {
        //on下的验证规则
        foreach ($validator->on as $scenario) {
            $scenarios[$scenario] = [];
        }
        //except下的验证规则
        foreach ($validator->except as $scenario) {
            $scenarios[$scenario] = [];
        }
    }
    $names = array_keys($scenarios);
    //获取验证规则类，并且遍历
    foreach ($this->getValidators() as $validator) {
        //如果本条验证规则没有on和except，就是所有场景下都适用
        if (empty($validator->on) && empty($validator->except)) {
            foreach ($names as $name) {
                foreach ($validator->attributes as $attribute) {
                    $scenarios[$name][$attribute] = true;
                }
            }
        //如果本条验证规则只在on场景下适用
        } elseif (empty($validator->on)) {
            foreach ($names as $name) {
                //这个验证规则不在except场景下出现
                if (!in_array($name, $validator->except, true)) {
                    foreach ($validator->attributes as $attribute) {
                        $scenarios[$name][$attribute] = true;
                    }
                }
            }
        } else {
            foreach ($validator->on as $name) {
                foreach ($validator->attributes as $attribute) {
                    $scenarios[$name][$attribute] = true;
                }
            }
        }
    }
    foreach ($scenarios as $scenario => $attributes) {
        if (!empty($attributes)) {
            $scenarios[$scenario] = array_keys($attributes);
        }
    }
    return $scenarios;
}
```
获取验证类逻辑如下  
```
public function getValidators()
{
    if ($this->_validators === null) {
        $this->_validators = $this->createValidators();
    }
    return $this->_validators;
}
public function createValidators()
{
    $validators = new ArrayObject();
    foreach ($this->rules() as $rule) {
        if ($rule instanceof Validator) {
            $validators->append($rule);
        } elseif (is_array($rule) && isset($rule[0], $rule[1])) { // attributes, validator type
            $validator = Validator::createValidator($rule[1], $this, (array) $rule[0], array_slice($rule, 2));
            $validators->append($validator);
        } else {
            throw new InvalidConfigException('Invalid validation rule: a rule must specify both attribute names and validator type.');
        }
    }

    return $validators;
}
```
涉及到了底层了验证Validator类，验证通过createValidator方法实例化  
```
public static function createValidator($type, $model, $attributes, $params = [])
{
    $params['attributes'] = $attributes;

    if ($type instanceof \Closure || ($model->hasMethod($type) && !isset(static::$builtInValidators[$type]))) {
        // method-based validator
        $params['class'] = __NAMESPACE__ . '\InlineValidator';
        $params['method'] = $type;
    } else {
        if (isset(static::$builtInValidators[$type])) {
            $type = static::$builtInValidators[$type];
        }
        if (is_array($type)) {
            $params = array_merge($type, $params);
        } else {
            $params['class'] = $type;
        }
    }
    return Yii::createObject($params);
}
```
在createValidator里面实例化后还会走init
```
public function init()
{
    parent::init();
    $this->attributes = (array) $this->attributes;
    $this->on = (array) $this->on;
    $this->except = (array) $this->except;
}
```
可用的验证规则如下
```
public static $builtInValidators = [
    'boolean' => 'yii\validators\BooleanValidator',
    'captcha' => 'yii\captcha\CaptchaValidator',
    'compare' => 'yii\validators\CompareValidator',
    'date' => 'yii\validators\DateValidator',
    'datetime' => [
        'class' => 'yii\validators\DateValidator',
        'type' => DateValidator::TYPE_DATETIME,
    ],
    'time' => [
        'class' => 'yii\validators\DateValidator',
        'type' => DateValidator::TYPE_TIME,
    ],
    'default' => 'yii\validators\DefaultValueValidator',
    'double' => 'yii\validators\NumberValidator',
    'each' => 'yii\validators\EachValidator',
    'email' => 'yii\validators\EmailValidator',
    'exist' => 'yii\validators\ExistValidator',
    'file' => 'yii\validators\FileValidator',
    'filter' => 'yii\validators\FilterValidator',
    'image' => 'yii\validators\ImageValidator',
    'in' => 'yii\validators\RangeValidator',
    'integer' => [
        'class' => 'yii\validators\NumberValidator',
        'integerOnly' => true,
    ],
    'match' => 'yii\validators\RegularExpressionValidator',
    'number' => 'yii\validators\NumberValidator',
    'required' => 'yii\validators\RequiredValidator',
    'safe' => 'yii\validators\SafeValidator',
    'string' => 'yii\validators\StringValidator',
    'trim' => [
        'class' => 'yii\validators\FilterValidator',
        'filter' => 'trim',
        'skipOnArray' => true,
    ],
    'unique' => 'yii\validators\UniqueValidator',
    'url' => 'yii\validators\UrlValidator',
    'ip' => 'yii\validators\IpValidator',
];
```
如果不想用以上的验证规则，可以在自己的类里面新建一个验证规则  
```
class User extends Model
{
    public function rules()
    {
        return [
            [["name"],"myValidators"]
        ];
    }
    
    public function myValidators($value)
    {
        if($value != 123){
            return false;
        }
        return true;
    }
}
```
执行验证的方法是validate，就是根据rules和场景来进行Validator类的验证
```
public function validate($attributeNames = null, $clearErrors = true)
{
    if ($clearErrors) {
        //清除所有验证错误信息
        $this->clearErrors();
    }
    
    //验证开始事件
    if (!$this->beforeValidate()) {
        return false;
    }
    //获取所有场景下的验证规则
    $scenarios = $this->scenarios();
    //获取场景
    $scenario = $this->getScenario();
    if (!isset($scenarios[$scenario])) {
        throw new InvalidArgumentException("Unknown scenario: $scenario");
    }
    //获取本场景下的验证规则
    if ($attributeNames === null) {
        $attributeNames = $this->activeAttributes();
    }
    $attributeNames = (array)$attributeNames;
    //获取验证规则对应的验证类，遍历
    foreach ($this->getActiveValidators() as $validator) {
        $validator->validateAttributes($this, $attributeNames);
    }
    //验证结束事件
    $this->afterValidate();
    //是否有验证错误信息
    return !$this->hasErrors();
}
```
具体的验证在Validator类里面的validateAttributes方法  
```
public function validateAttributes($model, $attributes = null)
{
    //获取被验证类需要验证的属性
    $attributes = $this->getValidationAttributes($attributes);
    foreach ($attributes as $attribute) {
        //在该属性已经有错误信息或者该属性是空的情况下跳过
        $skip = $this->skipOnError && $model->hasErrors($attribute)
            || $this->skipOnEmpty && $this->isEmpty($model->$attribute);
        if (!$skip) {
            if ($this->when === null || call_user_func($this->when, $model, $attribute)) {
                //执行验证
                $this->validateAttribute($model, $attribute);
            }
        }
    }
}
```
判断为空的逻辑如下
```
public function isEmpty($value)
{
    if ($this->isEmpty !== null) {
        return call_user_func($this->isEmpty, $value);
    }

    return $value === null || $value === [] || $value === '';
}
```
如果要在申明规则下不跳过错误并且不跳过空的，可以
```
public function rules()
{
    return [
        [['name'], 'required'],
        //申明height和sex属性不跳过错误并且不跳过空
        [['height','sex'], 'boolean', 'on' => ['default','login'],'skipOnEmpty'=>false,"skipOnError"=>false],
    ];
}
```
验证方法为各个验证类的validateValue方法，如果验证失败会执行addError添加验证错误信息
```
public function addError($attribute, $error = '')
{
    $this->_errors[$attribute][] = $error;
}
```
