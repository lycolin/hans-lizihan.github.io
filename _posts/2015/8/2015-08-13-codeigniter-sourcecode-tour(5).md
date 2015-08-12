---
layout:     post
title:      php CI 源码探索(4)
date:       2015-08-11 1:33
summary:    php CI 代码解读(4) Hooks, Config
categories: php CodeIgniter
---

1. ~~如果 log_message 先被调用了 那么加载 `Exceptions` 类 (动态加载，只有在程序需要的时候加载) 副作用就是一起加载了 Log 类~~
2. ~~加载 `Benchmark` ($BM)~~
3. ~~加载 `Hooks` ($EXT)~~
4. ~~加载 `Config` ($CFG)~~
5. ~~加载 `Utf8` ($UNI)~~
6. ~~加载 `URI` ($URI)~~
7. **加载 `Router` ($RTR)**
8. 加载 `Output` ($OUT)
9. 加载 `Security` ($SEC)
10. 加载 `Language` ($LANG)

既然开坑了那就不放弃 继续吐血看

``` php
<?php
class CI_Router
{
    public $config;

    public $routes =    array();

    // current class
    public $class = '';

    // current method
    public $method = 'index';

    // sub-directory e.g. application/controllers/feature1/SomeController.php
    public $directory = '';

    public $default_controller;

    /**
     * Translate URI dashes
     *
     * Determines whether dashes in controller & method segments
     * should be automatically replaced by underscores.
     *
     * @var bool
     */
    public $translate_uri_dashes = false;

    /**
     * Enable query strings flag
     *
     * Determines wether to use GET parameters or segment URIs
     *
     * @var bool
     */
    public $enable_query_strings = false;

    public function __construct($routing = null)
    {
        $this->config =& load_class('Config', 'core');
        $this->uri =& load_class('URI', 'core');

        $this->enable_query_strings = (! is_cli() && $this->config->item('enable_query_strings') === true);

        // 感觉这就是整个 routing 的核心了
        $this->_set_routing();

        // Set any routing overrides that may exist in the main index file
        // 还记得那年大明湖畔的 (index.php) 中的 $routing 数组么
        // 这个数组被最后就被传进了这个 constructor
        if (is_array($routing)) {
            if (isset($routing['directory'])) {
                $this->set_directory($routing['directory']);
            }

            if (! empty($routing['controller'])) {
                $this->set_class($routing['controller']);
            }

            if (! empty($routing['function'])) {
                $this->set_method($routing['function']);
            }
        }

        log_message('info', 'Router Class Initialized');
    }

    // --------------------------------------------------------------------

    /**
     * Determines what should be served based on the URI request,
     * as well as any "routes" that have been set in the routing config file.
     */
    protected function _set_routing()
    {
        // Are query strings enabled in the config file? Normally CI doesn't utilize query strings
        // since URI segments are more search-engine friendly, but they can optionally be used.
        // If this feature is enabled, we will gather the directory/class/method a little differently
        // 将允许 query_string 的 query_string 翻译成各种 controller + method + directory
        if ($this->enable_query_strings) {
            // e.g. d='admin'
            $_d = $this->config->item('directory_trigger');
            $_d = isset($_GET[$_d]) ? trim($_GET[$_d], " \t\n\r\0\x0B/") : '';
            if ($_d !== '') {
                $this->uri->filter_uri($_d);
                // $this->directory = 'admin/'
                $this->set_directory($_d);
            }

            $_c = trim($this->config->item('controller_trigger'));
            // 我擦这个逻辑差不多啊
            if (! empty($_GET[$_c])) {
                $this->uri->filter_uri($_GET[$_c]);
                $this->set_class($_GET[$_c]);

                $_f = trim($this->config->item('function_trigger'));
                if (! empty($_GET[$_f])) {
                    $this->uri->filter_uri($_GET[$_f]);
                    $this->set_method($_GET[$_f]);
                }

                // 醉了直接操作 property 还真是少见
                $this->uri->rsegments = array(
                    1 => $this->class,
                    2 => $this->method
                );
            } else {
                $this->_set_default_controller();
            }

            // Routing rules don't apply to query strings and we don't need to detect
            // directories, so we're done here
            return;
        }

        // Load the routes.php file.
        if (file_exists(APPPATH.'config/routes.php')) {
            include(APPPATH.'config/routes.php');
        }

        if (file_exists(APPPATH.'config/'.ENVIRONMENT.'/routes.php')) {
            include(APPPATH.'config/'.ENVIRONMENT.'/routes.php');
        }

        // Validate & get reserved routes
        if (isset($route) && is_array($route)) {
            // 这两个是特殊对待的
            isset($route['default_controller']) && $this->default_controller = $route['default_controller'];
            isset($route['translate_uri_dashes']) && $this->translate_uri_dashes = $route['translate_uri_dashes'];
            unset($route['default_controller'], $route['translate_uri_dashes']);
            // 将 default 和 translate_uri_dashes 这两个过滤
            $this->routes = $route;
        }

        // Is there anything to parse?
        // 两种情况
        // 1. localhost:8000 那么进入 set_default_controller
        // 2. locahost:8000/foo/bar 那么必须要对 path 进行解析
        if ($this->uri->uri_string !== '') {
            $this->_parse_routes();
        } else {
            $this->_set_default_controller();
        }
    }

    protected function _set_default_controller()
    {
        if (empty($this->default_controller)) {
            show_error('Unable to determine what should be displayed. A default route has not been specified in the routing file.');
        }

        // Is the method being specified?
        // localhost:8000/controller/ -> controller->index()
        if (sscanf($this->default_controller, '%[^/]/%s', $class, $method) !== 2) {
            $method = 'index';
        }

        if (! file_exists(APPPATH.'controllers/'.$this->directory.ucfirst($class).'.php')) {
            // This will trigger 404 later
            return;
        }

        $this->set_class($class);
        $this->set_method($method);

        // Assign routed segments, index starting from 1
        $this->uri->rsegments = array(
            1 => $class,
            2 => $method
        );

        log_message('debug', 'No URI present. Default controller set.');
    }

    /**
     * Parse Routes
     *
     * Matches any routes that may exist in the config/routes.php file
     * against the URI to determine if the class/method need to be remapped.
     */
    protected function _parse_routes()
    {
        // Turn the segment array into a URI string
        // 命名可以把这 implode 封装进 $URI 更好读得
        // foo/bar
        $uri = implode('/', $this->uri->segments);

        // Get HTTP verb
        $http_verb = isset($_SERVER['REQUEST_METHOD']) ? strtolower($_SERVER['REQUEST_METHOD']) : 'cli';

        // Is there a literal match?  If so we're done
        // 正好碰上了 (没有 regex :num 这些东西的情况)
        if (isset($this->routes[$uri])) {
            // Check default routes format
            if (is_string($this->routes[$uri])) {
                $this->_set_request(explode('/', $this->routes[$uri]));
                return;
            }
            // Is there a matching http verb?
            elseif (is_array($this->routes[$uri]) && isset($this->routes[$uri][$http_verb])) {
                $this->_set_request(explode('/', $this->routes[$uri][$http_verb]));
                return;
            }
        }

        // Loop through the route array looking for wildcards
        foreach ($this->routes as $key => $val) {
            // Check if route format is using http verb
            if (is_array($val)) {
                if (isset($val[$http_verb])) {
                    $val = $val[$http_verb];
                } else {
                    continue;
                }
            }

            // Convert wildcards to RegEx
            // aha 终于被逮到了 没有什么魔法 都是 regex
            $key = str_replace(array(':any', ':num'), array('[^/]+', '[0-9]+'), $key);

            // Does the RegEx match?
            // 我擦 为什么这里用 # 做开头结尾。。。。。。。。
            if (preg_match('#^'.$key.'$#', $uri, $matches)) {
                // Are we using callbacks to process back-references?
                if (! is_string($val) && is_callable($val)) {
                    // Remove the original string from the matches array.
                    array_shift($matches);

                    // Execute the callback using the values in  matches as its parameters.
                    $val = call_user_func_array($val, $matches);
                }
                // Are we using the default routing method for back-references?
                // 分组 (\d+) + $1 -> 原始的 regex
                elseif (strpos($val, '$') !== false && strpos($key, '(') !== false) {
                    $val = preg_replace('#^'.$key.'$#', $val, $uri);
                }

                $this->_set_request(explode('/', $val));
                return;
            }
        }

        // If we got this far it means we didn't encounter a
        // matching route so we'll set the site default route
        $this->_set_request(array_values($this->uri ->segments));
    }

    protected function _set_request($segments = array())
    {
        $segments = $this->_validate_request($segments);
        // If we don't have any segments left - try the default controller;
        // WARNING: Directories get shifted out of the segments array!
        if (empty($segments)) {
            $this->_set_default_controller();
            return;
        }

        if ($this->translate_uri_dashes === true) {
            // class
            $segments[0] = str_replace('-', '_', $segments[0]);
            // method
            if (isset($segments[1])) {
                $segments[1] = str_replace('-', '_', $segments[1]);
            }
        }

        $this->set_class($segments[0]);
        if (isset($segments[1])) {
            $this->set_method($segments[1]);
        } else {
            $segments[1] = 'index';
        }

        array_unshift($segments, null);
        unset($segments[0]);
        $this->uri->rsegments = $segments;
    }

    // 小型验证 确定确实有这个 controller
    protected function _validate_request($segments)
    {
        $c = count($segments);
        // Loop through our segments and return as soon as a controller
        // is found or when such a directory doesn't exist
        while ($c-- > 0) {
            $test = $this->directory
                .ucfirst($this->translate_uri_dashes === true ? str_replace('-', '_', $segments[0]) : $segments[0]);

            if (! file_exists(APPPATH.'controllers/'.$test.'.php') && is_dir(APPPATH.'controllers/'.$this->directory.$segments[0])) {
                $this->set_directory(array_shift($segments), true);
                continue;
            }

            return $segments;
        }

        // This means that all segments were actually directories
        return $segments;
    }

    public function set_class($class)
    {
        $this->class = str_replace(array('/', '.'), '', $class);
    }

    public function set_method($method)
    {
        $this->method = $method;
    }

    public function set_directory($dir, $append = false)
    {
        if ($append !== true or empty($this->directory)) {
            $this->directory = str_replace('.', '', trim($dir, '/')).'/';
        } else {
            $this->directory .= str_replace('.', '', trim($dir, '/')).'/';
        }
    }
}

?>
```

也不长 基本都是些处理字符串的逻辑。

虽然没有仔细研究但是感觉还是乱乱的 因为感觉很多函数都被各种 trim 过很多次了

总结过来就是这个类总的来说就是将合适的http verb赋值给合适的route

route 可以是
1. 回调
2. path/to/controller/method/params

另外要注意的是在 index.php 里面的 $route 可以 override 所有在 config 里面的 route (感觉很奇怪，应该是后来不知道什么时候偶然加上去的)
