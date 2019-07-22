其实事件就是将一个回调或者是类方法存在一个类属性里面(附加行为)，可以去删除某个附加的行为(删除行为)，也可以执行这个行为(执行回调或者类方法)  
模拟一个简单的事件，原理和yii的一样
```
class obj{
	public $events = [];

	public function on($name,$func){
		$this->events[$name] = $func;
	}

	public function off($name){
		if(isset($this->events[$name])){
			unset($this->events[$name]);
		}
	}

	public function trigger($name){
		call_user_func($this->events[$name]);
	}
}
$func = function(){
	echo 1;
};
$obj = new Obj();
$obj -> on("a",$func);
$obj -> on("b",$func);
$obj -> off("b");
$obj -> trigger("a");
```
# 附加事件处理器
控制器中代码例子如下
```
public function actionE(){
    $func = function(){
        echo 1;
    };
    $this->on("a",[TestController::class,"eventTestStatic"]);
    $this->on("b",[$this,"eventTest"]);
    $this->on("c",function(){
        echo 2;
    });
    $this->on("c",$func);
    $this->trigger("a");
    $this->trigger("c");
    $this->off("a",[TestController::class,"eventTestStatic"]);
    $this->off("c",$func);
    return false;
}

public function eventTest(){
    echo __FUNCTION__;
}

public static function eventTestStatic($events){
    echo 3;
}
```
事件注册使用的on方法来附加处理器到事件上，事件的底层代码都Component类上，所以继承了Component的类都会有事件功能
```
public function on($name, $handler, $data = null, $append = true)
{
    //注入行为Behaviors，因为行为类也可能有事件
    $this->ensureBehaviors();
    if (strpos($name, '*') !== false) {
        if ($append || empty($this->_eventWildcards[$name])) {
            $this->_eventWildcards[$name][] = [$handler, $data];
        } else {
            array_unshift($this->_eventWildcards[$name], [$handler, $data]);
        }
        return;
    }

    if ($append || empty($this->_events[$name])) {
        $this->_events[$name][] = [$handler, $data];
    } else {
        array_unshift($this->_events[$name], [$handler, $data]);
    }
}
```
事件使用两个私有属性来保存，一个是_evevts，存储的是所有不带星号的事件；一个是_eventWildcards，存储的是有星号的事件  
区别如下，会输出12  
```
public function actionE(){
  $this->on("test*",function(){
    echo 1;
  });
  $this->on("test_abc",function(){
    echo 2;
  });
  $this->trigger("test_abc");
}
```
存储事件的都是二维数组，也就是说可以附加相同名字的事件，在执行的时候会按照附加的顺序来执行(但是也可以避免，下面会讲)  
删除事件调用的是off方法
```
public function off($name, $handler = null)
{
    //注入行为Behaviors，因为行为类也可能有事件
    $this->ensureBehaviors();
    if (empty($this->_events[$name]) && empty($this->_eventWildcards[$name])) {
        return false;
    }
    //将所有name的事件全部删除
    if ($handler === null) {
        unset($this->_events[$name], $this->_eventWildcards[$name]);
        return true;
    }

    $removed = false;
    // plain event names
    if (isset($this->_events[$name])) {
        foreach ($this->_events[$name] as $i => $event) {
            //删除name中匹配到的事件
            if ($event[0] === $handler) {
                unset($this->_events[$name][$i]);
                $removed = true;
            }
        }
        if ($removed) {
            $this->_events[$name] = array_values($this->_events[$name]);
            return $removed;
        }
    }

    // wildcard event names
    if (isset($this->_eventWildcards[$name])) {
        foreach ($this->_eventWildcards[$name] as $i => $event) {
            if ($event[0] === $handler) {
                unset($this->_eventWildcards[$name][$i]);
                $removed = true;
            }
        }
        if ($removed) {
            $this->_eventWildcards[$name] = array_values($this->_eventWildcards[$name]);
            // remove empty wildcards to save future redundant regex checks:
            if (empty($this->_eventWildcards[$name])) {
                unset($this->_eventWildcards[$name]);
            }
        }
    }

    return $removed;
}
```
这里需要注意一下，比如如下代码
```
public function actionE(){
  $this->on("abcd",function(){echo1;});
  $this->off("abcd",function(){echo1;});//无法将abcd的回调删除
}
```
因为
```
$a = function(){
  echo 1;
};
$b = function(){
  echo 1;
};
var_dump($a===$b);   //false
```
如果要删除回调需要更改附加代码
```
public function actionE(){
  $func = function(){
    echo 1;
  };
  $this->on("abcd",$func);
  $this->off("abcd",$func);
}
```
触发事件为trigger方法
```
public function trigger($name, Event $event = null)
{
    //行为
    $this->ensureBehaviors();

    $eventHandlers = [];
    foreach ($this->_eventWildcards as $wildcard => $handlers) {
    	//正则匹配，如注册了test*事件和test1事件，在调用test1事件时候也会调用test*
        if (StringHelper::matchWildcard($wildcard, $name)) {
	    //事件合并
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
	    //是否执行下一个事件了
            if ($event->handled) {
                return;
            }
        }
    }

    // invoke class-level attached handlers
    Event::trigger($this, $name, $event);
}
```
上面说过事件是按照顺序来执行的，但是可以指定handler为true，如
```
public function actionE(){
    
    $this->on("a",[TestController::class,"eventTestStatic"]);
    $this->on("a",[$this,"eventTest"]);
    $this->trigger("a");
    return false;
}

public function eventTest(){
    echo __FUNCTION__;
}

public static function eventTestStatic($events){
    $events->handled = true;
    echo 1;
}
```
在trigger源码内部，执行了call_user_func后handler会变成true，所以直接就true了  

# 类级别事件处理器
如果需要一个类的所有实例而不是一个指定的实例都响应一个被触发的事件，就需要类级别事件处理器了  
如想要在所有继承于web\Controller的类都能使用a事件，可以写如下代码  
```
use yii\base\Event;
use yii\web\Controller;
...
public function actionE(){
    Event::on(Controller::class,"a",function(){echo 123;});
    Event::trigger(TestController::class,"a");
}
```
相对于实例级别的on方法，类级别的on方法维护的是一个三维数组，并且属性是静态的
```
public static function on($class, $name, $handler, $data = null, $append = true)
{
    $class = ltrim($class, '\\');

    if (strpos($class, '*') !== false || strpos($name, '*') !== false) {
        if ($append || empty(self::$_eventWildcards[$name][$class])) {
            self::$_eventWildcards[$name][$class][] = [$handler, $data];
        } else {
            array_unshift(self::$_eventWildcards[$name][$class], [$handler, $data]);
        }
        return;
    }

    if ($append || empty(self::$_events[$name][$class])) {
        self::$_events[$name][$class][] = [$handler, $data];
    } else {
        array_unshift(self::$_events[$name][$class], [$handler, $data]);
    }
}
```
删除方法也是off，和实例级别的off方法类似
```
public static function off($class, $name, $handler = null)
{
    $class = ltrim($class, '\\');
    if (empty(self::$_events[$name][$class]) && empty(self::$_eventWildcards[$name][$class])) {
        return false;
    }
    if ($handler === null) {
        unset(self::$_events[$name][$class]);
        unset(self::$_eventWildcards[$name][$class]);
        return true;
    }

    // plain event names
    if (isset(self::$_events[$name][$class])) {
        $removed = false;
        foreach (self::$_events[$name][$class] as $i => $event) {
            if ($event[0] === $handler) {
                unset(self::$_events[$name][$class][$i]);
                $removed = true;
            }
        }
        if ($removed) {
            self::$_events[$name][$class] = array_values(self::$_events[$name][$class]);
            return $removed;
        }
    }

    // wildcard event names
    $removed = false;
    if (isset(self::$_eventWildcards[$name][$class])) {
        foreach (self::$_eventWildcards[$name][$class] as $i => $event) {
            if ($event[0] === $handler) {
                unset(self::$_eventWildcards[$name][$class][$i]);
                $removed = true;
            }
        }
        if ($removed) {
            self::$_eventWildcards[$name][$class] = array_values(self::$_eventWildcards[$name][$class]);
            // remove empty wildcards to save future redundant regex checks :
            if (empty(self::$_eventWildcards[$name][$class])) {
                unset(self::$_eventWildcards[$name][$class]);
                if (empty(self::$_eventWildcards[$name])) {
                    unset(self::$_eventWildcards[$name]);
                }
            }
        }
    }

    return $removed;
}
```
类级别还有一个全部删除事件的方法
```
public static function offAll()
{
    self::$_events = [];
    self::$_eventWildcards = [];
}
```
