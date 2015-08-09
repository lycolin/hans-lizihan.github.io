---
layout:     post
title:      php CI 源码探索(2)
date:       2015-08-07 18:52
summary:    php CI 代码解读(2)
categories: php CodeIgniter
---

`index.php` 已经过去了 现在开始阅读里两个 file 

1. CodeIgniter.php
2. Common.php

## 1 CodeIgniter.php

``` php
<?php
// 这句话我们会在所有的非 index.php 中看到 很恶心
// 我猜测初衷是照顾那些不会 set server 的 php 程序员
defined('BASEPATH') OR exit('No direct script access allowed');

// 这个可以理解
define('CI_VERSION', '3.0.0');

// 载入环境常量
if (file_exists(APPPATH.'config/'.ENVIRONMENT.'/constants.php')) {
  require_once(APPPATH.'config/'.ENVIRONMENT.'/constants.php');
}

// 默认位置在 `application/config/contants.php`
require_once(APPPATH.'config/constants.php');

// 好了 这个是大头 这个 require 加载了许多核心的 helper function
require_once(BASEPATH.'core/Common.php');

?>
```

``` php
<?php defined('BASEPATH') OR exit('No direct script access allowed');

// 这个 helper 还挺好用，因为 CI 处理了很多 php 版本的坑所以被用到的次数挺多的
if ( ! function_exists('is_php')) {
  /**
   * Determines if the current version of PHP is equal to or greater than the supplied value
   *
   * @param string
   * @return  bool  TRUE if the current version is $version or higher
   */
  function is_php($version) 
  {
    // static private var 是只能在这个函数内部访问到的一个指针
    // 这个数组维护了很多的 compare
    // e.g.:
    // [
    //   '5.3': true,
    //   '5.2': true
    // ]
    // 简单暴力还起到了提升性能的作用 (空间换时间)
    static $_is_php;
    $version = (string) $version;

    if ( ! isset($_is_php[$version])) {
      $_is_php[$version] = version_compare(PHP_VERSION, $version, '>=');
    }

    return $_is_php[$version];
  }
}

if ( ! function_exists('is_really_writable')) {
  function is_really_writable($file) 
  {
    // 这些情况下是不需要 ployfill 的 直接原声 API 解决
    if (DIRECTORY_SEPARATOR === '/' && (is_php('5.4') OR ! ini_get('safe_mode'))) {
      return is_writable($file);
    }

    // 这个比较恶心了。看下某一个 folder 真的可以写么如果可以就直接试图写一个新的
    // 然后删了它
    if (is_dir($file)) {
      // mt_rand 是一个性能好点的 random 函数
      $file = rtrim($file, '/').'/'.md5(mt_rand());
      // 'a' 这儿mod 负责在文件不存在的时候 create 出来
      if (($fp = @fopen($file, 'ab')) === FALSE) {
        return FALSE;
      }

      fclose($fp);
      @chmod($file, 0777);
      @unlink($file);
      return TRUE;
    } elseif ( ! is_file($file) OR ($fp = @fopen($file, 'ab')) === FALSE) {
      return FALSE;
    }

    fclose($fp);
    return TRUE;
  }
}

// 这个是整个 Helpers 中最重要的一个 helper function 了。这个是实现了 CI
// 类加载机制源头。
// 实现的方式是返回一个 reference 和用一个静态数组 共同来实现 单例模式
if ( ! function_exists('load_class')) {
  function &load_class($class, $directory = 'libraries', $param = NULL) 
  {
    static $_classes = array();

    // 在静态数组中直接找有没有这个类(这个机制和 laravel container 是一样的)
    // 如果有说明之前已经实例化过了，直接返回这个类。
    if (isset($_classes[$class])) {
      return $_classes[$class];
    }

    $name = FALSE;

    // 在
    // 1. application/libraries 中
    // 2. system/libraries 中
    // 去找 CI_ 开头的文件，如果有就加载进来
    // 当然 $directory 这个参数是可以改的
    foreach (array(APPPATH, BASEPATH) as $path) {
      if (file_exists($path.$directory.'/'.$class.'.php')) {
        $name = 'CI_'.$class;
        // 第二个参数是是否套用 __autoload 机制
        // 默认情况下 CI 并没有很好地使用 autoload
        if (class_exists($name, FALSE) === FALSE) {
          require_once($path.$directory.'/'.$class.'.php');
        }

        break;
      }
    }

    // config_item 是下面定义的函数 简单说就是去读取 application/config/ 中的配置
    if (file_exists(APPPATH.$directory.'/'.config_item('subclass_prefix').$class.'.php')) {
      // 这个 subclass_prefix 的作用是这样的。
      // MY_Controller extends CI_Controller 就是被当成了 `subclass` 啦
      // 所以这个也是被拿进来了
      $name = config_item('subclass_prefix').$class;

      if (class_exists($name, FALSE) === FALSE) {
        require_once(APPPATH.$directory.'/'.$name.'.php');
      }
    }

    // 如果没找到这个类
    if ($name === FALSE) {
      // Note: We use exit() rather then show_error() in order to avoid a
      // self-referencing loop with the Exceptions class
      // 看上面原始文档
      // 发现原来这样 exit 是因为 show_error 调用了这个方法 所以会导致
      // self-referencing loop
      set_status_header(503);
      echo 'Unable to locate the specified class: '.$class.'.php';
      exit(5); // EXIT_UNK_CLASS
    }

    // Keep track of what we just loaded
    // 这个函数在主要维护一个 key => value 击落所有的类名和类的小写名
    is_loaded($class);

    // $param 是传个这个类的 constructor 用得
    // 这里直接将 静态数组 键值 set 好了
    $_classes[$class] = isset($param)
      ? new $name($param)
      : new $name();

    // 直接返回这个类的引用。
    // 
    // 注意因为函数是 refrece 
    // 所以实例 是  &load_class 这样叫出来的
    // 那么 实例的状态变了 static $_classes[$class] 也会跟着变
    // 想法如果在调用`load_class` 这个函数的时候有没有加 `&` 那么实例变化不会影响
    // 内部的 `static $_classes` 这个数组
    return $_classes[$class];
  }
}

// 这个函数放回一个 container 中所有被加载过的类名
if (! function_exists('is_loaded')) {
    function &is_loaded($class = '')
    {
        static $_is_loaded = array();

        if ($class !== '') {
            $_is_loaded[strtolower($class)] = $class;
        }

        return $_is_loaded;
    }
}
?>
```

到此为止就是 CI 的 container 函数。 主要就是两个 

1. load_class
2. is_loaded

``` php
<?php
// 正常来讲 config 中的东西都应该是 `Config` 这个 类中拿出来的
// 奈何在系统启动的时候有时候需要一些 核心 config
// 所以要借助这个函数 提前加载 config
if (! function_exists('get_config')) {
    function &get_config(Array $replace = array())
    {
        static $config;

        // 首次加载
        if (empty($config)) {
            $file_path = APPPATH.'config/config.php';
            $found = false;
            if (file_exists($file_path)) {
                $found = true;
                // 注意: config.php 中有一个数组叫做 $config
                // 在这里 require 进来刚好就直接变成了这个函数的静态变量
                require($file_path);
            }

            // Is the config file in the environment folder?
            if (file_exists($file_path = APPPATH.'config/'.ENVIRONMENT.'/config.php')) {
                require($file_path);
            } elseif (! $found) {
                set_status_header(503);
                echo 'The configuration file does not exist.';
                exit(3); // EXIT_CONFIG
            }

            // Does the $config array exist in the file?
            if (! isset($config) or ! is_array($config)) {
                set_status_header(503);
                echo 'Your config file does not appear to be formatted correctly.';
                exit(3); // EXIT_CONFIG
            }
        }

        // Are any values being dynamically added or replaced?
        foreach ($replace as $key => $val) {
            $config[$key] = $val;
        }

        // 1 没有 replace 的内容
        // 2 有 config 内容 修改
        // 3 返回 reference 所以之前 & get_config() 得到的配置都被改变
        return $config;
    }
}

// 根据 key 值返回 config 的值
if (! function_exists('config_item')) {
    function config_item($item)
    {
        static $_config;

        if (empty($_config)) {
            // 文档不好 不知道这是对哪个版本进行兼容
            $_config[0] =& get_config();
        }

        return isset($_config[0][$item]) ? $_config[0][$item] : null;
    }
}
?>
```

这个就是 config 的设置了。绑定的比较死 要求在 `config.php` 中设置一个 $config 数组才可以

``` php
<?php
// 返回 config/mime.php 的数组
if (! function_exists('get_mimes')) {
    function &get_mimes()
    {
        static $_mimes;

        if (empty($_mimes)) {
            if (file_exists(APPPATH.'config/'.ENVIRONMENT.'/mimes.php')) {
                $_mimes = include(APPPATH.'config/'.ENVIRONMENT.'/mimes.php');
            } elseif (file_exists(APPPATH.'config/mimes.php')) {
                $_mimes = include(APPPATH.'config/mimes.php');
            } else {
                $_mimes = array();
            }
        }

        return $_mimes;
    }
}
?>
```

单纯 require 但是注意这里用得是 reference 所以如果调用的时候也是引用调用那么大家就都改了

``` php
<?php

if (! function_exists('is_https')) {
    function is_https()
    {
        if (! empty($_SERVER['HTTPS']) && strtolower($_SERVER['HTTPS']) !== 'off') {
            return true;
        } elseif (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
            return true;
        } elseif (! empty($_SERVER['HTTP_FRONT_END_HTTPS']) && strtolower($_SERVER['HTTP_FRONT_END_HTTPS']) !== 'off') {
            return true;
        }

        return false;
    }
}

if (! function_exists('is_cli')) {
    function is_cli()
    {
        return (PHP_SAPI === 'cli' or defined('STDIN'));
    }
}
?>
```

一些简单的 `is_*` 方法

``` php
<?php

// 这个 function 非常有用
// 所有的 5** status 都可以用这个 helper 打回去了
// 这个 helper 是从 container 里面拿出来了 core/Exception 这个单例
// 所以是一个简单的 wrapper
if (! function_exists('show_error')) {
    function show_error($message, $status_code = 500, $heading = 'An Error Was Encountered')
    {
        // abs 两个作用
        // 1. string to number
        // 2. 防止二笔程序员真的入一个 -500......
        $status_code = abs($status_code);
        if ($status_code < 100) {
            $exit_status = $status_code + 9; // 9 is EXIT__AUTO_MIN
            // 我迷乱了 这个 if 在查什么？ $status_code 已经 < 100 了所以
            // max(exit_status) 应该是 108 才对啊 (status_code = 99)
            // 所以这段永远都不可能执行
            if ($exit_status > 125) {
                // 125 is EXIT__AUTO_MAX

                $exit_status = 1; // EXIT_ERROR
            }

            $status_code = 500;
        } else {
            $exit_status = 1; // EXIT_ERROR
        }

        // system/core/Exceptions.php
        // class: CI_Exceptions
        // 这个类回头好好看 知道 这是一个封装好了的 error 类就好了
        $_error =& load_class('Exceptions', 'core');
        echo $_error->show_error($heading, $message, 'error_general', $status_code);
        exit($exit_status);
    }
}

// helper function
// 依然直接调用 core/Exceptions 单例
if (! function_exists('show_404')) {
    function show_404($page = '', $log_error = true)
    {
        $_error =& load_class('Exceptions', 'core');
        $_error->show_404($page, $log_error);
        exit(4); // EXIT_UNKNOWN_FILE
    }
}
?>
```

error handle 就这么结束了 简单讲适两个 helper function 实际上起作用的是 `core/Exceptions` 单例

``` php
<?php
// 和 error 一样
// 这个只是一个 helper function 从 core/Log 单例中调用方法
if (! function_exists('log_message')) {
    function log_message($level, $message)
    {
        static $_log;

        if ($_log === null) {
            // references cannot be directly assigned to static variables, so we use an array

            // 依然不知为何要这样写 不知道这是哪个版本的兼容
            // core/Log.php
            // class: CI_Log
            // 回头仔细看
            $_log[0] =& load_class('Log', 'core');
        }

        $_log[0]->write_log($level, $message);
    }
}
?>
```

同样地真正起作用的是 `core/Log` 单例

``` php
<?php
if (! function_exists('set_status_header')) {
    function set_status_header($code = 200, $text = '')
    {
        if (is_cli()) {
            return;
        }

        if (empty($code) or ! is_numeric($code)) {
            show_error('Status codes must be numeric', 500);
        }

        if (empty($text)) {
            is_int($code) or $code = (int) $code;
            $stati = array(
                200    => 'OK',
                201    => 'Created',
                202    => 'Accepted',
                203    => 'Non-Authoritative Information',
                204    => 'No Content',
                205    => 'Reset Content',
                206    => 'Partial Content',

                300    => 'Multiple Choices',
                301    => 'Moved Permanently',
                302    => 'Found',
                303    => 'See Other',
                304    => 'Not Modified',
                305    => 'Use Proxy',
                307    => 'Temporary Redirect',

                400    => 'Bad Request',
                401    => 'Unauthorized',
                403    => 'Forbidden',
                404    => 'Not Found',
                405    => 'Method Not Allowed',
                406    => 'Not Acceptable',
                407    => 'Proxy Authentication Required',
                408    => 'Request Timeout',
                409    => 'Conflict',
                410    => 'Gone',
                411    => 'Length Required',
                412    => 'Precondition Failed',
                413    => 'Request Entity Too Large',
                414    => 'Request-URI Too Long',
                415    => 'Unsupported Media Type',
                416    => 'Requested Range Not Satisfiable',
                417    => 'Expectation Failed',
                422    => 'Unprocessable Entity',

                500    => 'Internal Server Error',
                501    => 'Not Implemented',
                502    => 'Bad Gateway',
                503    => 'Service Unavailable',
                504    => 'Gateway Timeout',
                505    => 'HTTP Version Not Supported'
            );

            if (isset($stati[$code])) {
                $text = $stati[$code];
            } else {
                show_error('No status text available. Please check your status code number or supply your own message text.', 500);
            }
        }

        if (strpos(PHP_SAPI, 'cgi') === 0) {
            header('Status: '.$code.' '.$text, true);
        } else {
            $server_protocol = isset($_SERVER['SERVER_PROTOCOL']) ? $_SERVER['SERVER_PROTOCOL'] : 'HTTP/1.1';
            header($server_protocol.' '.$code.' '.$text, true, $code);
        }
    }
}
?>
```

这个 helper 里面转了所有的 status 和 status_code

``` php
<?php
if (! function_exists('_error_handler')) {
    // 这个 是内部的函数
    // 用来告诉 php 把所有的 error 都按照这个格式写进log
    // @see set_error_handler
    function _error_handler($severity, $message, $filepath, $line)
    {
        $is_error = (((E_ERROR | E_COMPILE_ERROR | E_CORE_ERROR | E_USER_ERROR) & $severity) === $severity);

        // When an error occurred, set the status header to '500 Internal Server Error'
        // to indicate to the client something went wrong.
        // This can't be done within the $_error->show_php_error method because
        // it is only called when the display_errors flag is set (which isn't usually
        // the case in a production environment) or when errors are ignored because
        // they are above the error_reporting threshold.
        //
        // 看上面原始文档 说明这个 set_header 是有必要的
        if ($is_error) {
            set_status_header(500);
        }

        // Should we ignore the error? We'll get the current error_reporting
        // level and add its bits with the severity bits to find out.
        // 人手比对我们设定的 error_level (见 index.php)
        // 是否包含了 截获的 $severity
        // 如果是就继续 否则返回
        if (($severity & error_reporting()) !== $severity) {
            return;
        }

        $_error =& load_class('Exceptions', 'core');
        $_error->log_exception($severity, $message, $filepath, $line);

        // Should we display the error?
        // case sensitive version of str_replace
        // 翻译一下就是如果说 `display_errors` 有 true 或者 1 等等就是要 display 的意思
        if (str_ireplace(array('off', 'none', 'no', 'false', 'null'), '', ini_get('display_errors'))) {
            $_error->show_php_error($severity, $message, $filepath, $line);
        }

        // If the error is fatal, the execution of the script should be stopped because
        // errors can't be recovered from. Halting the script conforms with PHP's
        // default error handling. See http://www.php.net/manual/en/errorfunc.constants.php
        // 
        // 最后退出程序
        if ($is_error) {
            exit(1); // EXIT_ERROR
        }
    }
}

if (! function_exists('_exception_handler')) {
    // 截获未被截获的 exception
    // @see set_exception_handler
    function _exception_handler($exception)
    {
        $_error =& load_class('Exceptions', 'core');
        $_error->log_exception('error', 'Exception: '.$exception->getMessage(), $exception->getFile(), $exception->getLine());

        // Should we display the error?
        if (str_ireplace(array('off', 'none', 'no', 'false', 'null'), '', ini_get('display_errors'))) {
            $_error->show_exception($exception);
        }

        exit(1); // EXIT_ERROR
    }
}

if (! function_exists('_shutdown_handler')) {
    // 这回真的长知识了
    // 1. set_error_handler 这个函数不会截获 php 语言级别的错误
    // 2. error_get_last() 这个函数可以获得任何的最近的 bug
    // 3. 当 php bug 发生的时候自己就到了 set_shutdown_handler() 这个函数里了
    function _shutdown_handler()
    {
        $last_error = error_get_last();
        if (isset($last_error) &&
            ($last_error['type'] & (E_ERROR | E_PARSE | E_CORE_ERROR | E_CORE_WARNING | E_COMPILE_ERROR | E_COMPILE_WARNING))) {
            _error_handler($last_error['type'], $last_error['message'], $last_error['file'], $last_error['line']);
        }
    }
}
?>
```

这其中长知识的点还是有的 虽然这个错误处理相对简陋而且过滤了 E_STRICT 这么有用的 errir handle 不过这真的是我第一次亲眼看到手撸的 语言 hook

``` php
<?php
// regex 这个真。 helper function
if (! function_exists('remove_invisible_characters')) {
    function remove_invisible_characters($str, $url_encoded = true)
    {
        $non_displayables = array();

        // every control character except newline (dec 10),
        // carriage return (dec 13) and horizontal tab (dec 09)
        if ($url_encoded) {
            $non_displayables[] = '/%0[0-8bcef]/';    // url encoded 00-08, 11, 12, 14, 15
            $non_displayables[] = '/%1[0-9a-f]/';    // url encoded 16-31
        }

        $non_displayables[] = '/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]+/S';    // 00-08, 11, 12, 14-31, 127

        do {
            $str = preg_replace($non_displayables, '', $str, -1, $count);
        } while ($count);

        return $str;
    }
}

if (! function_exists('html_escape')) {
    // 防止 xss
    function html_escape($var, $double_encode = true)
    {
        if (empty($var)) {
            return $var;
        }

        if (is_array($var)) {
            return array_map('html_escape', $var, array_fill(0, count($var), $double_encode));
        }

        return htmlspecialchars($var, ENT_QUOTES, config_item('charset'), $double_encode);
    }
}

if (! function_exists('_stringify_attributes')) {
    // 转化为字符串 helper
    function _stringify_attributes($attributes, $js = false)
    {
        $atts = null;

        if (empty($attributes)) {
            return $atts;
        }

        if (is_string($attributes)) {
            return ' '.$attributes;
        }

        $attributes = (array) $attributes;

        foreach ($attributes as $key => $val) {
            $atts .= ($js) ? $key.'='.$val.',' : ' '.$key.'="'.$val.'"';
        }

        return rtrim($atts, ',');
    }
}
?>
```

``` php
<?php
if (! function_exists('function_usable')) {
    /**
     * Function usable
     *
     * Executes a function_exists() check, and if the Suhosin PHP
     * extension is loaded - checks whether the function that is
     * checked might be disabled in there as well.
     *
     * This is useful as function_exists() will return FALSE for
     * functions disabled via the *disable_functions* php.ini
     * setting, but not for *suhosin.executor.func.blacklist* and
     * *suhosin.executor.disable_eval*. These settings will just
     * terminate script execution if a disabled function is executed.
     *
     * The above described behavior turned out to be a bug in Suhosin,
     * but even though a fix was commited for 0.9.34 on 2012-02-12,
     * that version is yet to be released. This function will therefore
     * be just temporary, but would probably be kept for a few years.
     *
     * @link  http://www.hardened-php.net/suhosin/
     * @param string  $function_name  Function to check for
     * @return  bool  TRUE if the function exists and is safe to call,
     *      FALSE otherwise.
     */

    // 看了半天上面的文档
    // 发现这个函数是一个为了 `suhosin` 的拓展填坑的零时函数
    // 暂时不用了解了
    function function_usable($function_name)
    {
        static $_suhosin_func_blacklist;

        if (function_exists($function_name)) {
            if (! isset($_suhosin_func_blacklist)) {
                if (extension_loaded('suhosin')) {
                    $_suhosin_func_blacklist = explode(',', trim(ini_get('suhosin.executor.func.blacklist')));

                    if (! in_array('eval', $_suhosin_func_blacklist, true) && ini_get('suhosin.executor.disable_eval')) {
                        $_suhosin_func_blacklist[] = 'eval';
                    }
                } else {
                    $_suhosin_func_blacklist = array();
                }
            }

            return ! in_array($function_name, $_suhosin_func_blacklist, true);
        }

        return false;
    }
}
?>
```

呼 感觉看完了 `Common.php` 就看完了一大半 CI 的核心了。 下面回到 `CodeIgniter.php` 继续整个 `application`

``` php
<?php
// 给 pre-php 5.4 填坑
if (! is_php('5.4')) {
    ini_set('magic_quotes_runtime', 0);

    if ((bool) ini_get('register_globals')) {
        $_protected = array(
            '_SERVER',
            '_GET',
            '_POST',
            '_FILES',
            '_REQUEST',
            '_SESSION',
            '_ENV',
            '_COOKIE',
            'GLOBALS',
            'HTTP_RAW_POST_DATA',
            'system_path',
            'application_folder',
            'view_folder',
            '_protected',
            '_registered'
        );

        $_registered = ini_get('variables_order');
        foreach (array('E' => '_ENV', 'G' => '_GET', 'P' => '_POST', 'C' => '_COOKIE', 'S' => '_SERVER') as $key => $superglobal) {
            if (strpos($_registered, $key) === false) {
                continue;
            }

            foreach (array_keys($$superglobal) as $var) {
                if (isset($GLOBALS[$var]) && ! in_array($var, $_protected, true)) {
                    $GLOBALS[$var] = null;
                }
            }
        }
    }
}


// 一些 php 方法用来截获 php的 erorr 自己来写 log
set_error_handler('_error_handler');
set_exception_handler('_exception_handler');
register_shutdown_function('_shutdown_handler');

// 回忆在 `index.php` 中有 $assign_to_config 这个选项
// $assign_to_config['subclass_prefix'] 是一个 super global hook
// 如果设置了这个选项这个选项会覆盖所有的 config
// 这个逻辑就在下面
if (! empty($assign_to_config['subclass_prefix'])) {
    get_config(array('subclass_prefix' => $assign_to_config['subclass_prefix']));
}

// composer support !!
// 吓死我了 CI 居然妥协了 composer
// 1. 默认 `application/vendor`
// 2. 用户指定 (绝对路径)
if ($composer_autoload = config_item('composer_autoload')) {
    if ($composer_autoload === true) {
        file_exists(APPPATH.'vendor/autoload.php')
            ? require_once(APPPATH.'vendor/autoload.php')
            : log_message('error', '$config[\'composer_autoload\'] is set to TRUE but '.APPPATH.'vendor/autoload.php was not found.');
    } elseif (file_exists($composer_autoload)) {
        require_once($composer_autoload);
    } else {
        log_message('error', 'Could not find the specified $config[\'composer_autoload\'] path: '.$composer_autoload);
    }
}

// 引入几个核心类的单例(应用层)
// 1. Benchmarks
$BM =& load_class('Benchmark', 'core');
$BM->mark('total_execution_time_start');
$BM->mark('loading_time:_base_classes_start');

// 2. Hooks
$EXT =& load_class('Hooks', 'core');
$EXT->call_hook('pre_system');

// 3. Config
// Config 被很多其他的类依赖
$CFG =& load_class('Config', 'core');

// Do we have any manually set config items in the index.php file?
// 用 index.php 中的 assign_to_config 来覆盖已有的 config
if (isset($assign_to_config) && is_array($assign_to_config)) {
    foreach ($assign_to_config as $key => $value) {
        $CFG->set_item($key, $value);
    }
}

// 全局中设定编码标准
$charset = strtoupper(config_item('charset'));
ini_set('default_charset', $charset);

if (extension_loaded('mbstring')) {
    define('MB_ENABLED', true);
    // mbstring.internal_encoding is deprecated starting with PHP 5.6
    // and it's usage triggers E_DEPRECATED messages.
    // 
    // php 5.6 + 直接去 default_charset 取出来了所以这个特性被废除了
    @ini_set('mbstring.internal_encoding', $charset);
    // This is required for mb_convert_encoding() to strip invalid characters.
    // That's utilized by CI_Utf8, but it's also done for consistency with iconv.
    mb_substitute_character('none');
} else {
    define('MB_ENABLED', false);
}

// There's an ICONV_IMPL constant, but the PHP manual says that using
// iconv's predefined constants is "strongly discouraged".
if (extension_loaded('iconv')) {
    define('ICONV_ENABLED', true);
    // iconv.internal_encoding is deprecated starting with PHP 5.6
    // and it's usage triggers E_DEPRECATED messages.
    @ini_set('iconv.internal_encoding', $charset);
} else {
    define('ICONV_ENABLED', false);
}

if (is_php('5.6')) {
    ini_set('php.internal_encoding', $charset);
}

// ployfills
require_once(BASEPATH.'core/compat/mbstring.php');
require_once(BASEPATH.'core/compat/hash.php');
require_once(BASEPATH.'core/compat/password.php');
require_once(BASEPATH.'core/compat/standard.php');

// 4. Utf8
$UNI =& load_class('Utf8', 'core');
// 5. URI
$URI =& load_class('URI', 'core');
// 6. Router
$RTR =& load_class('Router', 'core', isset($routing) ? $routing : null);
// 7. Output
$OUT =& load_class('Output', 'core');

// 如果已经有了 cache 那么就不用继续了 直接加载 cache 就好了
if ($EXT->call_hook('cache_override') === false && $OUT->_display_cache($CFG, $URI) === true) {
    exit;
}

// 8. security (csrf and xss)
$SEC =& load_class('Security', 'core');
// 9. Input
$IN  =& load_class('Input', 'core');
// 10. language
$LANG =& load_class('Lang', 'core');

// Load the base controller class
require_once BASEPATH.'core/Controller.php';

// 这个回头慢慢嚼
function &get_instance()
{
    return CI_Controller::get_instance();
}

// 加载所有核心拓展
// 所以 application/core/MY_*.php 会被自动加载
if (file_exists(APPPATH.'core/'.$CFG->config['subclass_prefix'].'Controller.php')) {
    require_once APPPATH.'core/'.$CFG->config['subclass_prefix'].'Controller.php';
}

// Set a mark point for benchmarking
$BM->mark('loading_time:_base_classes_end');

// 好的我们正式开始 appication 了

// 首先我们看下用户的 request 真的可以映射到相应的 Controller 方法上么?
$e404 = false;

// 注意这里 router 帮我们解决了映射正确的 url 到 controller 的事儿
$class = ucfirst($RTR->class);

// router 也帮我们解决了映射 url 到 method 的事儿
$method = $RTR->method;

// 接下来就是检查这些映射是否真的存在并且合法

// 1. 如果 controller 都不存在 自然 404
if (empty($class) or ! file_exists(APPPATH.'controllers/'.$RTR->directory.$class.'.php')) {
    $e404 = true;
} else {
    require_once(APPPATH.'controllers/'.$RTR->directory.$class.'.php');
    // 2. 如果 Controller 
    // 1) 类名没有写好 
    // 2) method 里面首位是 `_` 非法字符
    // 3) 使用了 CI_Controller 中的方法
    // 404
    if (! class_exists($class, false) or $method[0] === '_' or method_exists('CI_Controller', $method)) {
        $e404 = true;
    } elseif (method_exists($class, '_remap')) {
        // 顾名思义 remap (暂时没看到这个用途 TODO:回头补回来)
        $params = array($method, array_slice($URI->rsegments, 2));
        $method = '_remap';
    }
    // WARNING: It appears that there are issues with is_callable() even in PHP 5.2!
    // Furthermore, there are bug reports and feature/change requests related to it
    // that make it unreliable to use in this context. Please, DO NOT change this
    // work-around until a better alternative is available.
    // 
    // 看起来 5.2 中要给 is_callbale 函数填坑
    // 重点在于如果 3. controller 里面没有这个 method 那么就应该 404 了
    elseif (! in_array(strtolower($method), array_map('strtolower', get_class_methods($class)), true)) {
        $e404 = true;
    }
}

if ($e404) {
    // 调用我们自己定义的 404 page
    if (! empty($RTR->routes['404_override'])) {
        if (sscanf($RTR->routes['404_override'], '%[^/]/%s', $error_class, $error_method) !== 2) {
            $error_method = 'index';
        }

        $error_class = ucfirst($error_class);

        if (! class_exists($error_class, false)) {
            if (file_exists(APPPATH.'controllers/'.$RTR->directory.$error_class.'.php')) {
                require_once(APPPATH.'controllers/'.$RTR->directory.$error_class.'.php');
                $e404 = ! class_exists($error_class, false);
            }
            // Were we in a directory? If so, check for a global override
            elseif (! empty($RTR->directory) && file_exists(APPPATH.'controllers/'.$error_class.'.php')) {
                require_once(APPPATH.'controllers/'.$error_class.'.php');
                if (($e404 = ! class_exists($error_class, false)) === false) {
                    $RTR->directory = '';
                }
            }
        } else {
            $e404 = false;
        }
    }

    // Did we reset the $e404 flag? If so, set the rsegments, starting from index 1
    if (! $e404) {
        $class = $error_class;
        $method = $error_method;

        $URI->rsegments = array(
            1 => $class,
            2 => $method
        );
    } else {
        show_404($RTR->directory.$class.'/'.$method);
    }
}

// 默认参数由第三个 segment 开始传入
if ($method !== '_remap') {
    $params = array_slice($URI->rsegments, 2);
}

// load pre hook
$EXT->call_hook('pre_controller');

// benchmark 启动
$BM->mark('controller_execution_time_( '.$class.' / '.$method.' )_start');

// application 其实就是一个 controller
$CI = new $class();

// load post_controller hook
$EXT->call_hook('post_controller_constructor');

//
// 启动了整个 CI application
// 
// p.s. 觉得这个 & 操作符是没有必要的
// 实例化出来的对象本身就是引用传值了啊
call_user_func_array(array(&$CI, $method), $params);

// benchmark 结束
$BM->mark('controller_execution_time_( '.$class.' / '.$method.' )_end');

// load post controller hook
$EXT->call_hook('post_controller');

// 返回一个 response
if ($EXT->call_hook('display_override') === false) {
    $OUT->_display();
}

// 加载系统级别的 hook
$EXT->call_hook('post_system');
?>
```

呼 终于看完了整个 CI 的核心架构 下面就是时候一个个核心模块去探索了~
