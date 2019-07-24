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
- 会调用beforeAction事件，加载行为，将行为中的beforeAction事件也注册进去
- 按beforeAction事件的加载顺序执行事件(先执行的是过滤器的beforeAction事件)
```
yiisoft\yii2\base\Controller.php

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

public function beforeAction($action)
{
    $event = new ActionEvent($action);
    $this->trigger(self::EVENT_BEFORE_ACTION, $event);
    return $event->isValid;
}
```
在执行到trigger时候，也就是说明需要框架执行beforeAction事件，需要调用到Component类的trigger方法
```
public function trigger($name, Event $event = null)
{
    $this->ensureBehaviors();

    $eventHandlers = [];
    foreach ($this->_eventWildcards as $wildcard => $handlers) {
        if (StringHelper::matchWildcard($wildcard, $name)) {
            $eventHandlers = array_merge($eventHandlers, $handlers);
        }
    }

    if (!empty($this->_events[$name])) {
        $eventHandlers = array_merge($eventHandlers, $this->_events[$name]);
    }
    if (!empty($eventHandlers)) {
        if ($event === null) {
            $event = new Event();
        }
        if ($event->sender === null) {
            $event->sender = $this;
        }
        $event->handled = false;
        $event->name = $name;
        foreach ($eventHandlers as $handler) {
            $event->data = $handler[1];
            call_user_func($handler[0], $event);
            // stop further handling if the event is handled
            if ($event->handled) {
                return;
            }
        }
    }

    // invoke class-level attached handlers
    Event::trigger($this, $name, $event);
}
```
可见trigger源码的第一行就是去加载行为，所以如果控制器里面这样定义
```
class TestController extends Controller{
    public function behaviors(){
        return [
            [
                'class' => 'yii\filters\AjaxFilter',
                'only' => ['show', 'view'],
            ]
        ];
    }
}
```
会将AjaxFilter过滤器当成行为加载进去，行为加载后需要执行行为的attach方法去注册行为的事件
```
private function attachBehaviorInternal($name, $behavior)
{
    if($this instanceof Controller){   
    }
    if (!($behavior instanceof Behavior)) {
        $behavior = Yii::createObject($behavior);
    }
    if (is_int($name)) {
        //行为的事件注册进控制器
        $behavior->attach($this);
        //行为加载进控制器
        $this->_behaviors[] = $behavior;
    } else {
        if (isset($this->_behaviors[$name])) {
            //行为加载进控制器
            $this->_behaviors[$name]->detach();
        }
        //行为的事件注册进控制器
        $behavior->attach($this);
        $this->_behaviors[$name] = $behavior;
    }

    return $behavior;
}
```
所有的过滤器都要继承于ActionFilter类，在ActionFilter类中有attach方法，就是重新了行为Behavior的attach
```
public function attach($owner)
{
    //$owner是调用这个过滤器的类
    $this->owner = $owner;
    //将beforeAction事件注册进去
    $owner->on(Controller::EVENT_BEFORE_ACTION, [$this, 'beforeFilter']);
}
```
beforeAction事件注册进去后，调用beforeAction事件会执行beforeFilter方法
```
public function beforeFilter($event)
{
    //判断这个过滤器是否可用，on属性和except属性判断
    if (!$this->isActive($event->action)) {
        return;
    }
    //判断这个过滤器是否可用
    $event->isValid = $this->beforeAction($event->action);
    if ($event->isValid) {
        //将afterAction事件注册进去
        $this->owner->on(Controller::EVENT_AFTER_ACTION, [$this, 'afterFilter'], null, false);
    } else {
        //如果过滤器不可用，那么不会再执行相同事件
        $event->handled = true;
    }
}
```
过滤器是可以执行on和except属性的，on属性可以写明那些方法是可以使用过滤器，except是指明那些方法不可以使用这个过滤器
```
protected function isActive($action)
{
    //获取方法名，如果是actionTest方法，$id会是test
    $id = $this->getActionId($action);
    if (empty($this->only)) {
        $onlyMatch = true;
    } else {
        $onlyMatch = false;
        //正则匹配是否被on属性匹配
        foreach ($this->only as $pattern) {
            if (StringHelper::matchWildcard($pattern, $id)) {
                $onlyMatch = true;
                break;
            }
        }
    }

    $exceptMatch = false;
    //正则匹配是否被except属性匹配
    foreach ($this->except as $pattern) {
        if (StringHelper::matchWildcard($pattern, $id)) {
            $exceptMatch = true;
            break;
        }
    }

    return !$exceptMatch && $onlyMatch;
}
```
每个过滤器都会有beforeAction方法，如Ajax过滤器
```
public function beforeAction($action)
{
    //如果请求不是以ajax发过来的会直接抛异常
    if ($this->request->getIsAjax()) {
        return true;
    }

    throw new BadRequestHttpException($this->errorMessage);
}
```
