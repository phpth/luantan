# php中的线程和进程性能对比
> 前言 

 - 为什么要写这么一篇懒文呢，起因是偶然看到php手册上对php7.2的特性表述中，有一项不是很容易引起人们注意的地方：做到了真正的线程安全。
 大家都知道，线程安全在多线程编程中是一个非常重要的概念，线程安全环境中，线程操作变量等一些共享数据是安全的，互不影响的。
 - 在php线程安全版本中， 有些共享资源的操作也是线程安全的，这就引发了一个疑问，为啥在其他编程语言中线程操作公共资源是不安全的，但是在php中怎么变成安全的呢？
 这就要说到php实现线程安全的核心：TSRM（线程安全资源管理器）。它关注的是php中线程对共享资源的访问（通常都是隐式处理，比如自动加上互斥锁，全局变量置空等操作）。
 下面是一个TSRM对全局变量处理的案列，作为对比加上了多进程的案列:
```php
<?php
    global $a;
    $a = 'test_thread_global_var';
    //创建子线程，在子线程中打印全局变量
    $thread = new class extends Thread{
        public function run() {
            global $a ;
            echo '线程全局变量：';
            var_export ( $a);
             echo PHP_EOL;
        }
    };
    $thread->start();
    $thread->join();
    
   
    // 创建子进程，在子进程中打印全局变量
    $pid = pcntl_fork ();
    if($pid >0 ) {
        pcntl_wait ( $status);
        echo PHP_EOL;
    }
    else {
        global $a ;
        echo '进程全局变量：';
        var_export($a);
    }
    
    
```

 输出结果如下
```text   
线程全局变量：NULL
进程全局变量：'test_thread_global_var'
```

 所以 TSRM对性能的影响有多少呢？这就引出了本文的意义。
 
 #### 测试准备
 
 > 测试环境
 
 - linux环境下分别编译安装zts和nts版本php-7.2.9,在同一台电脑上保证配置信息和加载的扩展一致（zts会多一个pthreads），zts版本信息：
 ```txt
 root@8cca59230230:# $php -v
 PHP 7.2.9 (cli) (built: Sep 01 2018 17:18:53) ( ZTS )
 Copyright (c) 1997-2018 The PHP Group
 Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
 
 root@8cca59230230:/# $php -m
 [PHP Modules]
 bcmath
 Core
 ctype
 curl
 date
 dom
 fileinfo
 filter
 gd
 hash
 iconv
 json
 libxml
 mbstring
 mongodb
 mysqli
 mysqlnd
 openssl
 pcntl
 pcre
 PDO
 pdo_mysql
 pdo_sqlite
 Phar
 posix
 pthreads
 Reflection
 session
  
 [Zend Modules]
 
 root@8cca59230230:/#
  
 ```
 
 - nts版本php信息
 ```txt
 root@8cca59230230:/# php -v
 PHP 7.2.9 (cli) (built: Sep 01 2018 16:47:29) ( NTS )
 Copyright (c) 1997-2018 The PHP Group
 Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
 
 root@8cca59230230:/# php -m
 [PHP Modules]
 bcmath
 Core
 ctype
 curl
 date
 dom
 fileinfo
 filter
 gd
 hash
 iconv
 json
 libxml
 mbstring
 mongodb
 mysqli
 mysqlnd
 openssl
 pcntl
 pcre
 PDO
 pdo_mysql
 pdo_sqlite
 Phar
 posix
 Reflection
 session
 
 [Zend Modules]
 
 root@8cca59230230:/#
 
 ```
 
 > 测试脚本 
 
 - 我们的基准测试中，主要是对比在单位时间内进程和线程创建的数量。
 
 - 线程测试脚本 BenchmarkThread.php 
 ```php
<?php
 $max = @$argv[1] ? $argv[1] : 100;
 $sample = @$argv[2] ? $argv[2] : 5;
 
 printf("Start(%d) ...", $max);
 $it = 0;
 do {
     $s = microtime(true);
     /* begin test */
     $ts = [];
     while (count($ts)<$max) {
         $t = new class extends Thread{
             public function run(){}
         };
         $t->start();
         $ts[]=$t;
     }
     $ts = [];
     /* end test */
     
     $ti [] = $max/(microtime(true)-$s);
     printf(".");
 } while ($it++ < $sample);
 
 printf(" %.3f tps\n", array_sum($ti) / count($ti));
 ```
 - 进程测试脚本 BenchmarkProcess.php
 ```php
<?php 
$max = @$argv[1] ? $argv[1] : 100;
$sample = @$argv[2] ? $argv[2] : 5;

printf("Start(%d) ...", $max);
$it = 0;
do {
    $s = microtime(true);
    /* begin test */
    $ts = [];
    while (count($ts)<$max) {
        $t = pcntl_fork ();
        if($t>0)
        {
            $ts[]=$t;
        }
        else
        {
            //sleep(10);
            goto end ;
        }
        $ts[]=$t;
    }
    $ts = [];
    /* end test */

    $ti [] = $max/(microtime(true)-$s);
    printf(".");
} while ($it++ < $sample);

printf(" %.3f tps\n", array_sum($ti) / count($ti));

end:

 ```
 - 脚本说明：分为五组，每组开启一百线程/进程，然后再取平均值
 
 > 运行测试
 
 说明：$php 是我个人配置的线程安全版本的php二进制文件路径，php 是非线程安全版本可执行文件环境变量
 
 线程安全版本测试线程
 ```text
root@8cca59230230:/# $php BenchmarkThread.php 
Start(100) ......... 730.530 tps

```

非线程安全版本测试进程

```text
root@8cca59230230:/# php BenchmarkProcess.php 
Start(100) ......... 1135.717 tps

```

对比测试结果可以发现：两个版本中，线程创建开销远大于进程创建开销，多线程创建性能是多进程创建性能的64.32% ，TSRM 对性能的影响超过了30%。在高性能编程中这个数值还是很可观的。
另外的，我们在线程安全版本去运行两个测试脚本又会得到什么结果呢？看结果：

线程安全版本测试线程

```text
root@8cca59230230:/# $php BenchmarkThread.php 
Start(100) ......... 724.188 tps

```
 
线程安全版本测试进程

```text
root@8cca59230230:/# $php BenchmarkProcess.php 
Start(100) ......... 998.370 tps

``` 

从结果中看出创建线程的性能是是创建进程的72.55% ，性能消耗也达到了27.45%。我们知道Apache与php组合开发web的时候有一种通信方式是mpm方式，
就是把php作为Apache的模块来提供动态Web服务，由于Apache是多线程模型，所以php也必须是zts线程安全版本。

那么在普通应用中zts版本和nts版本性能的差异又是多少呢？我们在线程安全版本和非线程安全版本运行BenchmarkProcess.php脚本来模拟普通web应用开发来测试性能

线程安全版本运行 BenchmarkProcess.php
```text
root@8cca59230230:/# $php BenchmarkProcess.php 
Start(100) ......... 1005.291 tps

```

非线程安全版本运行 BenchmarkProcess.php
```text
root@8cca59230230:/# php BenchmarkProcess.php
Start(100) ......... 1169.120 tps

```

结果：nts版本的php普通程序性能只有nts版本的php普通程序性能的 85.59%， 性能损失也有15%左右。这都是TSRM的代价

> 结束语

虽然php7.2最大的特性就是实现了真正的线程安全，但是这也不代表pthreads就是我们的最佳选择。另外懒文的意思就是少说话看结果。（大牛别打，逃...）

