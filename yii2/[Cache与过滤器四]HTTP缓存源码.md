## 目录
* [执行流程](#执行流程)
* [源码分析](#源码分析)

# 执行流程

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
可见HTTP缓存
