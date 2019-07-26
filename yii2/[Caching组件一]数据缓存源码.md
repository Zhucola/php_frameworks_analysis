## 目录
* [文件缓存](#文件缓存)
* [Redis缓存](#Redis缓存)

**缓存只讲一下get、set、gc方法，其他方法源码很简单，不做涉及**

# 文件缓存
缓存组件通过依赖注入到组件中，默认使用的是FileCache缓存
```
'components' => [
    ....
    'cache' => [
        'class' => 'yii\caching\FileCache',
    ],
    ....
],
```
这样就可以在应用中通过cache组件来使用缓存了
```
$cache = Yii::$app->get("cache");
$cache->set("a",1);
echo $cache->get("a");
```
缓存类都要继承于底层yii\caching\Cache类，源码很简单，涉及的方法和属性也比较少，如果自己封装一套cache需要强制实现抽象方法
```
abstract protected function getValue($key);
abstract protected function setValue($key, $value, $duration);
abstract protected function addValue($key, $value, $duration);
abstract protected function deleteValue($key);
abstract protected function flushValues();
```
因为FileCache类有init，就是在实例化后需要走的第一个方法  
```
public function init()
{
    parent::init();
    //缓存路径别名，可以配成@开头的，也可以配置成全路径的
    $this->cachePath = Yii::getAlias($this->cachePath);
    if (!is_dir($this->cachePath)) {
        //如果目录不存在就创建目录
        FileHelper::createDirectory($this->cachePath, $this->dirMode, true);
    }
}
```
默认的缓存路径在
```
public $cachePath = '@runtime/cache';
```
缓存文件后缀为
```
public $cacheFileSuffix = '.bin';
```
目录深度为
```
public $directoryLevel = 1;
```
文件权限为
```
public $dirMode = 0775;
```
还需要执行一下父类的构造函数，去判断igbinary扩展是否可用，igbinary相比于serialize长度更短，大小更小
```
public function init()
{
    parent::init();
    $this->_igbinaryAvailable = \extension_loaded('igbinary');
}
```
创建目录是一个递归操作
```
public static function createDirectory($path, $mode = 0775, $recursive = true)
{
    if (is_dir($path)) {
        return true;
    }
    $parentDir = dirname($path);
    // recurse if parent dir does not exist and we are not at the root of the file system.
    if ($recursive && !is_dir($parentDir) && $parentDir !== $path) {
        static::createDirectory($parentDir, $mode, true);
    }
    try {
        if (!mkdir($path, $mode)) {
            return false;
        }
    } catch (\Exception $e) {
        if (!is_dir($path)) {// https://github.com/yiisoft/yii2/issues/9288
            throw new \yii\base\Exception("Failed to create directory \"$path\": " . $e->getMessage(), $e->getCode(), $e);
        }
    }
    try {
        return chmod($path, $mode);
    } catch (\Exception $e) {
        throw new \yii\base\Exception("Failed to change permissions for directory \"$path\": " . $e->getMessage(), $e->getCode(), $e);
    }
}
```
设置缓存的set操作如下
```
public function set($key, $value, $duration = null, $dependency = null)
{
    //过期时间
    if ($duration === null) {
        $duration = $this->defaultDuration;
    }
    //依赖
    if ($dependency !== null && $this->serializer !== false) {
        $dependency->evaluateDependency($this);
    }
    //对值进行序列号
    if ($this->serializer === null) {
        $value = serialize([$value, $dependency]);
    } elseif ($this->serializer !== false) {
        $value = call_user_func($this->serializer[0], [$value, $dependency]);
    }
    //构建key，因为key可以是一个数组
    $key = $this->buildKey($key);

    return $this->setValue($key, $value, $duration);
}
```
因为key不仅仅可以是字符串，yii缓存的key还可以是数组，所以需要构建key
```
public function buildKey($key)
{
    if (is_string($key)) {
        //长度大于32就直接md5
        $key = ctype_alnum($key) && StringHelper::byteLength($key) <= 32 ? $key : md5($key);
    } else {
        if ($this->_igbinaryAvailable) {
            $serializedKey = igbinary_serialize($key);
        } else {
            $serializedKey = serialize($key);
        }

        $key = md5($serializedKey);
    }
    //键名拼键前缀
    return $this->keyPrefix . $key;
}
```
获取缓存的get操作如下
```
public function get($key)
{
    $key = $this->buildKey($key);
    $value = $this->getValue($key);
    if ($value === false || $this->serializer === false) {
        return $value;
    } elseif ($this->serializer === null) {
        $value = unserialize($value);
    } else {
        $value = call_user_func($this->serializer[1], $value);
    }
    if (is_array($value) && !($value[1] instanceof Dependency && $value[1]->isChanged($this))) {
        return $value[0];
    }

    return false;
}
```
