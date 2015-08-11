---
layout:     post
title:      php CI 源码探索(4)
date:       2015-08-11 1:33
summary:    php CI 代码解读(4) Hooks, Config
categories: php CodeIgniter
---

1. ~~如果 log_message 先被调用了 那么加载 `Exceptions` 类 (动态加载，只有在程序需要的时候加载) 副作用就是一起加载了 Log 类~~
2. ~~加载 `Benchmark` ($BM)~~
3. **加载 `Hooks` ($EXT)**
4. **加载 `Config` ($CFG)**
5. 加载 `Utf8` ($UNI)
6. 加载 `URI` ($URI)
7. 加载 `Router` ($RTR)
8. 加载 `Output` ($OUT)
9. 加载 `Security` ($SEC)
10. 加载 `Language` ($LANG)

### Hooks.php

``` php
<?php
class CI_Hooks
{
    // 是否允许 hooks
    public $enabled = false;

    // 装所有的 config/hooks.php
    public $hooks =    array();

    protected $_objects = array();

    // 手动锁
    protected $_in_progress = false;

    public function __construct()
    {
        // 原来在 Hook 中已经注入了 Conifg 单例了
        $CFG =& load_class('Config', 'core');
        log_message('info', 'Hooks Class Initialized');

        // 用户没有说要开启 hooks 那么就不扯淡了
        if ($CFG->item('enable_hooks') === false) {
            return;
        }

        // 取得 hooks file
        if (file_exists(APPPATH.'config/hooks.php')) {
            // 这里用 include 为了允许二笔程序员犯错
            // 只是报 warning
            include(APPPATH.'config/hooks.php');
        }

        if (file_exists(APPPATH.'config/'.ENVIRONMENT.'/hooks.php')) {
            include(APPPATH.'config/'.ENVIRONMENT.'/hooks.php');
        }

        // 可以看到 期待用户的 hooks.php 中提供一个 $hook 数组
        if (! isset($hook) or ! is_array($hook)) {
            return;
        }

        $this->hooks =& $hook;
        $this->enabled = true;
    }

    public function call_hook($which = '')
    {
        // 我靠 静默 fail
        if (! $this->enabled or ! isset($this->hooks[$which])) {
            return false;
        }

        // $hook['pre_controller'][] = array(
        //     'class'    => 'MyClass',
        //     'function' => 'Myfunction',
        //     'filename' => 'Myclass.php',
        //     'filepath' => 'hooks',
        //     'params'   => array('beer', 'wine', 'snacks')
        // );
        // $hook['pre_controller'][] = array(
        //     'class'    => 'MyClass',
        //     'function' => 'Myfunction',
        //     'filename' => 'Myclass.php',
        //     'filepath' => 'hooks',
        //     'params'   => array('beer', 'wine', 'snacks')
        // );
        if (is_array($this->hooks[$which]) && ! isset($this->hooks[$which]['function'])) {
            foreach ($this->hooks[$which] as $val) {
                $this->_run_hook($val);
            }
        } else {
            // $hook['pre_controller'] = array(
            //     'class'    => 'MyClass',
            //     'function' => 'Myfunction',
            //     'filename' => 'Myclass.php',
            //     'filepath' => 'hooks',
            //     'params'   => array('beer', 'wine', 'snacks')
            // );
            //
            // 或者
            // $hook['post_controller'] = function() {
            //         /* do something here */
            // };
            $this->_run_hook($this->hooks[$which]);
        }

        return true;
    }

    protected function _run_hook($data)
    {
        // Closures/lambda functions and array($object, 'method') callables
        // 看上面注释 [$obj, 'method'] 也算是 callable
        if (is_callable($data)) {
            is_array($data)
                // 这里明确告诉 php $data[1] 是个变量
                // 因为不加大括号 php 可能将 $data[1] 当做一个字符串处理了
                ? $data[0]->{$data[1]}()
                : $data();

            return true;
        } elseif (! is_array($data)) {
            return false;
        }

        // -----------------------------------
        // Safety - Prevents run-away loops
        // -----------------------------------

        // If the script being called happens to have the same
        // hook call within it a loop can happen
        // 明白了
        // 由于用户提供的 hooks 可能递归调用 hooks
        // 为了防止这种无限循环 这里采用了一个手动锁
        if ($this->_in_progress === true) {
            return;
        }

        // 找不到 用户提供的 hook file
        if (! isset($data['filepath'], $data['filename'])) {
            return false;
        }

        $filepath = APPPATH.$data['filepath'].'/'.$data['filename'];

        if (! file_exists($filepath)) {
            return false;
        }

        // Determine and class and/or function names
        $class    = empty($data['class']) ? false : $data['class'];
        $function = empty($data['function']) ? false : $data['function'];
        $params   = isset($data['params']) ? $data['params'] : '';

        if (empty($function)) {
            return false;
        }

        // 手动锁
        $this->_in_progress = true;

        // Call the requested class and/or function
        // 将用户的 hook 缓存到 这个类的_object 数组中如果重复调用则直接从内存
        // 中调用
        // 如果没有在内存中找到才去 require file
        if ($class !== false) {
            // The object is stored?
            if (isset($this->_objects[$class])) {
                if (method_exists($this->_objects[$class], $function)) {
                    $this->_objects[$class]->$function($params);
                } else {
                    return $this->_in_progress = false;
                }
            } else {
                class_exists($class, false) or require_once($filepath);

                if (! class_exists($class, false) or ! method_exists($class, $function)) {
                    return $this->_in_progress = false;
                }

                // Store the object and execute the method
                $this->_objects[$class] = new $class();
                $this->_objects[$class]->$function($params);
            }
        } else {
            function_exists($function) or require_once($filepath);

            if (! function_exists($function)) {
                return $this->_in_progress = false;
            }

            $function($params);
        }

        $this->_in_progress = false;
        return true;
    }
}
?>
```

hooks 算是看完了 说实话写得真特么烂。。。 if else if else 都看疯了

``` php
<?php
class CI_Config
{
    // $config 数组 因为是用 reference 传进来的所以是个单例
    public $config = array();

    // 所有的 加载过的 files
    public $is_loaded =    array();

    // 找到 config file 的路径
    public $_config_paths =    array(APPPATH);

    public function __construct()
    {
        $this->config =& get_config();

        // Set the base_url automatically if none was provided
        if (empty($this->config['base_url'])) {
            // The regular expression is only a basic validation for a valid "Host" header.
            // It's not exhaustive, only checks for valid characters.
            if (isset($_SERVER['HTTP_HOST']) && preg_match('/^((\[[0-9a-f:]+\])|(\d{1,3}(\.\d{1,3}){3})|[a-z0-9\-\.]+)(:\d+)?$/i', $_SERVER['HTTP_HOST'])) {
                $base_url = (is_https() ? 'https' : 'http').'://'.$_SERVER['HTTP_HOST']
                    .substr($_SERVER['SCRIPT_NAME'], 0, strpos($_SERVER['SCRIPT_NAME'], basename($_SERVER['SCRIPT_FILENAME'])));
            } else {
                $base_url = 'http://localhost/';
            }

            $this->set_item('base_url', $base_url);
        }

        log_message('info', 'Config Class Initialized');
    }

    public function load($file = '', $use_sections = false, $fail_gracefully = false)
    {
        $file = ($file === '') ? 'config' : str_replace('.php', '', $file);
        $loaded = false;

        foreach ($this->_config_paths as $path) {
            foreach (array($file, ENVIRONMENT.DIRECTORY_SEPARATOR.$file) as $location) {
                $file_path = $path.'config/'.$location.'.php';
                if (in_array($file_path, $this->is_loaded, true)) {
                    return true;
                }

                if (! file_exists($file_path)) {
                    continue;
                }

                include($file_path);

                // 看起来只有那些 用 $config 的 config file 才可以被这个方法调用
                // 1. config.php
                // 2. migration
                // 3. memcached
                // 4. profiler
                // 这4个配置文件会被这个方法加载
                if (! isset($config) or ! is_array($config)) {
                    if ($fail_gracefully === true) {
                        return false;
                    }

                    show_error('Your '.$file_path.' file does not appear to contain a valid configuration array.');
                }

                if ($use_sections === true) {
                    $this->config[$file] = isset($this->config[$file])
                        ? array_merge($this->config[$file], $config)
                        : $config;
                } else {
                    $this->config = array_merge($this->config, $config);
                }

                $this->is_loaded[] = $file_path;
                $config = null;
                $loaded = true;
                log_message('debug', 'Config file loaded: '.$file_path);
            }
        }

        if ($loaded === true) {
            return true;
        } elseif ($fail_gracefully === true) {
            return false;
        }

        show_error('The configuration file '.$file.'.php does not exist.');
    }

    // 拿出 config item
    public function item($item, $index = '')
    {
        if ($index == '') {
            return isset($this->config[$item]) ? $this->config[$item] : null;
        }

        return isset($this->config[$index], $this->config[$index][$item]) ? $this->config[$index][$item] : null;
    }

    // helper
    public function slash_item($item)
    {
        if (! isset($this->config[$item])) {
            return null;
        } elseif (trim($this->config[$item]) === '') {
            return '';
        }

        return rtrim($this->config[$item], '/').'/';
    }

    //  Returns base_url . index_page [. uri_string]
    // -> http://localhost:8000/index.php or localhost:8000/index.php
    // 这个 uri 会是什么还暂时是个迷 后面看到 route 估计能看到
    public function site_url($uri = '', $protocol = null)
    {
        // localhost:8000/
        $base_url = $this->slash_item('base_url');

        if (isset($protocol)) {
            $base_url = $protocol.substr($base_url, strpos($base_url, '://'));
        }

        if (empty($uri)) {
            // localhost:8000/index.php
            return $base_url.$this->item('index_page');
        }

        $uri = $this->_uri_string($uri);

        if ($this->item('enable_query_strings') === false) {
            $suffix = isset($this->config['url_suffix']) ? $this->config['url_suffix'] : '';

            if ($suffix !== '') {
                if (($offset = strpos($uri, '?')) !== false) {
                    $uri = substr($uri, 0, $offset).$suffix.substr($uri, $offset);
                } else {
                    $uri .= $suffix;
                }
            }

            return $base_url.$this->slash_item('index_page').$uri;
        } elseif (strpos($uri, '?') === false) {
            $uri = '?'.$uri;
        }

        return $base_url.$this->item('index_page').$uri;
    }

    // http://localhost:8000/
    // or localhost:8000/
    public function base_url($uri = '', $protocol = null)
    {
        $base_url = $this->slash_item('base_url');

        if (isset($protocol)) {
            $base_url = $protocol.substr($base_url, strpos($base_url, '://'));
        }

        return $base_url.ltrim($this->_uri_string($uri), '/');
    }

    protected function _uri_string($uri)
    {
        // 这个是用来伪造 ?c=controller&m=method&d=directory
        // -> /directory/controller/method
        // 的设置 用得比较少
        // 
        // 正常情况下我们应该是拿到了一个 
        // ['directory', 'controller', 'method'] 这样的数组
        if ($this->item('enable_query_strings') === false) {
            if (is_array($uri)) {
                $uri = implode('/', $uri);
            }
            return trim($uri, '/');
        } elseif (is_array($uri)) {
            // array to query string
            return http_build_query($uri);
        }

        return $uri;
    }

    // item setter
    public function set_item($item, $value)
    {
        $this->config[$item] = $value;
    }
}
?>
```

呼 终于也看完了。

总体来说 CI 对于 config 的管理也是十分混乱 在 application/config 这个问价夹中有无数个 config file 但是很多 config file 里面生存着相同的 $config 数组。这让我有点看不懂了。

按照我的理解如果两个 config 用相同的数组那么两个文件应该合并才对 既然分开了就应该分成两个数组来做统一管理。

好吧，强忍着放弃的冲动继续看
