使用的wm版本为3.5.22

最近发现了一个问题，TcpConnection类是如何被销毁的，在反复翻看源码之后终于找到了答案，不禁感叹一下wm作者的牛逼  

在master进程创建了mainSocket之后，会fork子进程，然后子进程会将mainSocket扔到IO里面，进行read的事件监听  
```
//每个子进程都会将mainSocket扔到IO里面，然后IO去监听read事件
public function resumeAccept()
{
    // Register a listener to be notified when server socket is ready to read.
    if (static::$globalEvent && true === $this->_pauseAccept && $this->_mainSocket) {
        if ($this->transport !== 'udp') {
            static::$globalEvent->add($this->_mainSocket, EventInterface::EV_READ, array($this, 'acceptConnection'));
        } else {
            static::$globalEvent->add($this->_mainSocket, EventInterface::EV_READ, array($this, 'acceptUdpConnection'));
        }
        $this->_pauseAccept = false;
    }
}
```
然后如果有新连接请求进来，read事件会触发回调acceptConnection，具体做的步骤如下
- accept一个新的资源类型new_socket，
- 惊群操作处理
- 实例化TcpConnection类
- 将new_socket扔到IO里面，监听read事件，与之绑定的回调为TcpConnection的非静态方法baseRead
- 将TcpConnection对象保存在TcpConnection类的静态属性connections
- 将TcpConnection对象保存在Worker对象的worker非静态属性connections里面
```
public function acceptConnection($socket)
{
    \set_error_handler(function(){});
    //接受请求
    $new_socket = stream_socket_accept($socket, 0, $remote_address);
    \restore_error_handler();
    //惊群处理
    if (!$new_socket) {
        return;
    }

    // TcpConnection.
    $connection                         = new TcpConnection($new_socket, $remote_address);
    将TcpConnection对象保存在Worker对象的worker非静态属性connections里面
    $this->connections[$connection->id] = $connection;
    $connection->worker                 = $this;
    ....
}
public function __construct($socket, $remote_address = '')
{
    ...
    //将new_socket扔到IO里面，监听read事件，与之绑定的回调为TcpConnection的非静态方法baseRead
    Worker::$globalEvent->add($this->_socket, EventInterface::EV_READ, array($this, 'baseRead'));
    ...
    //将TcpConnection对象保存在TcpConnection类的静态属性connections
    static::$connections[$this->id] = $this;
}
```
此时TcpConnection类引用规则如下
- 自身的$this
- baseRead绑定了一个$this
- TcpConnection类的静态属性connections有一个$this
- Worker对象的worker非静态属性connections有一个$this

如果new_socket有close行为，会触发TcpConnection的destroy方法，然后会有两个unset操作
- unset掉TcpConnection类的静态属性connections的$this
- unset掉Worker对象的worker非静态属性connections的$this

那么问题是自身的$this和baseRead的$this是如何销毁的呢？？？？

核心就在于向IO扔mainSocket和new_socket时候，如果事件被触发，会用call_user_func_array调用
```
//IO select
if ($read) {
    foreach ($read as $fd) {
        $fd_key = (int)$fd;
        if (isset($this->_allEvents[$fd_key][self::EV_READ])) {
            \call_user_func_array($this->_allEvents[$fd_key][self::EV_READ][0],
                array($this->_allEvents[$fd_key][self::EV_READ][1]));
        }
    }
}

if ($write) {
    foreach ($write as $fd) {
        $fd_key = (int)$fd;
        if (isset($this->_allEvents[$fd_key][self::EV_WRITE])) {
            \call_user_func_array($this->_allEvents[$fd_key][self::EV_WRITE][0],
                array($this->_allEvents[$fd_key][self::EV_WRITE][1]));
        }
    }
}

if($except) {
    foreach($except as $fd) {
        $fd_key = (int) $fd;
        if(isset($this->_allEvents[$fd_key][self::EV_EXCEPT])) {
            \call_user_func_array($this->_allEvents[$fd_key][self::EV_EXCEPT][0],
                array($this->_allEvents[$fd_key][self::EV_EXCEPT][1]));
        }
    }
}
```
在第一次触发mainSocket的read事件后，call_user_func_array触发acceptConnection方法，执行结束后TcpConnection本身的$this就已经被销毁了，但是还有其他引用存在  
在资源close后的destroy方法中，会销毁掉TcpConnection类的静态属性connections的$this和worker非静态属性connections的$this  
在new_socket触发read事件后，call_user_func_array触发baseRead方法，执行结束后与baseRead关联的$this也会被销毁  
同样写操作也是使用call_user_func_array调用的，执行结束后也会销毁  

这样当TcpConnection的全部引用都销毁后会触发TcpConnction的魔术方法
```
public function __destruct()
{
    static $mod;
    self::$statistics['connection_count']--;
    if (Worker::getGracefulStop()) {
        if (!isset($mod)) {
            $mod = ceil((self::$statistics['connection_count'] + 1) / 3);
        }

        if (0 === self::$statistics['connection_count'] % $mod) {
            Worker::log('worker[' . \posix_getpid() . '] remains ' . self::$statistics['connection_count'] . ' connection(s)');
        }

        if(0 === self::$statistics['connection_count']) {
            Worker::stopAll();
        }
    }
}
```
