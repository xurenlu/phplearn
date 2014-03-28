最近在尝试跟进线上的日志报错,发现线上环境一个很奇怪的现象,就是error_log中总是只出现了
栈信息,却没有REQUEST_URI,RERERER 等相关上下文信息,导致错误不知道如何重现.而有时候,错误又
似乎被重复打印,格式也不统一.
经过跟进,发现现上日志配置似乎有问题,为了彻底弄个明白,我决定在自己的电脑上也跑一个nginx+
fpm+php的环境,研究一下日志输出.

##试验环境
以下是试验环境:

####php 
php 版本:5.4.21
php.ini的配置:
```
log_errors=on
error_log="/tmp/php_errors.log"
```

####php-fpm
php-fpm.conf的配置如下:
```
catch_workers_output=yes
error_log=/tmp/fpm.log
```
#### nginx
nginx的配置:
```
error_log /tmp/nginx_error.log notice;
```

再实验开始之前,去复习了一下catch_workers_output 的定义

####catch_workers_output 

参见:[http://www.php.net/manual/zh/install.fpm.configuration.php]:
```
catch_workers_output boolean
重定向运行过程中的 stdout 和 stderr 到主要的错误日志文件中。
如果没有设置，stdout 和 stderr 将会根据 FastCGI 的规则被重定向到 /dev/null。默认值：无。
```

## 线上脚本
我们从线上抓回来一段有输出的代码,然后经过一点点修改,尽可能地简单,来做为试验脚本:
```php
<?php
//err.php 访问地址:http://localhost/err.php
define("TSF_TRACE_LEVEL",0);
 function handleError($code, $message, $file, $line) {
        global $tsf_last_log;
        $log_prefix = "[tsf]";
        if ($code) {
            restore_error_handler();
            restore_exception_handler();
            if(isset($log_prefix)){
                $log = $log_prefix.trim($tsf_last_log)."\n";
            }else{
                $log = "";
            }
            $log .= $log_prefix."$message ($file:$line)\n".$log_prefix."Stack trace:\n";
            $trace = debug_backtrace();
            if (count($trace) > TSF_TRACE_LEVEL)
                $trace = array_slice($trace, TSF_TRACE_LEVEL);
            foreach ($trace as $i => $t) {
                if (!isset($t['file']))
                    $t['file'] = 'unknown';
                if (!isset($t['line']))
                    $t['line'] = 0;
                if (!isset($t['function']))
                    $t['function'] = 'unknown';
                $log .= $log_prefix. "#$i {$t['file']}({$t['line']}): ";
                if (isset($t['object']) && is_object($t['object']))
                    $log .= get_class($t['object']) . '->';
                $log .= "{$t['function']}()\n";
            }
            if (isset($_SERVER['REQUEST_URI']))
                $log .= $log_prefix. '[QUERY] ' . $_SERVER['REQUEST_URI']."\n";
            if ( isset($_SERVER["HTTP_REFERER"]) ){
                $log .= $log_prefix. "[REFERER] " . $_SERVER["HTTP_REFERER"]."\n";
            }
            error_log(trim($log));
        }
    }
    set_error_handler("handleError");

    error_reporting(E_ALL^E_STRICT);

    $arr=null;
    foreach($arr as $k=>$v){

    }

  
```

## 不同日志的格式差异

虽然做为一篇报告,应该先叙述实验过程;不过为了让大家容易记住结论,还是先考虑给第一个结论,
那就是,php,fpm,nginx三个不同的层面,日志的格式是有差别的,同时也还是挺好记的;所以这三种
日志样式,大家要记住能,确保能区分开.因为,接下来我们就要做试验了;
#### php error_log 格式
先看php的error_log的格式:
```
[20-Mar-2014 23:47:25 Asia/Shanghai] [tsf]
[tsf]Invalid argument supplied for foreach() (/home/x/htdocs/localhost/srp/err.php:44)
[tsf]Stack trace:
[tsf]#0 /home/x/htdocs/localhost/srp/err.php(44): handleError()
[tsf][QUERY] /err.php
```
#### fpm error_log 格式
接下来对比看一下fpm的日志,这个就啰嗦一些:
```
[20-Mar-2014 23:35:26] WARNING: [pool www] child 28624 said into stderr: "NOTICE: PHP message: [tsf]"
[20-Mar-2014 23:35:26] WARNING: [pool www] child 28624 said into stderr: "[tsf]Invalid argument supplied for foreach() (/home/x/htdocs/localhost/srp/err.php:44)"
[20-Mar-2014 23:35:26] WARNING: [pool www] child 28624 said into stderr: "[tsf]Stack trace:"
[20-Mar-2014 23:35:26] WARNING: [pool www] child 28624 said into stderr: "[tsf]#0 /home/x/htdocs/localhost/srp/err.php(44): handleError()"
[20-Mar-2014 23:35:26] WARNING: [pool www] child 28624 said into stderr: "[tsf][QUERY] /err.php"
```

#### nginx error_log 
这是nginx 经由fastcgi从上游(fpm)捕获的error的记录.这个记录的内容就更丰富一点,包括了是在哪个URL产生的消息,客户端IP是多少等等.
```
2014/03/20 23:42:30 [error] 1194#0: *3889 FastCGI sent in stderr: "PHP message: [tsf]
[tsf]Invalid argument supplied for foreach() (/home/x/htdocs/localhost/srp/err.php:44)
[tsf]Stack trace:
[tsf]#0 /home/x/htdocs/localhost/srp/err.php(44): handleError()
[tsf][QUERY] /err.php" while reading response header from upstream, client: 127.0.0.1, server: localhost, request: "GET /err.php HTTP/1.1", upstream: "fastcgi://127.0.0.1:9000", host: "localhost"
```

## 不同条件下各个环节的error_log的记录生效情况
接下来,进行我们的实验.我们的实验是,分别调整catch_workers_output为"yes",或"no",然后
再分别让php.ini中指定的error_log文件可写/不可写(模拟php的error_log写入失败的情况),看看
各个error_log文件,是否能成功记录;
实验的结论如下表:

![实验结果](http://share.162cm.net/files/201403/85650400.png)

这里我们可以做两个结论:

1. catch_workers_output 设置为yes或no,只能影响fpm自己的log的记录;
2. 如果php.ini所设定的error_log文件可以被写入,那么nginx 和fpm都无法记录php的error_log了;

## nginx和fpm的error_log 为什么记录的信息不全
最后再回到文章开头提过的一个信息:我们的error_log信息总是显示不全.大家回过头看刚才发过的那个php的源代码,会
发现我们是把出错的文件的REQUEST_URI是打出来了的,可是在生产环境,这个信息就几乎从来没有被看到过.
为什么呢?
我只能揣测,是不是fpm丢弃了部分错误信息?
我修改了err.php的源代码,还是保持php.ini中定义的error_log文件不可写入(sudo chown root /tmp/php_errors.php ,因为
php-fpm是用我自己的帐号起动的,改为root,php进程就无法写入了):
```
<?php
define("TSF_TRACE_LEVEL",0);
 function handleError($code, $message, $file, $line) {
        global $tsf_last_log;
        $log_prefix = $tmp  = "[tsf]01234012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789";
        $log_prefix .= $tmp;
        $log_prefix .= $tmp;
        $log_prefix .= $tmp;
        $log_prefix .= $tmp;
        $log_prefix .= $tmp;
        $log_prefix .= $tmp;
        $log_prefix .= $tmp;
        $log_prefix .= $tmp;
        //$log_prefix = "[tsf]";
        if ($code) {
            restore_error_handler();
            restore_exception_handler();
            if(isset($log_prefix)){
                $log = $log_prefix.trim($tsf_last_log)."\n";
            }else{
                $log = "";
            }
            $log .= $log_prefix."$message ($file:$line)\n".$log_prefix."Stack trace:\n";
            $trace = debug_backtrace();
            if (count($trace) > TSF_TRACE_LEVEL)
                $trace = array_slice($trace, TSF_TRACE_LEVEL);
            foreach ($trace as $i => $t) {
                if (!isset($t['file']))
                    $t['file'] = 'unknown';
                if (!isset($t['line']))
                    $t['line'] = 0;
                if (!isset($t['function']))
                    $t['function'] = 'unknown';
                $log .= $log_prefix. "#$i {$t['file']}({$t['line']}): ";
                if (isset($t['object']) && is_object($t['object']))
                    $log .= get_class($t['object']) . '->';
                $log .= "{$t['function']}()\n";
            }
            if (isset($_SERVER['REQUEST_URI']))
                $log .= $log_prefix. '[QUERY] ' . $_SERVER['REQUEST_URI']."\n";
            if ( isset($_SERVER["HTTP_REFERER"]) ){
                $log .= $log_prefix. "[REFERER] " . $_SERVER["HTTP_REFERER"]."\n";
            }
            error_log(trim($log));
        }
    }
    set_error_handler("handleError");

    error_reporting(E_ALL^E_STRICT);

    $arr=null;
    foreach($arr as $k=>$v){

    }
```
现在我们在log信息之前,加了一一堆无意义的占位字符,接着访问http://localhost/err.php,关注
nginx error_log中记录的信息:
```
2014/03/23 23:07:11 [error] 1194#0: *4229 FastCGI sent in stderr: "PHP message: [tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
[tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
[tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
[tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
[tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
[tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
[tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678

[tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
[tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
[tsf]0123401234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
[tsf]0" while reading response header from upstream, client: 127.0.0.1, server: localhost, request: "GET /err.php HTTP/1.1", upstream: "fastcgi://127.0.0.1:9000", host: "localhost"
```

清点了一下记录长度,发现被记录下来的信息刚好总是接近1024...看来是似乎有结论了.
插播一个广告:fpm的error_log和nginx的error_log都会把错误信息截断到1024字节,但是php.ini
中指定的/tmp/php_errors.log中就不会截断.
我们翻查了一下fpm的源代码实现:
```c
//php-5.4.18/sapi/fpm/fpm/fpm_stdio.c
static void fpm_stdio_child_said(struct fpm_event_s *ev, short which, void *arg) /* {{{ */
{
    static const int max_buf_size = 1024;
	int fd = ev->fd;
	char buf[max_buf_size];
	struct fpm_child_s *child;
	int is_stdout;
	struct fpm_event_s *event;
	int fifo_in = 1, fifo_out = 1;
	int is_last_message = 0;
	int in_buf = 0;
	int res;
    .....
```
看完这个函数,我们会注意到,原来,fpm的源代码里就写死了,反正就是1024;不管您输出多少,最多只记录1024个字节.好吧....这样还真是跟php一向的不严谨的性格很匹配呢.


一切似乎真相大白,只有一个问题待查,那就是,为啥线上的日志输出如此混乱呢?直接给出php,fpm,nginx的配置,不多说,略有修改:
```
//nginx.conf
error_log /tmp/php_error.log notice;
```

```
//php.ini
;error_log=syslog //注,就是php.ini的error_log是没有修改的;
```

```
//php-fpm.conf
error_log = /tmp/php_error.log 
```

原来,线上的环境是,nginx的error_log和fpm的error_log指向同一个文件的,而php.ini中的error_log
是没有配置的;所以实际上我们看到的所有error_log都是nginx或fpm产生的,都被截断到1024个字节了....
