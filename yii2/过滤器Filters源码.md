## 目录
* [过滤器执行基本流程源码](#过滤器执行基本流程源码)
* [Ajax过滤器](#Ajax过滤器)

# 过滤器执行基本流程源码
要了解过滤器是如何执行的，需要知道yii2的事件和行为概念  
* [Behavior行为源码](yii2/%5B关键概念一%5DBehavior行为源码.md)
* [Events事件源码](yii2/%5B关键概念二%5DEvents事件源码.md)
也需要简单了解下控制中的方法是如何执行的  
- 组件、路由、配置初始化等等  
- 执行base\Controller.php的runAction方法  
- 和过滤器有关的就是执行$this的beforeAction
```
public function runAction($id, $params = [])
{ 
    ....
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
    ....
}
```
