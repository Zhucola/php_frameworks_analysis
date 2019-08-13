## 目录
* [Session](#Session)
* [Redis_Session](#Redis_Session)

# Session 
  Sesssion组件默认在config/web.php是没有配置的，在Application底层代码中会走默认的依赖配置
```
//verdor/yiisoft/yii2/base/Application.php
public function preInit(&$config)
{
    ...
    foreach ($this->coreComponents() as $id => $component) {
        if (!isset($config['components'][$id])) {
            $config['components'][$id] = $component;
        } elseif (is_array($config['components'][$id]) && !isset($config['components'][$id]['class'])) {
            $config['components'][$id]['class'] = $component['class'];
        }
    }
}

public function coreComponents()
{
    return [
        'log' => ['class' => 'yii\log\Dispatcher'],
        'view' => ['class' => 'yii\web\View'],
        'formatter' => ['class' => 'yii\i18n\Formatter'],
        'i18n' => ['class' => 'yii\i18n\I18N'],
        'mailer' => ['class' => 'yii\swiftmailer\Mailer'],
        'urlManager' => ['class' => 'yii\web\UrlManager'],
        'assetManager' => ['class' => 'yii\web\AssetManager'],
        'security' => ['class' => 'yii\base\Security'],
    ];
}

//verdor/yiisoft/yii2/web/Application.php
public function coreComponents()
{
    return array_merge(parent::coreComponents(), [
        'request' => ['class' => 'yii\web\Request'],
        'response' => ['class' => 'yii\web\Response'],
        'session' => ['class' => 'yii\web\Session'],
        'user' => ['class' => 'yii\web\User'],
        'errorHandler' => ['class' => 'yii\web\ErrorHandler'],
    ]);
}
````
但是你也可以在web/config.php中配置Session组件
```
$config = [
    'id' => 'basic',
    'basePath' => dirname(__DIR__),
    'bootstrap' => ['log'],
    'aliases' => [
        '@bower' => '@vendor/bower-asset',
        '@npm'   => '@vendor/npm-asset',
    ],
    'components' => [
        'session' => [
            'cookieParams' => [
                'lifetime' => time()+30,
                'httponly' => 'true',
            ]
        ]
        ...      
```
Session组件源码比较简单，就是基本的session操作
```
$session = Yii::$app->session;
$session->set("a","b");
$session["b"] = "c";
$session["arr"] = [
  "name" => 123,
  "age"  => 222
];
var_dump($session->has("a"));
var_dump($session->get("b"));
var_dump($_SESSION);
```
实例化Session组件会走到init
```
public function init()
{
    parent::init();
    //注册一个session关闭的函数，在应用执行结束时候调用
    register_shutdown_function([$this, 'close']);
    if ($this->getIsActive()) {//如果session已经session_start了，就去判断flash性质的session属性是否可用
        Yii::warning('Session is already started', __METHOD__);
        $this->updateFlashCounters();
    }
}
```
session的获取和设置源码如下
```
public function set($key, $value)
{
    $this->open();
    $_SESSION[$key] = $value;
}
public function get($key, $defaultValue = null)
{
    $this->open();
    return isset($_SESSION[$key]) ? $_SESSION[$key] : $defaultValue;
}
public function remove($key)
{
    $this->open();
    if (isset($_SESSION[$key])) {
        $value = $_SESSION[$key];
        unset($_SESSION[$key]);

        return $value;
    }

    return null;
}
public function has($key)
{
    $this->open();
    return isset($_SESSION[$key]);
}
```
可见不需要手动调用session_start，在open方法中有session_start操作
```
public function open()
{
    if ($this->getIsActive()) {
        return;
    }
    //注册自定义的session handler
    $this->registerSessionHandler();
    //设置与session相关的cookie属性
    $this->setCookieParamsInternal();
    //session开启
    YII_DEBUG ? session_start() : @session_start();

    if ($this->getIsActive()) {
        Yii::info('Session started', __METHOD__);
        //如果开启成功，则走flash判断逻辑
        $this->updateFlashCounters();
    } else {
        $error = error_get_last();
        $message = isset($error['message']) ? $error['message'] : 'Failed to start session.';
        Yii::error($message, __METHOD__);
    }
}
public function getIsActive()
{
    return session_status() === PHP_SESSION_ACTIVE;
}
```
设置与session相关的cookie属性源码如下
```
private function setCookieParamsInternal()
{
    $data = $this->getCookieParams();
    if (isset($data['lifetime'], $data['path'], $data['domain'], $data['secure'], $data['httponly'])) {
        session_set_cookie_params($data['lifetime'], $data['path'], $data['domain'], $data['secure'], $data['httponly']);
    } else {
        throw new InvalidArgumentException('Please make sure cookieParams contains these elements: lifetime, path, domain, secure and httponly.');
    }
}
```
基本的session增删改查操作源码非常简单，因为Session组件实现了IteratorAggregate、ArrayAccess、Countable接口，所以可以进行数组式访问、迭代和count操作
```
public function offsetSet($offset, $item)
{
    $this->open();
    $_SESSION[$offset] = $item;
}
public function offsetGet($offset)
{
    $this->open();

    return isset($_SESSION[$offset]) ? $_SESSION[$offset] : null;
}
public function getCount()
{
    $this->open();
    return count($_SESSION);
}
public function getIterator()
{
    $this->open();
    //返回的是一个自己封装的迭代器
    return new SessionIterator();
}
```
