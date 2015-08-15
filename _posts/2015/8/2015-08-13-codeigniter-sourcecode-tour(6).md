---
layout:     post
title:      php CI 源码探索(6)
date:       2015-08-11 1:33
summary:    php CI 代码解读(5) Route
categories: php CodeIgniter
---

1. ~~如果 log_message 先被调用了 那么加载 `Exceptions` 类 (动态加载，只有在程序需要的时候加载) 副作用就是一起加载了 Log 类~~
2. ~~加载 `Benchmark` ($BM)~~
3. ~~加载 `Hooks` ($EXT)~~
4. ~~加载 `Config` ($CFG)~~
5. ~~加载 `Utf8` ($UNI)~~
6. ~~加载 `URI` ($URI)~~
7. ~~加载 `Router` ($RTR)~~
8. **加载 `Output` ($OUT)**
9. 加载 `Security` ($SEC)
10. 加载 `Language` ($LANG)

既然开坑了那就不放弃 继续吐血看

Output 应该是负责渲染所有的 views 的所以这个类实现的东西比较多

所以这个类相对比较长

``` php
<?php
class CI_Output
{
    public $final_output;

    public $cache_expiration = 0;

    public $headers = array();

    public $mimes =    array();

    protected $mime_type = 'text/html';

    public $enable_profiler = false;

    protected $_zlib_oc = false;

    protected $_compress_output = false;

    protected $_profiler_sections =    array();

    public $parse_exec_vars = true;

    // constructor 会看下 要不要用 zlib 
    public function __construct()
    {
        $this->_zlib_oc = (bool) ini_get('zlib.output_compression');
        // 要想压缩必须开这些东西才行
        $this->_compress_output = (
            $this->_zlib_oc === false
            && config_item('compress_output') === true
            && extension_loaded('zlib')
        );

        // Get mime types for later
        $this->mimes =& get_mimes();

        log_message('info', 'Output Class Initialized');
    }

    public function get_output()
    {
        return $this->final_output;
    }

    public function set_output($output)
    {
        $this->final_output = $output;
        return $this;
    }

    public function append_output($output)
    {
        $this->final_output .= $output;
        // 为了 chaining
        return $this;
    }

    public function set_header($header, $replace = true)
    {
        // If zlib.output_compression is enabled it will compress the output,
        // but it will not modify the content-length header to compensate for
        // the reduction, causing the browser to hang waiting for more data.
        // We'll just skip content-length in those cases.
        // 看 comment 感觉应该是 zlib 会对数据进行压缩但是好像并没有改变 content-length
        // 的大小。导致最后浏览器刷不出数据，因为期待得到的大小和实际大小不一
        // 所以 CI 给出的 solution居然是放弃 content-length 这个 header
        // 注意这个类的作者似乎很中意用 strncasecmp 这个函数 这应该就是一个复杂版的 
        // `==` 当只有两个字符串相等的时候这个函数才返回 0
        if ($this->_zlib_oc && strncasecmp($header, 'content-length', 14) === 0) {
            return $this;
        }

        $this->headers[] = array($header, $replace);
        return $this;
    }

    public function set_content_type($mime_type, $charset = null)
    {
        if (strpos($mime_type, '/') === false) {
            $extension = ltrim($mime_type, '.');

            // Is this extension supported?
            if (isset($this->mimes[$extension])) {
                $mime_type =& $this->mimes[$extension];

                if (is_array($mime_type)) {
                    // 因为这个 mime_type 是作为一个引用传进来的 所以在
                    // 整个 application 里面做过任何 next pre等等操作
                    // 都会改变这个数组的 current 指针
                    // 现在默认就是指向了数组的第一个元素
                    $mime_type = current($mime_type);
                }
            }
        }

        $this->mime_type = $mime_type;

        if (empty($charset)) {
            $charset = config_item('charset');
        }

        $header = 'Content-Type: '.$mime_type
            .(empty($charset) ? '' : '; charset='.$charset);

        $this->headers[] = array($header, true);
        return $this;
    }

    public function get_content_type()
    {
        for ($i = 0, $c = count($this->headers); $i < $c; $i++) {
            if (sscanf($this->headers[$i][0], 'Content-Type: %[^;]', $content_type) === 1) {
                return $content_type;
            }
        }

        return 'text/html';
    }

    public function get_header($header)
    {
        // Combine headers already sent with our batched headers
        $headers = array_merge(
            // We only need [x][0] from our multi-dimensional array
            // 意思就是值关心每个二维数组的第一个元素
            // 同时还给 $this->headers 降了一个维度
            // 这个写法挺凶的 受教
            // 
            // 原理等同
            // array_map(function(item) {
            //     return array_shift(item);
            // }, $this->headers);
            array_map('array_shift', $this->headers),
            headers_list()
        );

        if (empty($headers) or empty($header)) {
            return null;
        }

        // 传统 for loop 找出来 header
        for ($i = 0, $c = count($headers); $i < $c; $i++) {
            if (strncasecmp($header, $headers[$i], $l = strlen($header)) === 0) {
                return trim(substr($headers[$i], $l+1));
            }
        }

        return null;
    }

    public function set_status_header($code = 200, $text = '')
    {
        set_status_header($code, $text);
        return $this;
    }

    public function enable_profiler($val = true)
    {
        $this->enable_profiler = is_bool($val) ? $val : true;
        return $this;
    }

    public function set_profiler_sections($sections)
    {
        // 感觉这是后来补坑补回来的
        if (isset($sections['query_toggle_count'])) {
            $this->_profiler_sections['query_toggle_count'] = (int) $sections['query_toggle_count'];
            unset($sections['query_toggle_count']);
        }

        foreach ($sections as $section => $enable) {
            $this->_profiler_sections[$section] = ($enable !== false);
        }

        return $this;
    }

    public function cache($time)
    {
        $this->cache_expiration = is_numeric($time) ? $time : 0;
        return $this;
    }

    /**
     * Display Output
     * 这个方法就比较关键了 推测 controller 会在幕后将 view 通过
     * set_finaloutput 这个方法传给 Output 类然后当一个请求结束的时候
     * 调用这个方法将所有的 header 和所有的 output 全都甩给 browser
     *
     * 这里原始的注释比较全而且出了奇的有用
     *
     * Processes and sends finalized output data to the browser along
     * with any server headers and profile data. It also stops benchmark
     * timers so the page rendering speed and memory usage can be shown.
     *
     * Note: All "view" data is automatically put into $this->final_output
     *   by controller class.
     */
    public function _display($output = '')
    {
        // Note:  We use load_class() because we can't use $CI =& get_instance()
        // since this function is sometimes called by the caching mechanism,
        // which happens before the CI super object is available.
        $BM =& load_class('Benchmark', 'core');
        $CFG =& load_class('Config', 'core');

        // Grab the super object if we can.
        if (class_exists('CI_Controller', false)) {
            $CI =& get_instance();
        }

        // --------------------------------------------------------------------

        // Set the output data
        if ($output === '') {
            $output =& $this->final_output;
        }

        // --------------------------------------------------------------------

        // Do we need to write a cache file? Only if the controller does not have its
        // own _output() method and we are not dealing with a cache file, which we
        // can determine by the existence of the $CI object above
        if ($this->cache_expiration > 0 && isset($CI) && ! method_exists($CI, '_output')) {
            $this->_write_cache($output);
        }

        // --------------------------------------------------------------------

        // Parse out the elapsed time and memory usage,
        // then swap the pseudo-variables with the data
        // 终于见光了 之前 $BM 中的 {elapsed_time} 原来就是这个的简写
        $elapsed = $BM->elapsed_time('total_execution_time_start', 'total_execution_time_end');

        if ($this->parse_exec_vars === true) {
            $memory    = round(memory_get_usage() / 1024 / 1024, 2).'MB';
            $output = str_replace(array('{elapsed_time}', '{memory_usage}'), array($elapsed, $memory), $output);
        }

        // --------------------------------------------------------------------

        // Is compression requested?
        if (isset($CI) // This means that we're not serving a cache file, if we were, it would already be compressed
            && $this->_compress_output === true
            && isset($_SERVER['HTTP_ACCEPT_ENCODING']) && strpos($_SERVER['HTTP_ACCEPT_ENCODING'], 'gzip') !== false) {
            // 开一个 buffer 来装所有被压缩的
            ob_start('ob_gzhandler');
        }

        // --------------------------------------------------------------------

        // Are there any server headers to send?
        if (count($this->headers) > 0) {
            foreach ($this->headers as $header) {
                @header($header[0], $header[1]);
            }
        }

        // --------------------------------------------------------------------

        // Does the $CI object exist?
        // If not we know we are dealing with a cache file so we'll
        // simply echo out the data and exit.
        if (! isset($CI)) {
            if ($this->_compress_output === true) {
                if (isset($_SERVER['HTTP_ACCEPT_ENCODING']) && strpos($_SERVER['HTTP_ACCEPT_ENCODING'], 'gzip') !== false) {
                    header('Content-Encoding: gzip');
                    header('Content-Length: '.strlen($output));
                } else {
                    // User agent doesn't support gzip compression,
                    // so we'll have to decompress our cache
                    $output = gzinflate(substr($output, 10, -8));
                }
            }

            echo $output;
            log_message('info', 'Final output sent to browser');
            log_message('debug', 'Total execution time: '.$elapsed);
            return;
        }

        // --------------------------------------------------------------------

        // Do we need to generate profile data?
        // If so, load the Profile class and run it.
        if ($this->enable_profiler === true) {
            $CI->load->library('profiler');
            if (! empty($this->_profiler_sections)) {
                $CI->profiler->set_sections($this->_profiler_sections);
            }

            // If the output data contains closing </body> and </html> tags
            // we will remove them and add them back after we insert the profile data
            $output = preg_replace('|</body>.*?</html>|is', '', $output, -1, $count).$CI->profiler->run();
            if ($count > 0) {
                $output .= '</body></html>';
            }
        }

        // Does the controller contain a function named _output()?
        // If so send the output there.  Otherwise, echo it.
        if (method_exists($CI, '_output')) {
            $CI->_output($output);
        } else {
            echo $output; // Send it to the browser!
        }

        log_message('info', 'Final output sent to browser');
        log_message('debug', 'Total execution time: '.$elapsed);
    }

    public function _write_cache($output)
    {
        $CI =& get_instance();
        $path = $CI->config->item('cache_path');
        $cache_path = ($path === '') ? APPPATH.'cache/' : $path;

        if (! is_dir($cache_path) or ! is_really_writable($cache_path)) {
            log_message('error', 'Unable to write cache file: '.$cache_path);
            return;
        }

        $uri = $CI->config->item('base_url')
            .$CI->config->item('index_page')
            .$CI->uri->uri_string();

        if ($CI->config->item('cache_query_string') && ! empty($_SERVER['QUERY_STRING'])) {
            $uri .= '?'.$_SERVER['QUERY_STRING'];
        }
        // 加个 hash 防撞名
        $cache_path .= md5($uri);

        if (! $fp = @fopen($cache_path, 'w+b')) {
            log_message('error', 'Unable to write cache file: '.$cache_path);
            return;
        }

        if (flock($fp, LOCK_EX)) {
            // If output compression is enabled, compress the cache
            // itself, so that we don't have to do that each time
            // we're serving it
            if ($this->_compress_output === true) {
                $output = gzencode($output);

                if ($this->get_header('content-type') === null) {
                    $this->set_content_type($this->mime_type);
                }
            }

            $expire = time() + ($this->cache_expiration * 60);

            // Put together our serialized info.
            $cache_info = serialize(array(
                'expire'    => $expire,
                'headers'    => $this->headers
            ));

            $output = $cache_info.'ENDCI--->'.$output;

            for ($written = 0, $length = strlen($output); $written < $length; $written += $result) {
                if (($result = fwrite($fp, substr($output, $written))) === false) {
                    break;
                }
            }

            flock($fp, LOCK_UN);
        } else {
            log_message('error', 'Unable to secure a file lock for file at: '.$cache_path);
            return;
        }

        fclose($fp);

        if (is_int($result)) {
            chmod($cache_path, 0640);
            log_message('debug', 'Cache file written: '.$cache_path);

            // Send HTTP cache-control headers to browser to match file cache settings.
            $this->set_cache_header($_SERVER['REQUEST_TIME'], $expire);
        } else {
            @unlink($cache_path);
            log_message('error', 'Unable to write the complete cache content at: '.$cache_path);
        }
    }

    public function _display_cache(&$CFG, &$URI)
    {
        // 我擦这一大堆重复逻辑

        $cache_path = ($CFG->item('cache_path') === '') ? APPPATH.'cache/' : $CFG->item('cache_path');

        // Build the file path. The file name is an MD5 hash of the full URI
        $uri = $CFG->item('base_url').$CFG->item('index_page').$URI->uri_string;

        if ($CFG->item('cache_query_string') && ! empty($_SERVER['QUERY_STRING'])) {
            $uri .= '?'.$_SERVER['QUERY_STRING'];
        }

        $filepath = $cache_path.md5($uri);

        if (! file_exists($filepath) or ! $fp = @fopen($filepath, 'rb')) {
            return false;
        }

        flock($fp, LOCK_SH);

        $cache = (filesize($filepath) > 0) ? fread($fp, filesize($filepath)) : '';

        flock($fp, LOCK_UN);
        fclose($fp);

        // Look for embedded serialized file info.
        if (! preg_match('/^(.*)ENDCI--->/', $cache, $match)) {
            return false;
        }

        $cache_info = unserialize($match[1]);
        $expire = $cache_info['expire'];

        $last_modified = filemtime($cache_path);

        // Has the file expired?
        if ($_SERVER['REQUEST_TIME'] >= $expire && is_really_writable($cache_path)) {
            // If so we'll delete it.
            @unlink($filepath);
            log_message('debug', 'Cache file has expired. File deleted.');
            return false;
        } else {
            // Or else send the HTTP cache control headers.
            $this->set_cache_header($last_modified, $expire);
        }

        // Add headers from cache file.
        foreach ($cache_info['headers'] as $header) {
            $this->set_header($header[0], $header[1]);
        }

        // Display the cache
        $this->_display(substr($cache, strlen($match[0])));
        log_message('debug', 'Cache file is current. Sending it to browser.');
        return true;
    }

    public function delete_cache($uri = '')
    {
        $CI =& get_instance();
        $cache_path = $CI->config->item('cache_path');
        if ($cache_path === '') {
            $cache_path = APPPATH.'cache/';
        }

        if (! is_dir($cache_path)) {
            log_message('error', 'Unable to find cache path: '.$cache_path);
            return false;
        }

        if (empty($uri)) {
            $uri = $CI->uri->uri_string();

            if ($CI->config->item('cache_query_string') && ! empty($_SERVER['QUERY_STRING'])) {
                $uri .= '?'.$_SERVER['QUERY_STRING'];
            }
        }

        $cache_path .= md5($CI->config->item('base_url').$CI->config->item('index_page').$uri);

        if (! @unlink($cache_path)) {
            log_message('error', 'Unable to delete cache file for '.$uri);
            return false;
        }

        return true;
    }

    public function set_cache_header($last_modified, $expiration)
    {
        $max_age = $expiration - $_SERVER['REQUEST_TIME'];

        if (isset($_SERVER['HTTP_IF_MODIFIED_SINCE']) && $last_modified <= strtotime($_SERVER['HTTP_IF_MODIFIED_SINCE'])) {
            $this->set_status_header(304);
            exit;
        } else {
            header('Pragma: public');
            header('Cache-Control: max-age='.$max_age.', public');
            header('Expires: '.gmdate('D, d M Y H:i:s', $expiration).' GMT');
            header('Last-modified: '.gmdate('D, d M Y H:i:s', $last_modified).' GMT');
        }
    }
}
?>
```

后面都不想看了 管理太混乱，一方面 CI 实例会调用这个类的方法

另一方面这个类居然在内部还要取得 CI 实例 反正我已经彻底看乱了

这么看似乎是 output 主动发起 output 而不是 CI 实例控制 output 向浏览器吐出 output 
