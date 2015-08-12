---
layout:     post
title:      php CI 源码探索(4)
date:       2015-08-11 1:33
summary:    php CI 代码解读(4) UTF, URI
categories: php CodeIgniter
---

1. ~~如果 log_message 先被调用了 那么加载 `Exceptions` 类 (动态加载，只有在程序需要的时候加载) 副作用就是一起加载了 Log 类~~
2. ~~加载 `Benchmark` ($BM)~~
3. ~~加载 `Hooks` ($EXT)~~
4. ~~加载 `Config` ($CFG)~~
5. **加载 `Utf8` ($UNI)**
6. **加载 `URI` ($URI)**
7. 加载 `Router` ($RTR)
8. 加载 `Output` ($OUT)
9. 加载 `Security` ($SEC)
10. 加载 `Language` ($LANG)

继续吐血看

首先是 UTF-8 的支援

我并不太了解编码的东西 所以只知道这段是可以把非标准的 ascii 转码成标准 UTF-8

留着注释后面开始研究编码再看吧

``` php
<?php

class CI_Utf8
{

    public function __construct()
    {
        if (
            defined('PREG_BAD_UTF8_ERROR')                // PCRE must support UTF-8
            && (ICONV_ENABLED === true or MB_ENABLED === true)    // iconv or mbstring must be installed
            && strtoupper(config_item('charset')) === 'UTF-8'    // Application charset must be UTF-8
            ) {
            define('UTF8_ENABLED', true);
            log_message('debug', 'UTF-8 Support Enabled');
        } else {
            define('UTF8_ENABLED', false);
            log_message('debug', 'UTF-8 Support Disabled');
        }

        log_message('info', 'Utf8 Class Initialized');
    }

    /**
     * Clean UTF-8 strings
     *
     * Ensures strings contain only valid UTF-8 characters.
     *
     * @param   string  $str    String to clean
     * @return  string
     */
    public function clean_string($str)
    {
        if ($this->is_ascii($str) === false) {
            if (MB_ENABLED) {
                $str = mb_convert_encoding($str, 'UTF-8', 'UTF-8');
            } elseif (ICONV_ENABLED) {
                $str = @iconv('UTF-8', 'UTF-8//IGNORE', $str);
            }
        }

        return $str;
    }

    /**
     * Remove ASCII control characters
     *
     * Removes all ASCII control characters except horizontal tabs,
     * line feeds, and carriage returns, as all others can cause
     * problems in XML.
     *
     * @param   string  $str    String to clean
     * @return  string
     */
    public function safe_ascii_for_xml($str)
    {
        return remove_invisible_characters($str, false);
    }

    /**
     * Convert to UTF-8
     *
     * Attempts to convert a string to UTF-8.
     *
     * @param   string  $str        Input string
     * @param   string  $encoding   Input encoding
     * @return  string  $str encoded in UTF-8 or FALSE on failure
     */
    public function convert_to_utf8($str, $encoding)
    {
        if (MB_ENABLED) {
            return mb_convert_encoding($str, 'UTF-8', $encoding);
        } elseif (ICONV_ENABLED) {
            return @iconv($encoding, 'UTF-8', $str);
        }

        return false;
    }

    /**
     * Is ASCII?
     *
     * Tests if a string is standard 7-bit ASCII or not.
     *
     * @param   string  $str    String to check
     * @return  bool
     */
    public function is_ascii($str)
    {
        return (preg_match('/[^\x00-\x7F]/S', $str) === 0);
    }
}
?>
```

接下来是一个比较重要的核心模块 : URL

这个源代码相对比较长了

``` php
<?php
/**
 * Parses URIs and determines routing
 *
 * 这个是为数不多的有用的 description
 * 这里面要提前明白的一点是 `URI` 这个词汇不是 RFC 的标准
 * 而是根据 `REQUEST_URI` 拿出的 /foo/bar/baz?q=asdf 这条字符串
 */
class CI_URI
{
    /**
     * List of cached URI segments
     *
     * @var array
     */
    public $keyval = array();

    /**
     * Current URI string
     *
     * @var string
     */
    public $uri_string = '';

    /**
     * List of URI segments
     *
     * Starts at 1 instead of 0.
     *
     * @var array
     */
    public $segments = array();

    /**
     * List of routed URI segments
     *
     * Starts at 1 instead of 0.
     *
     * @var array
     */
    public $rsegments = array();

    /**
     * Permitted URI chars
     *
     * PCRE character group allowed in URI segments
     *
     * @var string
     */
    protected $_permitted_uri_chars;

    public function __construct()
    {
        $this->config =& load_class('Config', 'core');

        // 在 enable_query_strings 被允许的情况下
        // 就不需要对 URI 进行解析了
        // 有一个特例 就是用 cli 去叫一个 controller 的时候依然走的是传统的 path 模式
        if (is_cli() or $this->config->item('enable_query_strings') !== true) {
            // 一个 regex 用于过滤非法的 uri 字符
            $this->_permitted_uri_chars = $this->config->item('permitted_uri_chars');

            // 如果是 cli 那么就直接解析 argv
            if (is_cli()) {
                $uri = $this->_parse_argv();
            } else {
                $protocol = $this->config->item('uri_protocol');
                // 我靠这个写法不多见
                // 如果 empty 就会到 赋值
                // 如果不是 empty 就短路了
                // 学习了
                empty($protocol) && $protocol = 'REQUEST_URI';

                // localhost:8000/for?bar=baz
                switch ($protocol) {
                    // AUTO 不知道是几万年前的产物
                    case 'AUTO': // For BC purposes only
                    // /foo?bar=baz
                    case 'REQUEST_URI':
                        $uri = $this->_parse_request_uri();
                        break;
                    // bar=baz
                    case 'QUERY_STRING':
                        $uri = $this->_parse_query_string();
                        break;
                    // /foo
                    case 'PATH_INFO':
                    default:
                        $uri = isset($_SERVER[$protocol])
                            ? $_SERVER[$protocol]
                            : $this->_parse_request_uri();
                        break;
                }

                // 完成了这个switch 之后就直接parse 了
            }

            $this->_set_uri_string($uri);
        }

        log_message('info', 'URI Class Initialized');
    }

    protected function _parse_request_uri()
    {
        if (! isset($_SERVER['REQUEST_URI'], $_SERVER['SCRIPT_NAME'])) {
            return '';
        }

        // 天哪我竟然看到了这种牛叉的函数
        // 'http://username:password@hostname:9090/path?arg=value#anchor'
        // -> [
        //  'scheme' => 'http',
        //  'user'   => 'username',
        //  'pass' => 'password'
        //  'path' => '/path'
        //  'query' => 'arg=value'
        //  'fragment' => 'anchor'
        // ]
        // -> path or query
        // e.g. localhost:8000
        // -> [
        //  'scheme' => 'http'
        //  'host' => 'localhost'
        //  'port' => '8000'
        //  'path' => ''
        // ]
        $uri = parse_url($_SERVER['REQUEST_URI']);
        $query = isset($uri['query']) ? $uri['query'] : '';
        $uri = isset($uri['path']) ? $uri['path'] : '';

        // uri = ''

        // /index.php
        // 在没有开 rewrite 的环境中将 `index.php` 忽略掉
        if (isset($_SERVER['SCRIPT_NAME'][0])) {
            // uri('') 不在 /index.php 路径 里面
            if (strpos($uri, $_SERVER['SCRIPT_NAME']) === 0) {
                // false.toString 其实就是 ''
                $uri = (string) substr($uri, strlen($_SERVER['SCRIPT_NAME']));
            } elseif (strpos($uri, dirname($_SERVER['SCRIPT_NAME'])) === 0) {
                $uri = (string) substr($uri, strlen(dirname($_SERVER['SCRIPT_NAME'])));
            }
        }

        // 不太晓得这段代码在搞毛
        // 暂时略过
        // This section ensures that even on servers that require the URI to be in the query string (Nginx) a correct
        // URI is found, and also fixes the QUERY_STRING server var and $_GET array.

        // 这段if 是 querystring 里面没有 '/' 而且 没有 path
        // 但是后面在搞什么就真的不太明白了
        if (trim($uri, '/') === '' && strncmp($query, '/', 1) === 0) {
            $query = explode('?', $query, 2);
            $uri = $query[0];
            $_SERVER['QUERY_STRING'] = isset($query[1]) ? $query[1] : '';
        } else {
            $_SERVER['QUERY_STRING'] = $query;
        }
        // 这个方法也挺凶 将一个 first=value&arr[]=foo+bar&arr[]=baz
        // 这样的字符串解析成一个数组
        parse_str($_SERVER['QUERY_STRING'], $_GET);

        if ($uri === '/' or $uri === '') {
            return '/';
        }

        // Do some final cleaning of the URI and return it
        return $this->_remove_relative_directory($uri);
    }

    /**
     * Remove relative directory (../) and multi slashes (///)
     */
    protected function _remove_relative_directory($uri)
    {
        $uris = array();
        // 这个函数就是根据 `/` 将字符串做一个 expload
        $tok = strtok($uri, '/');
        while ($tok !== false) {
            // 过滤字符串
            if ((! empty($tok) or $tok === '0') && $tok !== '..') {
                $uris[] = $tok;
            }
            $tok = strtok('/');
        }

        return implode('/', $uris);
    }

    protected function _set_uri_string($str)
    {
        // Filter out control characters and trim slashes
        $this->uri_string = trim(remove_invisible_characters($str, false), '/');

        if ($this->uri_string !== '') {
            // Remove the URL suffix, if present
            if (($suffix = (string) $this->config->item('url_suffix')) !== '') {
                $slen = strlen($suffix);

                if (substr($this->uri_string, -$slen) === $suffix) {
                    $this->uri_string = substr($this->uri_string, 0, -$slen);
                }
            }

            // 因为不知道为什么他想从 1 开始 index
            $this->segments[0] = null;

            // Populate the segments array
            // 我有点搞不明白为什么一直要 trim 这个 uri 感觉 uri 已经被拆分并且
            // 合并了很多次了
            // 写的太烂了。。。。
            foreach (explode('/', trim($this->uri_string, '/')) as $val) {
                $val = trim($val);
                // Filter segments for security
                // 根据用户提供的 regex $config->permitted_uri 来过滤有害的 uri
                $this->filter_uri($val);

                // 将所有的 segment 存进 segment 这个 变量
                if ($val !== '') {
                    $this->segments[] = $val;
                }
            }

            unset($this->segments[0]);
        }
    }

    protected function _parse_query_string()
    {
        // 这个底层不知道是哪里的坑
        $uri = isset($_SERVER['QUERY_STRING']) ? $_SERVER['QUERY_STRING'] : @getenv('QUERY_STRING');

        if (trim($uri, '/') === '') {
            // 没有 query
            return '';
        } elseif (strncmp($uri, '/', 1) === 0) {
            // explode('?', '?for=bar&baz=sadf', 2)
            // => [
            //      "",
            //      "for=bar&baz=sadf",
            //    ]
            // 所以这个 2 的意思是只将第一个 ？ explode 出来
            $uri = explode('?', $uri, 2);
            $_SERVER['QUERY_STRING'] = isset($uri[1]) ? $uri[1] : '';
            $uri = $uri[0];
        }

        parse_str($_SERVER['QUERY_STRING'], $_GET);

        return $this->_remove_relative_directory($uri);
    }

    // php index.php controller method -> /controller/method
    protected function _parse_argv()
    {
        $args = array_slice($_SERVER['argv'], 1);
        return $args ? implode('/', $args) : '';
    }

    // 非常直白
    public function filter_uri(&$str)
    {
        if (! empty($str) && ! empty($this->_permitted_uri_chars) && ! preg_match('/^['.$this->_permitted_uri_chars.']+$/i'.(UTF8_ENABLED ? 'u' : ''), $str)) {
            show_error('The URI you submitted has disallowed characters.', 400);
        }
    }

    // $n segment 的 number (从1开始)
    public function segment($n, $no_result = null)
    {
        return isset($this->segments[$n]) ? $this->segments[$n] : $no_result;
    }

    /**
     * Fetch URI "routed" Segment
     *
     * Returns the re-routed URI segment (assuming routing rules are used)
     * based on the index provided. If there is no routing, will return
     * the same result as CI_URI::segment().
     *
     * 这个 方法 signature 和 self::segment 是一样的
     * 但是因为涉及到 Router 这个核心类所以现在看起来有点不明白
     *
     * 但是我猜应该是 Router 这个类会根据用户的输入进行一定的 mapping 将不同的 segment
     * map 给 routes 定义的 {东西}
     *
     */
    public function rsegment($n, $no_result = null)
    {
        return isset($this->rsegments[$n]) ? $this->rsegments[$n] : $no_result;
    }

    /**
     * URI to assoc
     *
     * 终于有了一个可读的 comment
     *
     * Generates an associative array of URI data starting at the supplied
     * segment index. For example, if this is your URI:
     *
     *  example.com/user/search/name/joe/location/UK/gender/male
     *
     * You can use this method to generate an array with this prototype:
     *
     *  array (
     *      name => joe
     *      location => UK
     *      gender => male
     *   )
     *
     * @param   int $n      Index (default: 3) 
     * 这里是因为 CI 觉得第一个是 controller 第二个应该是 method
     * @param   array   $default    Default values
     * @return  array
     */
    public function uri_to_assoc($n = 3, $default = array())
    {
        return $this->_uri_to_assoc($n, $default, 'segment');
    }

    // 同上
    public function ruri_to_assoc($n = 3, $default = array())
    {
        return $this->_uri_to_assoc($n, $default, 'rsegment');
    }

    // 总算是一个 refactor 的案例
    // $which 可以是 'segment' 或者 'rsegment'
    protected function _uri_to_assoc($n = 3, $default = array(), $which = 'segment')
    {
        if (! is_numeric($n)) {
            return $default;
        }

        if (isset($this->keyval[$which], $this->keyval[$which][$n])) {
            return $this->keyval[$which][$n];
        }

        // 这两个乱入 似乎是 debug 用得
        $total_segments = "total_{$which}s";
        $segment_array = "{$which}_array";

        if ($this->$total_segments() < $n) {
            return (count($default) === 0)
                ? array()
                : array_fill_keys($default, null);
        }

        $segments = array_slice($this->$segment_array(), ($n - 1));
        $i = 0;
        $lastval = '';
        $retval = array();
        foreach ($segments as $seg) {
            if ($i % 2) {
                $retval[$lastval] = $seg;
            } else {
                $retval[$seg] = null;
                $lastval = $seg;
            }

            $i++;
        }

        if (count($default) > 0) {
            foreach ($default as $val) {
                if (! array_key_exists($val, $retval)) {
                    $retval[$val] = null;
                }
            }
        }

        // Cache the array for reuse
        isset($this->keyval[$which]) or $this->keyval[$which] = array();
        $this->keyval[$which][$n] = $retval;
        return $retval;
    }

    // 上一个方法的逆向
    // 将一个 key=> value 变成 /key/value
    public function assoc_to_uri($array)
    {
        $temp = array();
        foreach ((array) $array as $key => $val) {
            $temp[] = $key;
            $temp[] = $val;
        }

        return implode('/', $temp);
    }

    // helper
    public function slash_segment($n, $where = 'trailing')
    {
        return $this->_slash_segment($n, $where, 'segment');
    }

    // helper
    public function slash_rsegment($n, $where = 'trailing')
    {
        return $this->_slash_segment($n, $where, 'rsegment');
    }

    // helper
    protected function _slash_segment($n, $where = 'trailing', $which = 'segment')
    {
        $leading = $trailing = '/';

        if ($where === 'trailing') {
            $leading    = '';
        } elseif ($where === 'leading') {
            $trailing    = '';
        }

        return $leading.$this->$which($n).$trailing;
    }

    // getter
    public function segment_array()
    {
        return $this->segments;
    }

    // getter
    public function rsegment_array()
    {
        return $this->rsegments;
    }

    // getter
    public function total_segments()
    {
        return count($this->segments);
    }

    // getter
    public function total_rsegments()
    {
        return count($this->rsegments);
    }

    // getter
    public function uri_string()
    {
        return $this->uri_string;
    }

    // 居然还有方法级别的依赖。。。。
    // 后面在研究 route
    public function ruri_string()
    {
        return ltrim(load_class('Router', 'core')->directory, '/').implode('/', $this->rsegments);
    }
}
?>
```

呼 终于读完了

不过总之 CI 的 URI 的解析比较直白，还停留在静态页面的水平

这个类内置了一些 helper method 

不过还是要喷一喷 CI 整体老旧的架构和诡异的依赖管理。

方法级别的 load_class 看上去像是后来补上来得 并不像是深思熟虑之后的
