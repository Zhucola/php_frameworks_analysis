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
Cache的get、set会执行真正缓存相关类的getValue、setValue
```
protected function setValue($key, $value, $duration)
{
    //gc逻辑
    $this->gc();
    //获取缓存文件绝对路径
    $cacheFile = $this->getCacheFile($key);
    if ($this->directoryLevel > 0) {
        //缓存文件深度
        @FileHelper::createDirectory(dirname($cacheFile), $this->dirMode, true);
    }
    if (is_file($cacheFile) && function_exists('posix_geteuid') && fileowner($cacheFile) !== posix_geteuid()) {
        @unlink($cacheFile);
    }
    if (@file_put_contents($cacheFile, $value, LOCK_EX) !== false) {
        if ($this->fileMode !== null) {
            @chmod($cacheFile, $this->fileMode);
        }
        if ($duration <= 0) {
            $duration = 31536000; // 1 year
        }
        //更新修改时间
        return @touch($cacheFile, $duration + time());
    }

    $error = error_get_last();
    Yii::warning("Unable to write cache file '{$cacheFile}': {$error['message']}", __METHOD__);
    return false;
}
```
缓存文件是有深度的，会根据key来做深度
```
protected function getCacheFile($key)
{
    if ($this->directoryLevel > 0) {
        $base = $this->cachePath;
        for ($i = 0; $i < $this->directoryLevel; ++$i) {
            if (($prefix = substr($key, $i + $i, 2)) !== false) {
                $base .= DIRECTORY_SEPARATOR . $prefix;
            }
        }

        return $base . DIRECTORY_SEPARATOR . $key . $this->cacheFileSuffix;
    }

    return $this->cachePath . DIRECTORY_SEPARATOR . $key . $this->cacheFileSuffix;
}
```
设置缓存有几率走gc操作，也可以使用flush方法来强制执行gc
```
public function gc($force = false, $expiredOnly = true)
{
    if ($force || mt_rand(0, 1000000) < $this->gcProbability) {
        $this->gcRecursive($this->cachePath, $expiredOnly);
    }
}
protected function gcRecursive($path, $expiredOnly)
{
    if (($handle = opendir($path)) !== false) {
        while (($file = readdir($handle)) !== false) {
            if ($file[0] === '.') {
                continue;
            }
            $fullPath = $path . DIRECTORY_SEPARATOR . $file;
            if (is_dir($fullPath)) {
                $this->gcRecursive($fullPath, $expiredOnly);
                if (!$expiredOnly) {
                    if (!@rmdir($fullPath)) {
                        $error = error_get_last();
                        Yii::warning("Unable to remove directory '{$fullPath}': {$error['message']}", __METHOD__);
                    }
                }
            } elseif (!$expiredOnly || $expiredOnly && @filemtime($fullPath) < time()) {
                if (!@unlink($fullPath)) {
                    $error = error_get_last();
                    Yii::warning("Unable to remove file '{$fullPath}': {$error['message']}", __METHOD__);
                }
            }
        }
        closedir($handle);
    }
}
```
getValue方法比较简单
```
protected function getValue($key)
{
    $cacheFile = $this->getCacheFile($key);

    if (@filemtime($cacheFile) > time()) {
        $fp = @fopen($cacheFile, 'r');
        if ($fp !== false) {
            @flock($fp, LOCK_SH);
            $cacheValue = @stream_get_contents($fp);
            @flock($fp, LOCK_UN);
            @fclose($fp);
            return $cacheValue;
        }
    }

    return false;
}
```
# Redis缓存
使用Redis缓存需要安装yii-redis，相关的redis基本源码可以参考(yii2/%5Bredis%5DConnection源码.md)
Redis缓存是可以使用从读主写模式的，默认是无主从关系，需要配置
```
public function actionTest(){
    $cache = Yii::$app->get("cache");
    $cache->enableReplicas = true;
    $cache->replicas = [
        ['hostname' => '192.168.124.10','port' => 6380,'database' => 0],  //也可以在web.php中配置cache信息
    ];
    $key = [3,4,5];
    $value = [1,2,3];
    $cache->set($key,$value);   //写操作走6379，在web.php中配置的cache信息
    var_dump($cache->get($key));
}
```
获取从节点的代码如下
```
protected function getReplica()
{
    if ($this->enableReplicas === false) {
        return $this->redis;
    }
    if ($this->_replica !== null) {
        return $this->_replica;
    }

    if (empty($this->replicas)) {
        return $this->_replica = $this->redis;
    }

    $replicas = $this->replicas;
    //打乱顺序
    shuffle($replicas);
    $config = array_shift($replicas);
    $this->_replica = Instance::ensure($config, Connection::className());
    return $this->_replica;
}
```
因为使用了redis组件，所以redis属性可以直接操作各种原始方法
```
$cache = Yii::$app->get("cache");
$cache -> redis -> get("a");
```
其他cache相关的get、set等操作其实就是redis的原始操作
