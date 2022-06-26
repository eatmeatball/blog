---
title: 关于 windows 下 php_cli 进程 reload 实现
toc: true
date: 2022-06-16 21:49:33
tags:
  - other
  - blog
categories:
  - other

---

这是去年12月，我写在 `workman` 论坛为 `webman` `windows` 下开发热重载的一个脚本，今天在自己博客记录一下。

参考的官网下面的例子，思路和我一开始说的差不多，就是获取pid，然后cmd去kill掉它。proc_open 创建的资源可以直接从父进程中获取执行的pid，所以在不借助其他中间媒介的情况下使用kill命令去kill程序。因为是proc_open是非阻塞的，所以也不会阻挡监控
https://www.php.net/manual/zh/function.proc-terminate.php


<!--more-->



这里写了一个关于windows下 reload webman 代码的实现。其中文件变动部分参照webman安装后的部分。关于唤起和重启部分一开始也考虑的很多方案（cmd start 后台，文件通信，http与监控通信，）结果发现 popen 本身就是可以在 windows 下运行。并且 popen 运行模式也比较干净（1，程序中可直接关闭，2，主程退出后，子程序也同步退出，如果用 start \B 则会产生一个跟随窗口的程序。不会主动退出。并且cmd start \b 比较难以直接获取）
近期有事情，暂不提pr了，有需要的同学可以参考以下狗啃版本(写的太乱了)。

打算将webman自带的监控该为两个，一个负责调度重启，一个负责文件监控。当前调度重启的代码和文件监控合在一起了。暂时不提pr，最近有点其他事情。

效果如下

![](/images/2022/php_reload1.png)

```php
<?php
require_once __DIR__ . '/vendor/autoload.php';

ini_set('display_errors', 'on');
error_reporting(E_ALL);

class M
{
    /**
     * @var array
     */
    protected $_paths = [];

    /**
     * @var array
     */
    protected $_extensions = [];

    public function __construct($monitor_dir, $monitor_extensions)
    {
        $this->_extensions = $monitor_extensions;
        exec('php -v ', $out, $var);
        $this->checkMode = ($var === 0);
    }

    public function check_files_change($monitor_dir)
    {
        static $last_mtime;
        if (!$last_mtime) {
            $last_mtime = time();
        }
        clearstatcache();
        if (!is_dir($monitor_dir)) {
            if (!is_file($monitor_dir)) {
                return;
            }
            $iterator = [new \SplFileInfo($monitor_dir)];
        } else {
            // recursive traversal directory
            $dir_iterator = new \RecursiveDirectoryIterator($monitor_dir);
            $iterator     = new \RecursiveIteratorIterator($dir_iterator);
        }
        foreach ($iterator as $file) {
            /** var SplFileInfo $file */
            if (is_dir($file)) {
                continue;
            }
            if ($file->getFilename() === 'log.php') {
//                echo ($file->getFilename()) . PHP_EOL;
//                echo date('Y-m-d H:i:s', $file->getMTime()) . PHP_EOL;
            }
            // check mtime
            if ($last_mtime < $file->getMTime() && in_array($file->getExtension(), $this->_extensions, true)) {
                $var = 0;
                if ($this->checkMode) {
                    $phpBin = PHP_BINARY;
                    exec('"' . $phpBin . '" -l ' . $file, $out, $var);
                    var_dump($var, $out);
                } else {
                    exec(PHP_BINARY . " -l " . $file, $out, $var);
                }
                var_dump($var);
                if ($var) {
                    $last_mtime = $file->getMTime();
                    continue;
                }
                echo $file . " update and reload\n";
                // send SIGUSR1 signal to master process for reload
                $last_mtime = $file->getMTime();
                return true;
            }
        }
        return false;
    }

}

//$str = <<<EOF
//wmic process where "name like '%php%' and commandline like '%[start.php]%' " get processid
//EOF;

//exec($run,$out,$var);
//exec("start /b php start.php",$out, $var);
//exec("taskkill /f /t /im php.exe");
//var_dump($out,$var);

$m = new M([
    __DIR__ . '/app',
    __DIR__ . '/config',
    __DIR__ . '/database',
    __DIR__ . '/process',
    __DIR__ . '/resource',
    __DIR__ . '/support',
], [
    'php', 'html', 'htm', 'env'
]);

$phpBin = '"' . PHP_BINARY . '"';
// 用 cmd 后台启动web服务
//exec('start /c php start.php  start ', $out, $var);

//exec("start /b php start.php",$out, $var);
//var_dump($out,$var);
$descriptorspec = [STDIN, STDOUT, STDOUT];

$run = 'php  start.php start';
$resource = proc_open('php  start.php start', $descriptorspec, $pipes);

//var_dump($pipes);

pln("run");
while (true) {
    $r = $m->check_files_change(__DIR__ . '/');
//    var_dump($r);
//    exec("taskkill /f /t /im php.exe");
    if ($r) {
//        exec("taskkill  /f /pid $pid", $out, $var);
//        pln("kill");
//        fwrite($pipes[0], 'die');
//        proc_terminate($resource);
//        proc_close($resource);
        $pstatus = proc_get_status($resource);
        $PID = $pstatus['pid'];
        kill($PID); // instead of proc_terminate($resource);
//        fclose($pipes[0]);
//        fclose($pipes[1]);
//        fclose($pipes[2]);
        proc_close($resource);
        $resource = proc_open('php  start.php start', $descriptorspec, $pipes);
        pln("restart");
    }
    sleep(1);
}
function pln($data)
{
    echo $data . PHP_EOL;
}

function kill($pid){
    return stripos(php_uname('s'), 'win')>-1  ? exec("taskkill /F /T /PID $pid") : exec("kill -9 $pid");
}
```


这应该之跑了一个不知道为啥windows下显示跑了俩web进程。不过qps是2w。同时可以reloading了。这样就可以使得 win下的php 在引用更多的依赖的前提下，高效开发了（比如引入node，py，go的文件监控来控制php）。

![](/images/2022/php_reload1.png)

如果着急用的同学可以把下面代码直接放到webman的跟目录，在win下可以直接运行

```php
<?php
require_once __DIR__ . '/vendor/autoload.php';
use Webman\Config;
use Dotenv\Dotenv;
ini_set('display_errors', 'on');
error_reporting(E_ALL);

/**
 * 1、win下可以使用 taskkill /F /T /PID $pid 精确杀死进程，如果通过执行程序名则会出现误杀现象
 * 2、win下路由可能存在空格，所以可执行文件路径需要加入引号包裹，以防止被完整路径无法被cmd识别
 * 3、win下可以使用 start http://xxx.xxx.xxx.xx:xx/ 可直接唤起默认浏览器
 */
if(!stristr(PHP_OS, 'WIN')){
    echo "请在windows下使用本程序";
    return;
}
if (class_exists('Dotenv\Dotenv') && file_exists(base_path().'/.env')) {
    if (method_exists('Dotenv\Dotenv', 'createUnsafeImmutable')) {
        Dotenv::createUnsafeImmutable(base_path())->load();
    } else {
        Dotenv::createMutable(base_path())->load();
    }
}

Config::load(config_path(), ['route', 'container']);

if ($timezone = config('app.default_timezone')) {
    date_default_timezone_set($timezone);
}

$config                               = config('server');
$pUrl = parse_url($config['listen']);

$localUrl = 'http://127.0.0.1'.(isset($pUrl['port'])?':'.$pUrl['port']:'');

class M
{
    /**
     * @var array
     */
    protected $_paths = [];

    /**
     * @var array
     */
    protected $_extensions = [];

    public function __construct($monitor_dir, $monitor_extensions)
    {
        $this->_extensions = $monitor_extensions;
        $this->_paths      = $monitor_dir;
        exec('php -v ', $out, $var);
        $this->checkMode = ($var === 0);
    }

    public function checkAll()
    {
        foreach ($this->_paths as $path) {
            if ($this->check_files_change($path)) {
                return true;
            }
        }
        return false;
    }

    public function check_files_change($monitor_dir)
    {
        static $last_mtime;
        if (!$last_mtime) {
            $last_mtime = time();
        }
        clearstatcache();
        if (!is_dir($monitor_dir)) {
            if (!is_file($monitor_dir)) {
                return false;
            }
            $iterator = [new \SplFileInfo($monitor_dir)];
        } else {
            // recursive traversal directory
            $dir_iterator = new \RecursiveDirectoryIterator($monitor_dir);
            $iterator     = new \RecursiveIteratorIterator($dir_iterator);
        }
        foreach ($iterator as $file) {
            /** var SplFileInfo $file */
            if (is_dir($file)) {
                continue;
            }
            if ($file->getFilename() === 'log.php') {
//                echo ($file->getFilename()) . PHP_EOL;
//                echo date('Y-m-d H:i:s', $file->getMTime()) . PHP_EOL;
            }
            // check mtime
            if ($last_mtime < $file->getMTime() && in_array($file->getExtension(), $this->_extensions, true)) {
                $var = 0;
                if ($this->checkMode) {
                    $phpBin = PHP_BINARY;
                    exec('"' . $phpBin . '" -l ' . $file, $out, $var);
                } else {
                    exec(PHP_BINARY . " -l " . $file, $out, $var);
                }
                if ($var) {
                    $last_mtime = $file->getMTime();
                    continue;
                }
                echo $file . " update and reload\n";
                // send SIGUSR1 signal to master process for reload
                $last_mtime = $file->getMTime();
                return true;
            }
        }
        return false;
    }

}

$m = new M([
    __DIR__ . '/app',
    __DIR__ . '/config',
    __DIR__ . '/database',
    __DIR__ . '/process',
    __DIR__ . '/resource',
    __DIR__ . '/support',
], [
    'php', 'html', 'htm', 'env'
]);

$phpBin         = '"' . PHP_BINARY . '"';
$descriptorspec = [STDIN, STDOUT, STDOUT];

$run      = 'php  start.php start';
$resource = proc_open('php  start.php start', $descriptorspec, $pipes);

exec("start $localUrl");
while (true) {
    $r = $m->checkAll();
    if ($r) {
        $pStatus = proc_get_status($resource);
        $PID     = $pStatus['pid'];
        kill($PID);
        proc_close($resource);
        $resource = proc_open('php  start.php start', $descriptorspec, $pipes);
    }
    sleep(1);
}
function pln($data)
{
    echo $data . PHP_EOL;
}

function kill($pid)
{
    return stripos(php_uname('s'), 'win') > -1 ? exec("taskkill /F /T /PID $pid") : exec("kill -9 $pid");
}
```