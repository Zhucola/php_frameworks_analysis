**PageCache还是需要走到PHP里面取做cache，应该使用nginx的proxy-cache来做PageCache**  
**不管是yii的PageCache还是nginx的proxy-cache，都是缓存了所有的数据，如果涉及到动态的信息(用户信息、订单详情等)，需要做动态处理，推荐ajax** 

## 目录
* [执行流程](#执行流程)
* [源码分析](#源码分析)

# 执行流程
- 项目初始化，路由解析
- 控制器初始化，走到base\Controller里面的runAction方法，执行beforeAction事件
- 加载过滤器行为，将PageCache的beforeAction事件注入，并且执行
- 判断requestedRoute(就是真实uri)拼的键是否在缓存中存在，存在则直接获取缓存数据，并且将beforeAction返回false，就是直接走到response组件的send了，不会再继续加载控制器方法了
- 如果缓存不存在，将页面响应信息存缓存

# 源码分析
PageCache其实就是一个过滤器
```
public function behaviors(){
    return [
        [
            'class' => 'yii\filters\PageCache',
            'only' => ['cacae*',"e"],
            'varyByRoute'=>true,
            'duration' => 60
        ]
    ];
}
```
依赖于beforeAction事件，在项目初始化后，路由解析后，会走到底层控制器base\Controller的runAction
```
public function runAction($id, $params = [])
{
    $action = $this->createAction($id);
    if ($action === null) {
        throw new InvalidRouteException('Unable to resolve the request: ' . $this->getUniqueId() . '/' . $id);
    }
    Yii::debug('Route to run: ' . $action->getUniqueId(), __METHOD__);

    if (Yii::$app->requestedAction === null) {
        Yii::$app->requestedAction = $action;
    }

    $oldAction = $this->action;
    $this->action = $action;

    $modules = [];
    $runAction = true;

    // call beforeAction on modules
    foreach ($this->getModules() as $module) {
        if ($module->beforeAction($action)) {
            array_unshift($modules, $module);
        } else {
            $runAction = false;
            break;
        }
    }

    $result = null;
    if ($runAction && $this->beforeAction($action)) {
        
        // run the action
        $result = $action->runWithParams($params);
        $result = $this->afterAction($action, $result);

        // call afterAction on modules
        foreach ($modules as $module) {
            /* @var $module Module */
            $result = $module->afterAction($action, $result);
        }
    }
    if ($oldAction !== null) {
        $this->action = $oldAction;
    }

    return $result;
}
```
具体的就是这段代码
```
if ($runAction && $this->beforeAction($action)) {
```
会去执行beforeAction事件
```
public function beforeAction($action)
{
    $event = new ActionEvent($action);
    $this->trigger(self::EVENT_BEFORE_ACTION, $event);
    return $event->isValid;
}
```
事件会去加载行为，具体的源码逻辑在
* [Behavior行为源码](yii2/%5B关键概念一%5DBehavior行为源码.md)
* [Events事件源码](yii2/%5B关键概念二%5DEvents事件源码.md)
* [过滤器源码](yii2/%5BCache与过滤器二%5D过滤器Filters源码.md)
这里就不讲过滤器底层逻辑了，直接将PageCahe的逻辑
```
public function beforeAction($action)
{
    if (!$this->enabled) {
        return true;
    }
    //实例化cache组件
    $this->cache = Instance::ensure($this->cache, 'yii\caching\CacheInterface');
    //依赖关系
    if (is_array($this->dependency)) {
        $this->dependency = Yii::createObject($this->dependency);
    }
    //获取response组件
    $response = Yii::$app->getResponse();
    //拼缓存的键
    $data = $this->cache->get($this->calculateCacheKey());
    //判断缓存是否可用
    if (!is_array($data) || !isset($data['cacheVersion']) || $data['cacheVersion'] !== static::PAGE_CACHE_VERSION) {
        $this->view->pushDynamicContent($this);
        //新开启一层ob_start
        ob_start();
        ob_implicit_flush(false);
        //注入事件
        $response->on(Response::EVENT_AFTER_SEND, [$this, 'cacheResponse']);
        Yii::debug('Valid page content is not found in the cache.', __METHOD__);
        //给底层过滤器true，让应用去执行控制器方法
        return true;
    }
    //缓存可用，直接给response组件赋值
    $this->restoreResponse($response, $data);
    Yii::debug('Valid page content is found in the cache.', __METHOD__);
    //给底层过滤器false，让应用跳过控制器方法
    return false;
}
```
PageCache的键名是根据路由组件拼的
```
protected function calculateCacheKey()
{
    $key = [__CLASS__];
    if ($this->varyByRoute) {
        $key[] = Yii::$app->requestedRoute;
    }
    return array_merge($key, (array)$this->variations);
}
```
requestRoute这个值在路径解析后被赋值
```
public function handleRequest($request)
{
    if (empty($this->catchAll)) {
        try {
            list($route, $params) = $request->resolve();
        } catch (UrlNormalizerRedirectException $e) {
            $url = $e->url;
            if (is_array($url)) {
                if (isset($url[0])) {
                    // ensure the route is absolute
                    $url[0] = '/' . ltrim($url[0], '/');
                }
                $url += $request->getQueryParams();
            }

            return $this->getResponse()->redirect(Url::to($url, $e->scheme), $e->statusCode);
        }
    } else {
        $route = $this->catchAll[0];
        $params = $this->catchAll;
        unset($params[0]);
    }
    try {
        Yii::debug("Route requested: '$route'", __METHOD__);
        //在这里被赋值
        $this->requestedRoute = $route;

        $result = $this->runAction($route, $params);
        
        if ($result instanceof Response) {
            return $result;
        }

        $response = $this->getResponse();
        if ($result !== null) {
            $response->data = $result;
        }

        return $response;
    } catch (InvalidRouteException $e) {
        throw new NotFoundHttpException(Yii::t('yii', 'Page not found.'), $e->getCode(), $e);
    }
}
```
如果缓存组件可用，就会将缓存中的各种响应信息赋值给response组件中，跳过控制器方法
```
protected function restoreResponse($response, $data)
{
    foreach (['format', 'version', 'statusCode', 'statusText', 'content'] as $name) {
        $response->{$name} = $data[$name];
    }
    foreach (['headers', 'cookies'] as $name) {
        if (isset($data[$name]) && is_array($data[$name])) {
            $response->{$name}->fromArray(array_merge($data[$name], $response->{$name}->toArray()));
        }
    }
    if (!empty($data['dynamicPlaceholders']) && is_array($data['dynamicPlaceholders'])) {
        $response->content = $this->updateDynamicContent($response->content, $data['dynamicPlaceholders'], true);
    }
    $this->afterRestoreResponse(isset($data['cacheData']) ? $data['cacheData'] : null);
}
```
具体真正响应的逻辑在Response组件中
```
public function send()
{
    if ($this->isSent) {
        return;
    }
    $this->trigger(self::EVENT_BEFORE_SEND);
    $this->prepare();
    $this->trigger(self::EVENT_AFTER_PREPARE);
    $this->sendHeaders();
    $this->sendContent();
    $this->trigger(self::EVENT_AFTER_SEND);
    $this->isSent = true;
}
```
如果缓存不可用，核心逻辑就是ob_get_clean，因为之前已经开过一次ob_start了，所以会获取到要响应的所有数据，将其保存起来
```
public function cacheResponse()
{
    $this->view->popDynamicContent();
    $beforeCacheResponseResult = $this->beforeCacheResponse();
    if ($beforeCacheResponseResult === false) {
        echo $this->updateDynamicContent(ob_get_clean(), $this->getDynamicPlaceholders());
        return;
    }

    $response = Yii::$app->getResponse();
    $response->off(Response::EVENT_AFTER_SEND, [$this, 'cacheResponse']);
    $data = [
        'cacheVersion' => static::PAGE_CACHE_VERSION,
        'cacheData' => is_array($beforeCacheResponseResult) ? $beforeCacheResponseResult : null,
        'content' => ob_get_clean(),
    ];
    if ($data['content'] === false || $data['content'] === '') {
        return;
    }

    $data['dynamicPlaceholders'] = $this->getDynamicPlaceholders();
    foreach (['format', 'version', 'statusCode', 'statusText'] as $name) {
        $data[$name] = $response->{$name};
    }
    $this->insertResponseCollectionIntoData($response, 'headers', $data);
    $this->insertResponseCollectionIntoData($response, 'cookies', $data);
    $this->cache->set($this->calculateCacheKey(), $data, $this->duration, $this->dependency);
    $data['content'] = $this->updateDynamicContent($data['content'], $this->getDynamicPlaceholders());
    echo $data['content'];
}
```
