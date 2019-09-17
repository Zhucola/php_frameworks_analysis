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
server {
  listern 81;
  root /tmp;
  location / {
    fastcgi_pass 127.0.0.1:9001;
    fastcgi_index index.php;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    include        fastcgi_params;
  }
}
```
配置两个fpm分别为master1和master2
-master1的listen为127.0.0.1:9000，master1的listen为127.0.0.1:9001
-其他均为默认配置
-分别启动master1和master2
```
/usr/local/php/sbin/php-fpm --fpm-config /usr/local/php/etc/master1.conf
/usr/local/php/sbin/php-fpm --fpm-config /usr/local/php/etc/master2.conf
```
在/tmp中创建a.php文件
```
<?php
  $init = curl_init();
  curl_setopt_array($init,[
      CURLOPT_URL => 'http://127.0.0.1/b.php'
  ]);
  curl_exec($curl);
```
在/tmp中创建b.php文件
```
<?php
  sleep(10);
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
难道reload不是平滑重启？？？？？很惊讶，找了各种资料发现这个问题已经有人提给过php官方
https://bugs.php.net/bug.php?id=60961
官方建议使用process_control_timeout来控制信号  
重启配置fpm
```
process_control_timeout=20
request_terminate_timeout=20
```
重启后再次请求a.php，然后在执行期间reload，不会发生502，master进程号不会变  
在reload期间，新请求会阻塞，直到reload结束  
  
在service管理工具中，restart的方式为先stop然后start，至于restart和reload的区别，本人看了下fpm源码，实在是没看懂，留在以后有能力再去研究吧
