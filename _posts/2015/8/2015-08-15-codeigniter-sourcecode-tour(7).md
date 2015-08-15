---
layout:     post
title:      php CI 源码探索(5)
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
8. ~~加载 `Output` ($OUT)~~
9. **加载 `Security` ($SEC)**
10. **加载 `Language` ($LANG)**

重要的一环 security 

这个核心类包括了 1. 过滤 xss 2. 提供csrf保护

暂时对安全没什么兴趣 不过可以看到 CI 默认是对 COOKIE 进行验证 而且是对每一个非 POST 请求都会验

接下来就是对于我比较有用的 i18n 支持了 代码不长 不过看一遍可以理解下

``` php
<?php
class CI_Lang
{
    // List of translations
    public $language =    array();

    // List of loaded language files
    public $is_loaded =    array();

    public function __construct()
    {
        log_message('info', 'Language Class Initialized');
    }

    /**
     * Load a language file
     *
     * @param   mixed   $langfile   Language file name
     * @param   string  $idiom      Language name (english, etc.)
     * @param   bool    $return     Whether to return the loaded array of translations
     * @param   bool    $add_suffix Whether to add suffix to $langfile
     * @param   string  $alt_path   Alternative path to look for the language file
     *
     * @return  void|string[]   Array containing translations, if $return is set to TRUE
     */
    public function load($langfile, $idiom = '', $return = false, $add_suffix = true, $alt_path = '')
    {
        if (is_array($langfile)) {
            foreach ($langfile as $value) {
                // 我擦 recursive 
                $this->load($value, $idiom, $return, $add_suffix, $alt_path);
            }

            return;
        }

        $langfile = str_replace('.php', '', $langfile);

        if ($add_suffix === true) {
            $langfile = preg_replace('/_lang$/', '', $langfile).'_lang';
        }

        $langfile .= '.php';

        if (empty($idiom) or ! preg_match('/^[a-z_-]+$/i', $idiom)) {
            $config =& get_config();
            $idiom = empty($config['language']) ? 'english' : $config['language'];
        }

        if ($return === false && isset($this->is_loaded[$langfile]) && $this->is_loaded[$langfile] === $idiom) {
            return;
        }

        // Load the base file, so any others found can override it
        // 默认在 system/language/english
        $basepath = BASEPATH.'language/'.$idiom.'/'.$langfile;
        if (($found = file_exists($basepath)) === true) {
            include($basepath);
        }

        // Do we have an alternative path to look in?
        if ($alt_path !== '') {
            $alt_path .= 'language/'.$idiom.'/'.$langfile;
            if (file_exists($alt_path)) {
                include($alt_path);
                $found = true;
            }
        } else {
            // instance 里面这个 get_package_paths 有待研究
            foreach (get_instance()->load->get_package_paths(true) as $package_path) {
                $package_path .= 'language/'.$idiom.'/'.$langfile;
                if ($basepath !== $package_path && file_exists($package_path)) {
                    include($package_path);
                    $found = true;
                    break;
                }
            }
        }

        if ($found !== true) {
            show_error('Unable to load the requested language file: language/'.$idiom.'/'.$langfile);
        }

        if (! isset($lang) or ! is_array($lang)) {
            log_message('error', 'Language file contains no data: language/'.$idiom.'/'.$langfile);

            if ($return === true) {
                return array();
            }
            return;
        }

        if ($return === true) {
            return $lang;
        }

        $this->is_loaded[$langfile] = $idiom;
        $this->language = array_merge($this->language, $lang);

        log_message('info', 'Language file loaded: language/'.$idiom.'/'.$langfile);
        return true;
    }

    // 将 $this->language 中的一个 key 拿出来
    public function line($line, $log_errors = true)
    {
        $value = isset($this->language[$line]) ? $this->language[$line] : false;

        // Because killer robots like unicorns!
        if ($value === false && $log_errors === true) {
            log_message('error', 'Could not find the language line "'.$line.'"');
        }

        return $value;
    }
}
?>
```

好了终于要到最重要的一环了 

CI_Controller

看了半天代码发现最后的一切都是  CI_Controller 这个始作俑者。

感觉所有的之前的一切都是在个 CI_Controller 这个核心中的战斗机做一些准备。

router 负责找到合适 controller instance 并 require 进来 然后 router 还负责找到适当地 controller method 和 variables

最后汇总成一句话 `call_user_func_array(array(&$CI, $method), $params);`

而所有 Controller 的超类都在这里 超短 但是因为之前那么一大坨准备功夫都做了所以

``` php
<?php
/**
 * Application Controller Class
 *
 * This class object is the super class that every library in
 * CodeIgniter will be assigned to.
 *
 * 看到这句话了么 `every` 就是说所有加载过的核心都会被作为 instance 加载进来
 *
 */
class CI_Controller
{
    /**
     * Reference to the CI singleton
     *
     * @var object
     */
    private static $instance;

    /**
     * Class constructor
     *
     * @return  void
     */
    public function __construct()
    {
        self::$instance =& $this;

        // Assign all the class objects that were instantiated by the
        // bootstrap file (CodeIgniter.php) to local class variables
        // so that CI can run as one big super object.
        // 
        // Benchmark Config etc...
        foreach (is_loaded() as $var => $class) {
            $this->$var =& load_class($class);
        }

        // 接下来就到了更加核心的一个组件了 Loader
        // 这个file 很长
        $this->load =& load_class('Loader', 'core');
        $this->load->initialize();
        log_message('info', 'Controller Class Initialized');
    }

    public static function &get_instance()
    {
        return self::$instance;
    }
}
?>
```
Loader 应该是整个 CI 程序的心脏了

平时看到的什么 

``` php
<?php $this->load('database'); ?>
```
什么的都是在这个大类里面实现的

``` php
<?php
class CI_Loader
{
    // All these are set automatically. Don't mess with them.
    /**
     * Nesting level of the output buffering mechanism
     *
     * @var int
     */
    protected $_ci_ob_level;

    /**
     * List of paths to load views from
     *
     * @var array
     */
    protected $_ci_view_paths =    array(VIEWPATH    => true);

    /**
     * List of paths to load libraries from
     *
     * @var array
     */
    protected $_ci_library_paths =    array(APPPATH, BASEPATH);

    /**
     * List of paths to load models from
     *
     * @var array
     */
    protected $_ci_model_paths =    array(APPPATH);

    /**
     * List of paths to load helpers from
     *
     * @var array
     */
    protected $_ci_helper_paths =    array(APPPATH, BASEPATH);

    /**
     * List of cached variables
     *
     * @var array
     */
    protected $_ci_cached_vars =    array();

    /**
     * List of loaded classes
     *
     * @var array
     */
    protected $_ci_classes =    array();

    /**
     * List of loaded models
     *
     * @var array
     */
    protected $_ci_models =    array();

    /**
     * List of loaded helpers
     *
     * @var array
     */
    protected $_ci_helpers =    array();

    /**
     * List of class name mappings
     *
     * @var array
     */
    protected $_ci_varmap =    array(
        'unit_test' => 'unit',
        'user_agent' => 'agent'
    );

    public function __construct()
    {
        // 之前 output 中可能有压缩所以可能是1或者 2
        $this->_ci_ob_level = ob_get_level();
        $this->_ci_classes =& is_loaded();

        log_message('info', 'Loader Class Initialized');
    }

    // 这才是这个类的骨架
    public function initialize()
    {
        $this->_ci_autoloader();
    }

    public function is_loaded($class)
    {
        return array_search(ucfirst($class), $this->_ci_classes, true);
    }

    // 常用的 $this->load->library('foo')
    public function library($library, $params = null, $object_name = null)
    {
        if (empty($library)) {
            return $this;
        } elseif (is_array($library)) {
            foreach ($library as $key => $value) {
                if (is_int($key)) {
                    $this->library($value, $params);
                } else {
                    $this->library($key, $params, $value);
                }
            }

            return $this;
        }

        if ($params !== null && ! is_array($params)) {
            $params = null;
        }
        // 委托给私有方法 
        $this->_ci_load_library($library, $params, $object_name);
        return $this;
    }

    // name 是一个 alias
    public function model($model, $name = '', $db_conn = false)
    {
        if (empty($model)) {
            return $this;
        } elseif (is_array($model)) {
            foreach ($model as $key => $value) {
                is_int($key) ? $this->model($value, '', $db_conn) : $this->model($key, $value, $db_conn);
            }

            return $this;
        }

        $path = '';

        // Is the model in a sub-folder? If so, parse out the filename and path.
        if (($last_slash = strrpos($model, '/')) !== false) {
            // The path is in front of the last slash
            $path = substr($model, 0, ++$last_slash);

            // And the model name behind it
            $model = substr($model, $last_slash);
        }

        if (empty($name)) {
            $name = $model;
        }
        // 缓存过了
        if (in_array($name, $this->_ci_models, true)) {
            return $this;
        }

        $CI =& get_instance();
        if (isset($CI->$name)) {
            show_error('The model name you are loading is the name of a resource that is already being used: '.$name);
        }

        if ($db_conn !== false && ! class_exists('CI_DB', false)) {
            if ($db_conn === true) {
                $db_conn = '';
            }

            $this->database($db_conn, false, true);
        }

        if (! class_exists('CI_Model', false)) {
            load_class('Model', 'core');
        }

        $model = ucfirst(strtolower($model));

        foreach ($this->_ci_model_paths as $mod_path) {
            if (! file_exists($mod_path.'models/'.$path.$model.'.php')) {
                continue;
            }

            require_once($mod_path.'models/'.$path.$model.'.php');

            // 将新的 Model 压入缓存
            $this->_ci_models[] = $name;
            // 所以我们可以用 $this->{$model_name}
            $CI->$name = new $model();
            return $this;
        }

        // couldn't find the model
        show_error('Unable to locate the model you have specified: '.$model);
    }

    /**
     * Database Loader
     *
     * @param   mixed   $params     Database configuration options
     * @param   bool    $return     Whether to return the database object
     * @param   bool    $query_builder  Whether to enable Query Builder
     *                  (overrides the configuration setting)
     *
     * @return  object|bool Database object if $return is set to TRUE,
     *                  FALSE on failure, CI_Loader instance in any other case
     */
    public function database($params = '', $return = false, $query_builder = null)
    {
        // Grab the super object
        $CI =& get_instance();

        // Do we even need to load the database class?
        if ($return === false && $query_builder === null && isset($CI->db) && is_object($CI->db) && ! empty($CI->db->conn_id)) {
            return false;
        }
        // function &DB
        require_once(BASEPATH.'database/DB.php');

        if ($return === true) {
            return DB($params, $query_builder);
        }

        // Initialize the db variable. Needed to prevent
        // reference errors with some configurations
        $CI->db = '';

        // Load the DB class
        // 原来是另一个用函数实现的单例。。。。
        $CI->db =& DB($params, $query_builder);
        return $this;
    }

    // 数据库暂时不在我操心范围内
    public function dbutil($db = null, $return = false)
    {
        $CI =& get_instance();

        if (! is_object($db) or ! ($db instanceof CI_DB)) {
            class_exists('CI_DB', false) or $this->database();
            $db =& $CI->db;
        }

        require_once(BASEPATH.'database/DB_utility.php');
        require_once(BASEPATH.'database/drivers/'.$db->dbdriver.'/'.$db->dbdriver.'_utility.php');
        $class = 'CI_DB_'.$db->dbdriver.'_utility';

        if ($return === true) {
            return new $class($db);
        }

        $CI->dbutil = new $class($db);
        return $this;
    }

    public function dbforge($db = null, $return = false)
    {
        $CI =& get_instance();
        if (! is_object($db) or ! ($db instanceof CI_DB)) {
            class_exists('CI_DB', false) or $this->database();
            $db =& $CI->db;
        }

        require_once(BASEPATH.'database/DB_forge.php');
        require_once(BASEPATH.'database/drivers/'.$db->dbdriver.'/'.$db->dbdriver.'_forge.php');

        if (! empty($db->subdriver)) {
            $driver_path = BASEPATH.'database/drivers/'.$db->dbdriver.'/subdrivers/'.$db->dbdriver.'_'.$db->subdriver.'_forge.php';
            if (file_exists($driver_path)) {
                require_once($driver_path);
                $class = 'CI_DB_'.$db->dbdriver.'_'.$db->subdriver.'_forge';
            }
        } else {
            $class = 'CI_DB_'.$db->dbdriver.'_forge';
        }

        if ($return === true) {
            return new $class($db);
        }

        $CI->dbforge = new $class($db);
        return $this;
    }

    /**
     * View Loader
     * 这个比较重要的 这个是 CI 的 return view('foo.bar')
     *
     * Loads "view" files.
     *
     * @param   string  $view   View name
     * @param   array   $vars   An associative array of data
     *              to be extracted for use in the view
     * @param   bool    $return Whether to return the view output
     *              or leave it to the Output class
     * @return  object|string
     */
    public function view($view, $vars = array(), $return = false)
    {
        return $this->_ci_load(array('_ci_view' => $view, '_ci_vars' => $this->_ci_object_to_array($vars), '_ci_return' => $return));
    }

    /**
     * Generic File Loader
     *
     * @param   string  $path   File path
     * @param   bool    $return Whether to return the file output
     * @return  object|string
     */
    public function file($path, $return = false)
    {
        return $this->_ci_load(array('_ci_path' => $path, '_ci_return' => $return));
    }
    // setter
    // controller 中所有的 vars 会在 view 中得到 access
    public function vars($vars, $val = '')
    {
        if (is_string($vars)) {
            $vars = array($vars => $val);
        }

        $vars = $this->_ci_object_to_array($vars);

        if (is_array($vars) && count($vars) > 0) {
            foreach ($vars as $key => $val) {
                $this->_ci_cached_vars[$key] = $val;
            }
        }

        return $this;
    }

    // 清缓存
    public function clear_vars()
    {
        $this->_ci_cached_vars = array();
        return $this;
    }

    // getter
    public function get_var($key)
    {
        return isset($this->_ci_cached_vars[$key]) ? $this->_ci_cached_vars[$key] : null;
    }

    // mass getter
    public function get_vars()
    {
        return $this->_ci_cached_vars;
    }

    // load helper
    public function helper($helpers = array())
    {
        foreach ($this->_ci_prep_filename($helpers, '_helper') as $helper) {
            if (isset($this->_ci_helpers[$helper])) {
                continue;
            }

            // Is this a helper extension request?
            $ext_helper = config_item('subclass_prefix').$helper;
            $ext_loaded = false;
            foreach ($this->_ci_helper_paths as $path) {
                if (file_exists($path.'helpers/'.$ext_helper.'.php')) {
                    include_once($path.'helpers/'.$ext_helper.'.php');
                    $ext_loaded = true;
                }
            }

            // If we have loaded extensions - check if the base one is here
            if ($ext_loaded === true) {
                $base_helper = BASEPATH.'helpers/'.$helper.'.php';
                if (! file_exists($base_helper)) {
                    show_error('Unable to load the requested file: helpers/'.$helper.'.php');
                }

                include_once($base_helper);
                $this->_ci_helpers[$helper] = true;
                log_message('info', 'Helper loaded: '.$helper);
                continue;
            }

            // No extensions found ... try loading regular helpers and/or overrides
            foreach ($this->_ci_helper_paths as $path) {
                if (file_exists($path.'helpers/'.$helper.'.php')) {
                    include_once($path.'helpers/'.$helper.'.php');

                    $this->_ci_helpers[$helper] = true;
                    log_message('info', 'Helper loaded: '.$helper);
                    break;
                }
            }

            // unable to load the helper
            if (! isset($this->_ci_helpers[$helper])) {
                show_error('Unable to load the requested file: helpers/'.$helper.'.php');
            }
        }

        return $this;
    }

    // array to string
    public function helpers($helpers = array())
    {
        return $this->helper($helpers);
    }

    // i18n 支援
    // 一般直接写在超类里面
    // e.g. /en/index -> 通过 pre_controller 将 en 提取成
    // params 传入 ctor
    // 然后调用 $this->lang->load('error_message', 'english');
    public function language($files, $lang = '')
    {
        get_instance()->lang->load($files, $lang);
        return $this;
    }

    public function config($file, $use_sections = false, $fail_gracefully = false)
    {
        return get_instance()->config->load($file, $use_sections, $fail_gracefully);
    }
    // db 相关
    public function driver($library, $params = null, $object_name = null)
    {
        if (is_array($library)) {
            foreach ($library as $driver) {
                $this->driver($driver);
            }

            return $this;
        } elseif (empty($library)) {
            return false;
        }

        if (! class_exists('CI_Driver_Library', false)) {
            // We aren't instantiating an object here, just making the base class available
            require BASEPATH.'libraries/Driver.php';
        }

        // We can save the loader some time since Drivers will *always* be in a subfolder,
        // and typically identically named to the library
        if (! strpos($library, '/')) {
            $library = ucfirst($library).'/'.$library;
        }

        return $this->library($library, $params, $object_name);
    }

    // 不用 composer 的代价就是自己实现 autoload
    public function add_package_path($path, $view_cascade = true)
    {
        $path = rtrim($path, '/').'/';

        array_unshift($this->_ci_library_paths, $path);
        array_unshift($this->_ci_model_paths, $path);
        array_unshift($this->_ci_helper_paths, $path);

        $this->_ci_view_paths = array($path.'views/' => $view_cascade) + $this->_ci_view_paths;

        // Add config file path
        $config =& $this->_ci_get_component('config');
        $config->_config_paths[] = $path;

        return $this;
    }

    public function get_package_paths($include_base = false)
    {
        return ($include_base === true) ? $this->_ci_library_paths : $this->_ci_model_paths;
    }

    public function remove_package_path($path = '')
    {
        $config =& $this->_ci_get_component('config');

        if ($path === '') {
            array_shift($this->_ci_library_paths);
            array_shift($this->_ci_model_paths);
            array_shift($this->_ci_helper_paths);
            array_shift($this->_ci_view_paths);
            array_pop($config->_config_paths);
        } else {
            $path = rtrim($path, '/').'/';
            foreach (array('_ci_library_paths', '_ci_model_paths', '_ci_helper_paths') as $var) {
                if (($key = array_search($path, $this->{$var})) !== false) {
                    unset($this->{$var}[$key]);
                }
            }

            if (isset($this->_ci_view_paths[$path.'views/'])) {
                unset($this->_ci_view_paths[$path.'views/']);
            }

            if (($key = array_search($path, $config->_config_paths)) !== false) {
                unset($config->_config_paths[$key]);
            }
        }

        // make sure the application default paths are still in the array
        $this->_ci_library_paths = array_unique(array_merge($this->_ci_library_paths, array(APPPATH, BASEPATH)));
        $this->_ci_helper_paths = array_unique(array_merge($this->_ci_helper_paths, array(APPPATH, BASEPATH)));
        $this->_ci_model_paths = array_unique(array_merge($this->_ci_model_paths, array(APPPATH)));
        $this->_ci_view_paths = array_merge($this->_ci_view_paths, array(APPPATH.'views/' => true));
        $config->_config_paths = array_unique(array_merge($config->_config_paths, array(APPPATH)));

        return $this;
    }

    /**
     * Internal CI Data Loader
     *
     * Used to load views and files.
     *
     * Variables are prefixed with _ci_ to avoid symbol collision with
     * variables made available to view files.
     *
     * @used-by CI_Loader::view()
     * @used-by CI_Loader::file()
     * @param   array   $_ci_data   Data to load
     * @return  object
     */
    protected function _ci_load($_ci_data)
    {
        // Set the default data variables
        foreach (array('_ci_view', '_ci_vars', '_ci_path', '_ci_return') as $_ci_val) {
            // 传说中的元编程
            $$_ci_val = isset($_ci_data[$_ci_val]) ? $_ci_data[$_ci_val] : false;
        }

        $file_exists = false;

        // Set the path to the requested file
        if (is_string($_ci_path) && $_ci_path !== '') {
            $_ci_x = explode('/', $_ci_path);
            $_ci_file = end($_ci_x);
        } else {
            $_ci_ext = pathinfo($_ci_view, PATHINFO_EXTENSION);
            $_ci_file = ($_ci_ext === '') ? $_ci_view.'.php' : $_ci_view;

            foreach ($this->_ci_view_paths as $_ci_view_file => $cascade) {
                if (file_exists($_ci_view_file.$_ci_file)) {
                    $_ci_path = $_ci_view_file.$_ci_file;
                    $file_exists = true;
                    break;
                }

                if (! $cascade) {
                    break;
                }
            }
        }

        if (! $file_exists && ! file_exists($_ci_path)) {
            show_error('Unable to load the requested file: '.$_ci_file);
        }

        // This allows anything loaded using $this->load (views, files, etc.)
        // to become accessible from within the Controller and Model functions.
        $_ci_CI =& get_instance();
        foreach (get_object_vars($_ci_CI) as $_ci_key => $_ci_var) {
            if (! isset($this->$_ci_key)) {
                $this->$_ci_key =& $_ci_CI->$_ci_key;
            }
        }

        /*
         * Extract and cache variables
         *
         * You can either set variables using the dedicated $this->load->vars()
         * function or via the second parameter of this function. We'll merge
         * the two types and cache them so that views that are embedded within
         * other views can have access to these variables.
         */
        if (is_array($_ci_vars)) {
            $this->_ci_cached_vars = array_merge($this->_ci_cached_vars, $_ci_vars);
        }
        extract($this->_ci_cached_vars);

        /*
         * Buffer the output
         *
         * We buffer the output for two reasons:
         * 1. Speed. You get a significant speed boost.
         * 2. So that the final rendered template can be post-processed by
         *  the output class. Why do we need post processing? For one thing,
         *  in order to show the elapsed page load time. Unless we can
         *  intercept the content right before it's sent to the browser and
         *  then stop the timer it won't be accurate.
         */
        ob_start();

        // If the PHP installation does not support short tags we'll
        // do a little string replacement, changing the short tags
        // to standard PHP echo statements.
        if (! is_php('5.4') && ! ini_get('short_open_tag') && config_item('rewrite_short_tags') === true && function_usable('eval')) {
            echo eval('?>'.preg_replace('/;*\s*\?>/', '; ?>', str_replace('<?=', '<?php echo ', file_get_contents($_ci_path))));
        } else {
            include($_ci_path); // include() vs include_once() allows for multiple views with the same name
        }

        log_message('info', 'File loaded: '.$_ci_path);

        // Return the file data if requested
        if ($_ci_return === true) {
            $buffer = ob_get_contents();
            @ob_end_clean();
            return $buffer;
        }

        /*
         * Flush the buffer... or buff the flusher?
         *
         * In order to permit views to be nested within
         * other views, we need to flush the content back out whenever
         * we are beyond the first level of output buffering so that
         * it can be seen and included properly by the first included
         * template and any subsequent ones. Oy!
         */
        if (ob_get_level() > $this->_ci_ob_level + 1) {
            ob_end_flush();
        } else {
            $_ci_CI->output->append_output(ob_get_contents());
            @ob_end_clean();
        }

        return $this;
    }

    // --------------------------------------------------------------------

    /**
     * Internal CI Library Loader
     *
     * @used-by CI_Loader::library()
     * @uses    CI_Loader::_ci_init_library()
     *
     * @param   string  $class      Class name to load
     * @param   mixed   $params     Optional parameters to pass to the class constructor
     * @param   string  $object_name    Optional object name to assign to
     * @return  void
     */
    protected function _ci_load_library($class, $params = null, $object_name = null)
    {
        // Get the class name, and while we're at it trim any slashes.
        // The directory path can be included as part of the class name,
        // but we don't want a leading slash
        $class = str_replace('.php', '', trim($class, '/'));

        // Was the path included with the class name?
        // We look for a slash to determine this
        if (($last_slash = strrpos($class, '/')) !== false) {
            // Extract the path
            $subdir = substr($class, 0, ++$last_slash);

            // Get the filename from the path
            $class = substr($class, $last_slash);
        } else {
            $subdir = '';
        }

        $class = ucfirst($class);

        // Is this a stock library? There are a few special conditions if so ...
        if (file_exists(BASEPATH.'libraries/'.$subdir.$class.'.php')) {
            return $this->_ci_load_stock_library($class, $subdir, $params, $object_name);
        }

        // Let's search for the requested library file and load it.
        foreach ($this->_ci_library_paths as $path) {
            // BASEPATH has already been checked for
            if ($path === BASEPATH) {
                continue;
            }

            $filepath = $path.'libraries/'.$subdir.$class.'.php';

            // Safety: Was the class already loaded by a previous call?
            if (class_exists($class, false)) {
                // Before we deem this to be a duplicate request, let's see
                // if a custom object name is being supplied. If so, we'll
                // return a new instance of the object
                if ($object_name !== null) {
                    $CI =& get_instance();
                    if (! isset($CI->$object_name)) {
                        return $this->_ci_init_library($class, '', $params, $object_name);
                    }
                }

                log_message('debug', $class.' class already loaded. Second attempt ignored.');
                return;
            }
            // Does the file exist? No? Bummer...
            elseif (! file_exists($filepath)) {
                continue;
            }

            include_once($filepath);
            return $this->_ci_init_library($class, '', $params, $object_name);
        }

        // One last attempt. Maybe the library is in a subdirectory, but it wasn't specified?
        if ($subdir === '') {
            return $this->_ci_load_library($class.'/'.$class, $params, $object_name);
        }

        // If we got this far we were unable to find the requested class.
        log_message('error', 'Unable to load the requested class: '.$class);
        show_error('Unable to load the requested class: '.$class);
    }

    // --------------------------------------------------------------------

    /**
     * Internal CI Stock Library Loader
     *
     * @used-by CI_Loader::_ci_load_library()
     * @uses    CI_Loader::_ci_init_library()
     *
     * @param   string  $library    Library name to load
     * @param   string  $file_path  Path to the library filename, relative to libraries/
     * @param   mixed   $params     Optional parameters to pass to the class constructor
     * @param   string  $object_name    Optional object name to assign to
     * @return  void
     */
    protected function _ci_load_stock_library($library_name, $file_path, $params, $object_name)
    {
        $prefix = 'CI_';

        if (class_exists($prefix.$library_name, false)) {
            if (class_exists(config_item('subclass_prefix').$library_name, false)) {
                $prefix = config_item('subclass_prefix');
            }

            // Before we deem this to be a duplicate request, let's see
            // if a custom object name is being supplied. If so, we'll
            // return a new instance of the object
            if ($object_name !== null) {
                $CI =& get_instance();
                if (! isset($CI->$object_name)) {
                    return $this->_ci_init_library($library_name, $prefix, $params, $object_name);
                }
            }

            log_message('debug', $library_name.' class already loaded. Second attempt ignored.');
            return;
        }

        $paths = $this->_ci_library_paths;
        array_pop($paths); // BASEPATH
        array_pop($paths); // APPPATH (needs to be the first path checked)
        array_unshift($paths, APPPATH);

        foreach ($paths as $path) {
            if (file_exists($path = $path.'libraries/'.$file_path.$library_name.'.php')) {
                // Override
                include_once($path);
                if (class_exists($prefix.$library_name, false)) {
                    return $this->_ci_init_library($library_name, $prefix, $params, $object_name);
                } else {
                    log_message('debug', $path.' exists, but does not declare '.$prefix.$library_name);
                }
            }
        }

        include_once(BASEPATH.'libraries/'.$file_path.$library_name.'.php');

        // Check for extensions
        $subclass = config_item('subclass_prefix').$library_name;
        foreach ($paths as $path) {
            if (file_exists($path = $path.'libraries/'.$file_path.$subclass.'.php')) {
                include_once($path);
                if (class_exists($subclass, false)) {
                    $prefix = config_item('subclass_prefix');
                    break;
                } else {
                    log_message('debug', APPPATH.'libraries/'.$file_path.$subclass.'.php exists, but does not declare '.$subclass);
                }
            }
        }

        return $this->_ci_init_library($library_name, $prefix, $params, $object_name);
    }

    // --------------------------------------------------------------------

    /**
     * Internal CI Library Instantiator
     *
     * @used-by CI_Loader::_ci_load_stock_library()
     * @used-by CI_Loader::_ci_load_library()
     *
     * @param   string      $class      Class name
     * @param   string      $prefix     Class name prefix
     * @param   array|null|bool $config     Optional configuration to pass to the class constructor:
     *                      FALSE to skip;
     *                      NULL to search in config paths;
     *                      array containing configuration data
     * @param   string      $object_name    Optional object name to assign to
     * @return  void
     */
    protected function _ci_init_library($class, $prefix, $config = false, $object_name = null)
    {
        // Is there an associated config file for this class? Note: these should always be lowercase
        if ($config === null) {
            // Fetch the config paths containing any package paths
            $config_component = $this->_ci_get_component('config');

            if (is_array($config_component->_config_paths)) {
                $found = false;
                foreach ($config_component->_config_paths as $path) {
                    // We test for both uppercase and lowercase, for servers that
                    // are case-sensitive with regard to file names. Load global first,
                    // override with environment next
                    if (file_exists($path.'config/'.strtolower($class).'.php')) {
                        include($path.'config/'.strtolower($class).'.php');
                        $found = true;
                    } elseif (file_exists($path.'config/'.ucfirst(strtolower($class)).'.php')) {
                        include($path.'config/'.ucfirst(strtolower($class)).'.php');
                        $found = true;
                    }

                    if (file_exists($path.'config/'.ENVIRONMENT.'/'.strtolower($class).'.php')) {
                        include($path.'config/'.ENVIRONMENT.'/'.strtolower($class).'.php');
                        $found = true;
                    } elseif (file_exists($path.'config/'.ENVIRONMENT.'/'.ucfirst(strtolower($class)).'.php')) {
                        include($path.'config/'.ENVIRONMENT.'/'.ucfirst(strtolower($class)).'.php');
                        $found = true;
                    }

                    // Break on the first found configuration, thus package
                    // files are not overridden by default paths
                    if ($found === true) {
                        break;
                    }
                }
            }
        }

        $class_name = $prefix.$class;

        // Is the class name valid?
        if (! class_exists($class_name, false)) {
            log_message('error', 'Non-existent class: '.$class_name);
            show_error('Non-existent class: '.$class_name);
        }

        // Set the variable name we will assign the class to
        // Was a custom class name supplied? If so we'll use it
        if (empty($object_name)) {
            $object_name = strtolower($class);
            if (isset($this->_ci_varmap[$object_name])) {
                $object_name = $this->_ci_varmap[$object_name];
            }
        }

        // Don't overwrite existing properties
        $CI =& get_instance();
        if (isset($CI->$object_name)) {
            if ($CI->$object_name instanceof $class_name) {
                log_message('debug', $class_name." has already been instantiated as '".$object_name."'. Second attempt aborted.");
                return;
            }

            show_error("Resource '".$object_name."' already exists and is not a ".$class_name." instance.");
        }

        // Save the class name and object name
        $this->_ci_classes[$object_name] = $class;

        // Instantiate the class
        $CI->$object_name = isset($config)
            ? new $class_name($config)
            : new $class_name();
    }

    // --------------------------------------------------------------------

    /**
     * CI Autoloader
     *
     * Loads component listed in the config/autoload.php file.
     *
     * @used-by CI_Loader::initialize()
     * @return  void
     */
    protected function _ci_autoloader()
    {
        if (file_exists(APPPATH.'config/autoload.php')) {
            include(APPPATH.'config/autoload.php');
        }

        if (file_exists(APPPATH.'config/'.ENVIRONMENT.'/autoload.php')) {
            include(APPPATH.'config/'.ENVIRONMENT.'/autoload.php');
        }

        if (! isset($autoload)) {
            return;
        }

        // Autoload packages
        if (isset($autoload['packages'])) {
            foreach ($autoload['packages'] as $package_path) {
                $this->add_package_path($package_path);
            }
        }

        // Load any custom config file
        if (count($autoload['config']) > 0) {
            foreach ($autoload['config'] as $val) {
                $this->config($val);
            }
        }

        // Autoload helpers and languages
        foreach (array('helper', 'language') as $type) {
            if (isset($autoload[$type]) && count($autoload[$type]) > 0) {
                $this->$type($autoload[$type]);
            }
        }

        // Autoload drivers
        if (isset($autoload['drivers'])) {
            foreach ($autoload['drivers'] as $item) {
                $this->driver($item);
            }
        }

        // Load libraries
        if (isset($autoload['libraries']) && count($autoload['libraries']) > 0) {
            // Load the database driver.
            if (in_array('database', $autoload['libraries'])) {
                $this->database();
                $autoload['libraries'] = array_diff($autoload['libraries'], array('database'));
            }

            // Load all other libraries
            foreach ($autoload['libraries'] as $item) {
                $this->library($item);
            }
        }

        // Autoload models
        if (isset($autoload['model'])) {
            $this->model($autoload['model']);
        }
    }

    // --------------------------------------------------------------------

    /**
     * CI Object to Array translator
     *
     * Takes an object as input and converts the class variables to
     * an associative array with key/value pairs.
     *
     * @param   object  $object Object data to translate
     * @return  array
     */
    protected function _ci_object_to_array($object)
    {
        return is_object($object) ? get_object_vars($object) : $object;
    }

    // --------------------------------------------------------------------

    /**
     * CI Component getter
     *
     * Get a reference to a specific library or model.
     *
     * @param   string  $component  Component name
     * @return  bool
     */
    protected function &_ci_get_component($component)
    {
        $CI =& get_instance();
        return $CI->$component;
    }

    // --------------------------------------------------------------------

    /**
     * Prep filename
     *
     * This function prepares filenames of various items to
     * make their loading more reliable.
     *
     * @param   string|string[] $filename   Filename(s)
     * @param   string      $extension  Filename extension
     * @return  array
     */
    protected function _ci_prep_filename($filename, $extension)
    {
        if (! is_array($filename)) {
            return array(strtolower(str_replace(array($extension, '.php'), '', $filename).$extension));
        } else {
            foreach ($filename as $key => $val) {
                $filename[$key] = strtolower(str_replace(array($extension, '.php'), '', $val).$extension);
            }

            return $filename;
        }
    }
}
?>
```

天哪实在看不下去了 这个 loader 负责的东西太特么多了 看到它既负责连接 database 还特么负责 autoload 还特么负责渲染 views

。。。

我崩溃了。。。

迅速逃离 CI
