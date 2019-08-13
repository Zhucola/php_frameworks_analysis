## 目录
* [Session](#Session)
* [Flash数据](#Flash数据)
* [Redis_Session](#Redis_Session)
* [Cookie](#Cookie)

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
# Flash数据
  Session组件封装了一个Flash数据，就是设置了之后只在下一次访问有效
```
$session = Yii::$app->session;
$session->setFlash("a",123); //在下一次访问$session->getFlash("a")后失效
$session->setFlash("b",123,false); //在下一次调用实例化Session组件后失效
```
  Flash数据使用第三个参数来控制如何失效，默认是下一次使用getFlash获取后失效  
  如果设置第三个参数为false，则在下一次实例化Session组件后失效
```
public function setFlash($key, $value = true, $removeAfterAccess = true)
{
    $counters = $this->get($this->flashParam, []);
    //控制数据如何失效
    $counters[$key] = $removeAfterAccess ? -1 : 0;
    $_SESSION[$key] = $value;
    $_SESSION[$this->flashParam] = $counters;
}
```
获取Flash数据和初始化Session组件会控制Flash数据是否失效
```
public function getFlash($key, $defaultValue = null, $delete = false)
{
    $counters = $this->get($this->flashParam, []);
    if (isset($counters[$key])) {
        $value = $this->get($key, $defaultValue);
        if ($delete) {
            $this->removeFlash($key);
        } elseif ($counters[$key] < 0) {
            // mark for deletion in the next request
            $counters[$key] = 1;
            $_SESSION[$this->flashParam] = $counters;
        }

        return $value;
    }

    return $defaultValue;
}
protected function updateFlashCounters()
{
    $counters = $this->get($this->flashParam, []);
    if (is_array($counters)) {
        foreach ($counters as $key => $count) {
            if ($count > 0) {
                unset($counters[$key], $_SESSION[$key]);
            } elseif ($count == 0) {
                $counters[$key]++;
            }
        }
        $_SESSION[$this->flashParam] = $counters;
    } else {
        // fix the unexpected problem that flashParam doesn't return an array
        unset($_SESSION[$this->flashParam]);
    }
}
```
# Redis_Session
  在多服务器情况下需要Session共享，可以将session存到redis里面，需要Yii2安装redis扩展
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
         'session_redis' => [
            'class' => 'yii\redis\Connection',
            'hostname' => '192.168.124.10',
            'port' => 6380,
            'database' => 0,
        ],
        'session'=>[
            'class'=>'yii\redis\Session',
            'redis'=>'session_redis'
        ]
        .....
```
底层其实使用的是session_set_save_handler来做读写关闭操作
```
//yii2-redis/src/Session.php
public function getUseCustomStorage()
{
    return true;
}
//yii2/web/Session.php
protected function registerSessionHandler()
{
    if ($this->handler !== null) {
        if (!is_object($this->handler)) {
            $this->handler = Yii::createObject($this->handler);
        }
        if (!$this->handler instanceof \SessionHandlerInterface) {
            throw new InvalidConfigException('"' . get_class($this) . '::handler" must implement the SessionHandlerInterface.');
        }
        YII_DEBUG ? session_set_save_handler($this->handler, false) : @session_set_save_handler($this->handler, false);
    } elseif ($this->getUseCustomStorage()) {
        if (YII_DEBUG) {
            session_set_save_handler(
                [$this, 'openSession'],
                [$this, 'closeSession'],
                [$this, 'readSession'],
                [$this, 'writeSession'],
                [$this, 'destroySession'],
                [$this, 'gcSession']
            );
        } else {
            @session_set_save_handler(
                [$this, 'openSession'],
                [$this, 'closeSession'],
                [$this, 'readSession'],
                [$this, 'writeSession'],
                [$this, 'destroySession'],
                [$this, 'gcSession']
            );
        }
    }
}
```
在调用session_start时候，会调用openSession和readSession函数，在写session时候会调用writeSession函数
```
protected function calculateKey($id)
{
    return $this->keyPrefix . md5(json_encode([__CLASS__, $id]));
}
public function readSession($id)
{
    $data = $this->redis->executeCommand('GET', [$this->calculateKey($id)]);

    return $data === false || $data === null ? '' : $data;
}

public function writeSession($id, $data)
{
    return (bool) $this->redis->executeCommand('SET', [$this->calculateKey($id), $data, 'EX', $this->getTimeout()]);
}
```
需要注意的是，使用redis-session组件，写session在redis里面的最大生存周期是ini_get('session.gc_maxlifetime')
```
public function getTimeout()
{
    return (int) ini_get('session.gc_maxlifetime');
}
```
# Cookie
  Yii的Cookie组件分为请求Cookie和响应Cookie
```
//获取请求的cookie
$request_cookies = Yii::$app->request->cookies;
//响应的cookie
$response_cookies = Yii::$app->response->cookies;
```
因为cookie有加密操作，所以先从发送一个cookie开始说起  
```
$response_cookies = Yii::$app->response->cookies;
$response_cookies->add(new \yii\web\Cookie([
    'name' => 'language',
    'value' => 'zh-CN',
]));
```
响应的cookie会从Response组件中获取
```
//yiisoft/yii2/web/Response.php
public function getCookies()
{
    if ($this->_cookies === null) {
        $this->_cookies = new CookieCollection();
    }

    return $this->_cookies;
}
```
add操作会实例化一个Cookie类
```
class Cookie extends \yii\base\BaseObject
{
    public $name;

    public $value = '';

    public $domain = '';

    public $expire = 0;

    public $path = '/';

    public $secure = false;

    public $httpOnly = true;

    public function __toString()
    {
        return (string) $this->value;
    }
}
```
可见默认的httponly为true，secure为false等 
add操作就是将Cookie类添加进CookieCollection  
```
public function add($cookie)
{
    if ($this->readOnly) {
        throw new InvalidCallException('The cookie collection is read only.');
    }
    $this->_cookies[$cookie->name] = $cookie;
}
```
添加了一个cookie后，也可以删除它，因为Response组件的Cookie的readOnly属性是false
```
public function remove($cookie, $removeFromBrowser = true)
{
    if ($this->readOnly) {
        throw new InvalidCallException('The cookie collection is read only.');
    }
    if ($cookie instanceof Cookie) {
        $cookie->expire = 1;   //过期时间为1，就是删除
        $cookie->value = '';
    } else {
        $cookie = Yii::createObject([
            'class' => 'yii\web\Cookie',
            'name' => $cookie,
            'expire' => 1,
        ]);
    }
    if ($removeFromBrowser) {
        $this->_cookies[$cookie->name] = $cookie;
    } else {
        unset($this->_cookies[$cookie->name]);
    }
}
```
最后响应的时候，会走到Response组件的sendCookies，会对cookie进行加密操作
```
protected function sendCookies()
{
    if ($this->_cookies === null) {
        return;
    }
    $request = Yii::$app->getRequest();
    if ($request->enableCookieValidation) {
        if ($request->cookieValidationKey == '') {
            throw new InvalidConfigException(get_class($request) . '::cookieValidationKey must be configured with a secret key.');
        }
        $validationKey = $request->cookieValidationKey;
    }
    foreach ($this->getCookies() as $cookie) {
        $value = $cookie->value;
        if ($cookie->expire != 1 && isset($validationKey)) {
            //加密
            $value = Yii::$app->getSecurity()->hashData(serialize([$cookie->name, $value]), $validationKey);
        }
        setcookie($cookie->name, $value, $cookie->expire, $cookie->path, $cookie->domain, $cookie->secure, $cookie->httpOnly);
    }
}
```
加密的方式默认是sha256
```
public function hashData($data, $key, $rawHash = false)
{
    //$this->macHash默认是sha256
    $hash = hash_hmac($this->macHash, $data, $key, $rawHash);
    if (!$hash) {
        throw new InvalidConfigException('Failed to generate HMAC with hash algorithm: ' . $this->macHash);
    }

    return $hash . $data;
}
```
也就是说会用配置文件中的cookieValidationKey，去对serialize([$cookie->name, $value])做sha256加密，返回的结果是响应的cookie值
