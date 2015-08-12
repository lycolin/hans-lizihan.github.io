---
layout:     post
title:      php CI 源码探索(3)
date:       2015-08-10 21:43
summary:    php CI 代码解读(3) Log, Exceptions, Benchmark
categories: php CodeIgniter
---

## 1 CodeIgniter 核心类

在 CodeIgniter.php 中整个 CI 程序被彻底点燃(Ignited)

而这个燃料就是 CI 的所有核心类们

顺序是这样的

1. **如果 log_message 先被调用了 那么加载 `Exceptions` 类 (动态加载，只有在程序需要的时候加载) 副作用就是一起加载了 Log 类**
2. **加载 `Benchmark` ($BM)**
3. 加载 `Hooks` ($EXT)
4. 加载 `Config` ($CFG)
5. 加载 `Utf8` ($UNI)
6. 加载 `URI` ($URI)
7. 加载 `Router` ($RTR)
8. 加载 `Output` ($OUT)
9. 加载 `Security` ($SEC)
10. 加载 `Language` ($LANG)

好了现在开始按顺序看 

### `Benchmark`

``` php
<?php
// 简单讲就是有两个优先设置的东西
// 1 {elapsed_tiem} 这个会直接被 替换成真正的 framework elapsed time
// 2 {memory_usage} 这个会被直接 替换成真正的 framework memory usage
class CI_Benchmark 
{

    // 一个数组装着所有的 key => start_time
    public $marker = array();

    // 简单的 setter
    public function mark($name)
    {
        $this->marker[$name] = microtime(TRUE);
    }

    // 给两个时间点 返回一个时间 duration
    public function elapsed_time($point1 = '', $point2 = '', $decimals = 4)
    {
        // 如果没有参数进来就直接返回一个叫做 {elapsed_time} 
        // 的伪 marker
        // framework 会维护这个伪 marker 并且替换成真正的 elapsed time
        if ($point1 === '') {
            return '{elapsed_time}';
        }

        // 找不到 point1 这个 key
        // 返回 空字符串
        if ( ! isset($this->marker[$point1])) {
            return '';
        }

        // 如果没有第二个参数 就直接将其设为现在
        if ( ! isset($this->marker[$point2])) {
            $this->marker[$point2] = microtime(TRUE);
        }

        // 计算两个时间点之间的时间长度
        return number_format($this->marker[$point2] - $this->marker[$point1], $decimals);
    }

    // 直接返回 {memory_usage} 这个伪类
    public function memory_usage()
    {
        return '{memory_usage}';
    }

}
?>
```

看这个类似乎 看不出来什么 但是后面总结下所有的 $BM 的 marker 们就可以明白了

### `Exceptions`

``` php
<?php
class CI_Exceptions
{
    /**
     * Nesting level of the output buffering mechanism
     *
     * @var int
     */
    public $ob_level;

    // 默认中这里有一些 key => value 对
    // 用来做可以话
    public $levels = array(
        E_ERROR           =>    'Error',
        E_WARNING         =>    'Warning',
        E_PARSE           =>    'Parsing Error',
        E_NOTICE          =>    'Notice',
        E_CORE_ERROR      =>    'Core Error',
        E_CORE_WARNING    =>    'Core Warning',
        E_COMPILE_ERROR   =>    'Compile Error',
        E_COMPILE_WARNING =>    'Compile Warning',
        E_USER_ERROR      =>    'User Error',
        E_USER_WARNING    =>    'User Warning',
        E_USER_NOTICE     =>    'User Notice',
        E_STRICT          =>    'Runtime Notice'
    );

    public function __construct()
    {
        // For users confused about getting "1" as a return value 
        // from ob_get_level at the beginning of a script: 
        // this likely means the PHP ini directive "output_buffering" 
        // is not set to off / 0. PHP automatically starts output 
        // buffering for all your scripts if this directive is not off 
        // (which acts as if you called ob_start on the first 
        // line of your script).
        // 
        // @http://php.net/manual/en/function.ob-get-level.php
        // 
        // 一般建议的环境都是把 output-buffering 
        // 这个参数设置成 4096 的 我的开发环境中也是默认这么设置的
        // 简单讲 ob (output buffer) 是将所有的 echo print
        // 之类的函数的输出全都装进一个 buffer 里面 当 php 脚本跑完的时候
        // 统一输出
        // 
        // 查看 buffer 级别主要是像看看是不是在 template 里面出错了
        // 因为往往 template 会开一个 ob_start()
        //
        $this->ob_level = ob_get_level();
        // Note: Do not log messages from this constructor.
        // 这是因为 Exception 这个类被实例化的时机不一定？不知道为什么
    }

    // 这个方法只有在 set_error_handler 中才会被调用
    // 这里参数原本地引用了 原声 set_error_handler 中的所有参数
    public function log_exception($severity, $message, $filepath, $line)
    {
        $severity = isset($this->levels[$severity]) ? $this->levels[$severity] : $severity;
        log_message('error', 'Severity: '.$severity.' --> '.$message.' '.$filepath.' '.$line);
    }

    // 一个 helper 专门实现 error_404 的糖
    public function show_404($page = '', $log_error = true)
    {
        if (is_cli()) {
            $heading = 'Not Found';
            $message = 'The controller/method pair you requested was not found.';
        } else {
            $heading = '404 Page Not Found';
            $message = 'The page you requested was not found.';
        }

        // By default we log this, but allow a dev to skip it
        if ($log_error) {
            log_message('error', $heading.': '.$page);
        }

        echo $this->show_error($heading, $message, 'error_404', 404);
        exit(4); // EXIT_UNKNOWN_FILE
    }

    // 500 的错误页面 show_error helper function 正是调用了这个方法
    public function show_error($heading, $message, $template = 'error_general', $status_code = 500)
    {
        $templates_path = config_item('error_views_path');
        if (empty($templates_path)) {
            // 默认是 /application/views/errors/
            $templates_path = VIEWPATH.'errors'.DIRECTORY_SEPARATOR;
        }

        if (is_cli()) {
            $message = "\t".(is_array($message) ? implode("\n\t", $message) : $message);
            $template = 'cli'.DIRECTORY_SEPARATOR.$template;
        } else {
            set_status_header($status_code);
            $message = '<p>'.(is_array($message) ? implode('</p><p>', $message) : $message).'</p>';
            $template = 'html'.DIRECTORY_SEPARATOR.$template;
        }

        // 还没有想明白为什么这么写
        if (ob_get_level() > $this->ob_level + 1) {
            ob_end_flush();
        }
        ob_start();
        include($templates_path.$template.'.php');
        $buffer = ob_get_contents();
        ob_end_clean();
        return $buffer;
    }

    public function show_exception(Exception $exception)
    {
        $templates_path = config_item('error_views_path');
        if (empty($templates_path)) {
            $templates_path = VIEWPATH.'errors'.DIRECTORY_SEPARATOR;
        }

        $message = $exception->getMessage();
        if (empty($message)) {
            $message = '(null)';
        }

        if (is_cli()) {
            $templates_path .= 'cli'.DIRECTORY_SEPARATOR;
        } else {
            set_status_header(500);
            $templates_path .= 'html'.DIRECTORY_SEPARATOR;
        }

        if (ob_get_level() > $this->ob_level + 1) {
            ob_end_flush();
        }

        ob_start();
        include($templates_path.'error_exception.php');
        $buffer = ob_get_contents();
        ob_end_clean();
        echo $buffer;
    }

    public function show_php_error($severity, $message, $filepath, $line)
    {
        $templates_path = config_item('error_views_path');
        if (empty($templates_path)) {
            $templates_path = VIEWPATH.'errors'.DIRECTORY_SEPARATOR;
        }

        $severity = isset($this->levels[$severity]) ? $this->levels[$severity] : $severity;

        // For safety reasons we don't show the full file path in non-CLI requests
        if (! is_cli()) {
            $filepath = str_replace('\\', '/', $filepath);
            if (false !== strpos($filepath, '/')) {
                $x = explode('/', $filepath);
                $filepath = $x[count($x)-2].'/'.end($x);
            }

            $template = 'html'.DIRECTORY_SEPARATOR.'error_php';
        } else {
            $template = 'cli'.DIRECTORY_SEPARATOR.'error_php';
        }

        // 重构预警！
        // 显然重复了 要是我就解成一个方法了
        if (ob_get_level() > $this->ob_level + 1) {
            ob_end_flush();
        }
        ob_start();
        include($templates_path.$template.'.php');
        $buffer = ob_get_contents();
        ob_end_clean();
        echo $buffer;
    }
}
?>
```

纵观来看这个类有大量的重复代码 实现的并不是很好 不过是可以用得

唯一搞不懂的就是这个 `ob_get_level` 了 没关系 后面继续慢慢看

此外  `log_message` 方法调用了 `log_message` 函数 所以 `Log` 这个核心类也被实例化了

``` php
<?php
class CI_Log
{
    protected $_log_path;

    protected $_file_permissions = 0644;

    // logging 等级
    protected $_threshold = 1;

    protected $_threshold_array = array();

    protected $_date_fmt = 'Y-m-d H:i:s';

    protected $_file_ext;

    // loggoer 是否可以写入 logfile
    protected $_enabled = true;

    // 自定义的一些个等级
    // (不用 const 就导致IDE及其不友好)
    protected $_levels = array('ERROR' => 1, 'DEBUG' => 2, 'INFO' => 3, 'ALL' => 4);

    public function __construct()
    {
        $config =& get_config();

        // 默认: application/logs/*.php
        $this->_log_path = ($config['log_path'] !== '') ? $config['log_path'] : APPPATH.'logs/';
        $this->_file_ext = (isset($config['log_file_extension']) && $config['log_file_extension'] !== '')
            ? ltrim($config['log_file_extension'], '.') : 'php';

        // 如果没有这个 path 就创建
        file_exists($this->_log_path) or mkdir($this->_log_path, 0755, true);

        if (! is_dir($this->_log_path) or ! is_really_writable($this->_log_path)) {
            $this->_enabled = false;
        }

        if (is_numeric($config['log_threshold'])) {
            $this->_threshold = (int) $config['log_threshold'];
        } elseif (is_array($config['log_threshold'])) {
            $this->_threshold = 0;
            // key => value -> value => key
            $this->_threshold_array = array_flip($config['log_threshold']);
        }

        if (! empty($config['log_date_format'])) {
            $this->_date_fmt = $config['log_date_format'];
        }

        if (! empty($config['log_file_permissions']) && is_int($config['log_file_permissions'])) {
            $this->_file_permissions = $config['log_file_permissions'];
        }
    }

    // 最重要的方法 log_message 就是用得这个
    // e.g. $LOG->write_log('error', 'foobar');
    public function write_log($level, $msg)
    {
        if ($this->_enabled === false) {
            return false;
        }

        $level = strtoupper($level);

        // 这段还挺聪明 如果用户没有设置写 Log 那就不用鸟了
        if ((! isset($this->_levels[$level]) or ($this->_levels[$level] > $this->_threshold))
            && ! isset($this->_threshold_array[$this->_levels[$level]])) {
            return false;
        }

        // application/logs/log-2015-08-11.php
        $filepath = $this->_log_path.'log-'.date('Y-m-d').'.'.$this->_file_ext;
        $message = '';

        if (! file_exists($filepath)) {
            $newfile = true;
            // Only add protection to php files
            if ($this->_file_ext === 'php') {
                $message .= "<?php defined('BASEPATH') OR exit('No direct script access allowed'); ?>\n\n";
            }
        }

        // 文件开不了就跪了
        if (! $fp = @fopen($filepath, 'ab')) {
            return false;
        }

        // Instantiating DateTime with microseconds appended to initial date is needed for proper support of this format
        if (strpos($this->_date_fmt, 'u') !== false) {
            $microtime_full = microtime(true);
            $microtime_short = sprintf("%06d", ($microtime_full - floor($microtime_full)) * 1000000);
            $date = new DateTime(date('Y-m-d H:i:s.'.$microtime_short, $microtime_full));
            $date = $date->format($this->_date_fmt);
        } else {
            $date = date($this->_date_fmt);
        }
        // 2015-08-11 1:05:40 --> foobar
        $message .= $level.' - '.$date.' --> '.$msg."\n";

        // 写log 可能有多个进程 所以要给文件上个锁
        // LOCK_EX 是一个非常耗性能的锁 这个锁会阻塞
        // 其他的线程/进程直到这个锁别 LOCK_UN
        // 所以这个 IO 写入会阻塞
        // 
        // 还有一个选项是 LOCK_SH 这个选项是写入不阻塞但是读入阻塞
        // 
        // LOCK_NB 是一个新的 FLAG 可以告诉这个锁不要阻塞
        flock($fp, LOCK_EX);

        for ($written = 0, $length = strlen($message); $written < $length; $written += $result) {
            if (($result = fwrite($fp, substr($message, $written))) === false) {
                break;
            }
        }

        // php 5.3 + 中只要关闭文件 这个锁自然就解除了
        flock($fp, LOCK_UN);
        fclose($fp);

        if (isset($newfile) && $newfile === true) {
            chmod($filepath, $this->_file_permissions);
        }

        return is_int($result);
    }
}
?>
```

费劲啊。

纵观 CI 代码 充满了远古荒蛮时代的 php4 procedure_code 的气息。

整个应用的依赖没有一个很好地管理 所以在 `Exceptions` 这个类里面奇特地将 `Log` 实例化了出来，而且是隐藏式的。

在 `Exception` 类中 大量的重复逻辑，例如 `include template` 相关。解出来一个方法会舒服很多。

最悲剧的一点是在 `Logger` 的处理上面没有迎合市场上非常流行的 `psr-3` 标准，导致 CI 框架在 log 这个方面就跟正规的 `laravel` 和 `sf2` 差了一个量级

最后的这个文件锁 一个超阶的阻塞锁可能在并发较高的时候导致用户等很久也是够拼得。

总之这个框架感觉为了迎合二笔程序员做了太多的不正确的事儿。 虽然阅读源代码可以张不少知识，但是依然掩盖不了这个曾经的 rockstart 已经 '泯然众人矣'
