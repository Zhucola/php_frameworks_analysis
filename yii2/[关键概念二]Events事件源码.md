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
    $this->on("b",[$this,"eventTestStatic"]);
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
  $this->off("abcd",$func);//无法将abcd的回调删除
}
```
# 类级别事件处理器
