**这个cache的难点就是在于以什么标准去算Last-Modify，如果以文件修改时间来当做Last-Modify，那么PHP获取一个动态的数据库数据，那么客户端刷新会得不到新的数据库数据**  

## 目录
* [HttpCache基本知识](#HttpCache基本知识)
* [执行流程](#执行流程)
* [源码分析](#源码分析)

# HttpCache基本知识
[HttpCache基本知识](https://github.com/Zhucola/advanced-nginx/blob/master/HTTP%E7%BC%93%E5%AD%98%E4%B8%8Eheader%E6%A8%A1%E5%9D%97.md)
# 执行流程
- 判断HttpCache是否可用，根据请求方法和lastModify、etagSeed回调
- 算出lastModify和etag
- 发送响应头Cache-Control，默认让客户端缓存3600秒
- 发送响应头ETag
- 判断是否发生304
- 发送响应头Last-Modify
- 响应304或者执行后续流程

# 源码分析
HTTP缓存是一个过滤器，使用了request、response组件，我们直接从HTTP缓存过滤器的beforeAction开始，beforeAction是执行控制器具体方法前要执行的
```
public function beforeAction($action)
{
    if (!$this->enabled) {
        return true;
    }

    $verb = Yii::$app->getRequest()->getMethod();
    if ($verb !== 'GET' && $verb !== 'HEAD' || $this->lastModified === null && $this->etagSeed === null) {
        return true;
    }

    $lastModified = $etag = null;
    if ($this->lastModified !== null) {
        $lastModified = call_user_func($this->lastModified, $action, $this->params);
    }
    if ($this->etagSeed !== null) {
        $seed = call_user_func($this->etagSeed, $action, $this->params);
        if ($seed !== null) {
            $etag = $this->generateEtag($seed);
        }
    }
    $this->sendCacheControlHeader();
    $response = Yii::$app->getResponse();
    if ($etag !== null) {
        $response->getHeaders()->set('Etag', $etag);
    }
    $cacheValid = $this->validateCache($lastModified, $etag);
    // https://tools.ietf.org/html/rfc7232#section-4.1
    if ($lastModified !== null && (!$cacheValid || ($cacheValid && $etag === null))) {
        $response->getHeaders()->set('Last-Modified', gmdate('D, d M Y H:i:s', $lastModified) . ' GMT');
    }
    if ($cacheValid) {
        $response->setStatusCode(304);
        return false;
    }

    return true;
}
```
可见HTTP缓存只能用于请求方法为GET或者HEAD，或者定义了lastModify或者etagSeed回调的情况
```
$verb = Yii::$app->getRequest()->getMethod();
if ($verb !== 'GET' && $verb !== 'HEAD' || $this->lastModified === null && $this->etagSeed === null) {
    return true;
}
```
etag算法是如下
```
protected function generateEtag($seed)
{
    $etag = '"' . rtrim(base64_encode(sha1($seed, true)), '=') . '"';
    return $this->weakEtag ? 'W/' . $etag : $etag;
}
```
默认会让客户端缓存资源3600秒
```
public $cacheControlHeader = 'public, max-age=3600';
protected function sendCacheControlHeader()
{
    if ($this->sessionCacheLimiter !== null) {
        if ($this->sessionCacheLimiter === '' && !headers_sent() && Yii::$app->getSession()->getIsActive()) {
            header_remove('Expires');
            header_remove('Cache-Control');
            header_remove('Last-Modified');
            header_remove('Pragma');
        }

        Yii::$app->getSession()->setCacheLimiter($this->sessionCacheLimiter);
    }

    $headers = Yii::$app->getResponse()->getHeaders();
    if ($this->cacheControlHeader !== null) {
        $headers->set('Cache-Control', $this->cacheControlHeader);
    }
}
```
判断是否发生304响应的逻辑为如下，etag会比lastModify的优先级高
```
protected function validateCache($lastModified, $etag)
{
    if (Yii::$app->request->headers->has('If-None-Match')) {
        return $etag !== null && in_array($etag, Yii::$app->request->getETags(), true);
    } elseif (Yii::$app->request->headers->has('If-Modified-Since')) {
        return $lastModified !== null && @strtotime(Yii::$app->request->headers->get('If-Modified-Since')) >= $lastModified;
    }

    return false;
}
```
