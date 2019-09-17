nginx配置如下
```
server {
  listen 80;
  root /tmp;
  location / {
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    include        fastcgi_params;
  }
}
```
fpm配置为默认配置，其中使用static进程模式，两个worker进程，将pid配置为/tmp/php-fpm-master1.pid，监听listern为127.0.0.1:9000  
在/tmp中创建a.php文件
```
<?php
  sleep(20);
```
在service管理工具的fpm源码中，reload操作是这样的
```
kill -USR2 `cat $php_fpm_PID`
```
请求a.php文件
```
curl 'http://127.0.0.1/a.php'
```
fpm的master进程pid为3349
```
[root@localhost etc]# cat /tmp/php-fpm-master1.pid                                                
3349
```
在curl执行期间reload
```
kill -USR2 `cat /tmp/php-fpm-master1.pid`
```
请求立刻发生502，然后fpm的master的pid发生改变
```
[root@localhost etc]# cat /tmp/php-fpm-master1.pid                                                
4425
```
