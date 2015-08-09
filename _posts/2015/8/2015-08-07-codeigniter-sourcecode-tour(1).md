---
layout:     post
title:      php CI 源码探索(1)
date:       2015-08-07 18:52
summary:    php CI 代码解读(1)
categories: php CodeIgniter
---

## 重拾 CI3 

回来看一看发现开始研究源代码之后收获良多。很多之前看不懂的地方现在终于可以看明白了。

同时也发现了 CI 的实现简直太不优雅了，全篇满眼望去全都是重复的代码，全都是用 `define()` 定义的全局变量。现在回头来看终于明白了为什么之前大家都喷 php 的开发了，因为旧版本的php这样实现起来简直惨不忍睹。

(还有每一个文件夹都有一个 index.html 说不许直接访问。。。。,每个phpfile 都要打一句 (defined('BASEPATH') OR exit('No direct script access allowed');) 真是醉了。。。。

不过无论如何 不可否认的是 CI 这个框架对于现代的 php 框架发展还是有着很多的借鉴作用的。此外由于它惨不忍睹的实现和不拥抱社区的理念，CI 自己倒是踏踏实实地撸了一套无依赖的库来调用，这点我真的挺佩服的。

有时候不太想用 laravel 就是因为它过分依赖外部的 `vendor` 了，导致太多的细节都被忽略在了深层的 `vendor` 里面，有点让人望而生畏。 相比之下 CI 的代码量就很少了，更主要的就是没有依赖所以阅读起来能学到很多的东西。所以感觉学 php 底层 API， CI 这款框架确实是个很好地入手点。

## index.php

这个是整个 CI 系统的系统入口。 代码不多 一段段来

``` php
<?php 
/**
 *---------------------------------------------------------------
 * 应用环境应用
 *---------------------------------------------------------------
 *
 * 三个选项:
 *
 *     development
 *     testing
 *     production
 * 
 * e.g. 在环境中 .bashrc 中 export CI_ENV=development 就可以改变这个环境了
 */
define('ENVIRONMENT', isset($_SERVER['CI_ENV']) ? 
  $_SERVER['CI_ENV'] 
  : 'development');
?>
```

这段主要就是仿照 ROR 来定义下系统执行环境。 

``` php
<?php
/*
 *---------------------------------------------------------------
 * 错误报告
 *---------------------------------------------------------------
 * php 语言级别错误
 *
 */
switch (ENVIRONMENT) {
    case 'development':
        error_reporting(-1);
        ini_set('display_errors', 1);
    break;

    /*
     * PHP 5.3 or later, the default value is 
     * E_ALL & ~E_NOTICE & ~E_STRICT & ~E_DEPRECATED. 
     * This setting does not show E_NOTICE, E_STRICT 
     * and E_DEPRECATED level errors. 
     * 
     * Prior to PHP 5.3.0, the default value is 
     * E_ALL & ~E_NOTICE & ~E_STRICT. 
     * 
     * In PHP 4 the default value is E_ALL & ~E_NOTICE.
     */
    case 'testing':
    case 'production':
        ini_set('display_errors', 0);
        if (version_compare(PHP_VERSION, '5.3', '>=')) {
            error_reporting(E_ALL & ~E_NOTICE & ~E_DEPRECATED & ~E_STRICT & ~E_USER_NOTICE & ~E_USER_DEPRECATED);
        } else {
            error_reporting(E_ALL & ~E_NOTICE & ~E_STRICT & ~E_USER_NOTICE);
        }
    break;

    default:
        header('HTTP/1.1 503 Service Unavailable.', true, 503);
        echo 'The application environment is not set correctly.';
        exit(1); // EXIT_ERROR
}
?>
```

开始长见识了。 其中 `& ~E_STRICT` 翻译一下的意思就是不要这个东西，=> `and not E_STRICT`

简单看下 `E_STRICT` 这个对应 javascript 中的 `use strict`，旨在强迫用户不要用php 中的糟粕部分。可惜CI为了兼容之前的版本和培养更多二笔php程序员来继续抹黑 php 没有开启这个选项。

``` php
<?php
// 框架代码在 'path/to/ci/root/folder/' . $system (相对路径于root)
$system_path = 'system';

// 程序代码在 'path/to/ci/root/folder/' . $application
$application_folder = 'application';

// 1. 不填东西， fallback 到了 `application/views` 文件夹
// 2. 填一个绝对路径
$view_folder = '';

/**
 * 默认 Controller 这段代码暂时用不到 等后面看到了 routing 再回来找这段
 *
 * 个人感觉 $routing 这种全局变量极其恶心啊
 */

// The directory name, relative to the "controllers" folder.  Leave blank
// if your controller is not in a sub-folder within the "controllers" folder
// $routing['directory'] = '';

// The controller class file name.  Example:  mycontroller
// $routing['controller'] = '';

// The controller function you wish to be called.
// $routing['function'] = '';


/**
 * 个性化 config 依然放到后面的 Config 去一起看
 * 
 * 依然觉得这样 $assign_to_config 这种全局有些不妥
 */
// $assign_to_config['name_of_config_item'] = 'value of config item';
?>
```

这段就是一些 config 了， `$routing` 和 `$assign_to_config` 两个选项现在完全看不出来有什么卵用，所以先放在这里，以后看到了 routing 和 config 两个核心的时候再回来嚼.

``` php
<?php
/**
 * 路径设定(很多兼容性考虑)
 */

// Set the current directory correctly for CLI requests
if (defined('STDIN')) { // STDIN 是使用 php-cli 特有 的一个全局常量
    chdir(dirname(__FILE__)); // 旧的写法 (pre5.3) 新的写法直接 __dir__
}

// realpath 会返回一个文件的真实路径, e.g. /user/../user -> /user
// 也会查看当前路径下的 $filename 如果存在就返回这个文件的绝对路径，如果不存在就返回 null

// 上面 config 过了 $system_path = 'system'
// realpath -> /path/to/ci/system
// 如果没有找到这个文件夹那么就会返回 false
if (($_temp = realpath($system_path)) !== false) { 
    $system_path = $_temp.'/';
} else {
    // 如果出于任何问题上面的 `if` block 反悔了 falsy value 那么做最后的尝试

    // 确保 $system_path 有一个结束的 `/`

    // 1. 将末尾的所有 `/` 都去掉
    // 2. 在字符串末尾最后加上一个 `/`
    $system_path = rtrim($system_path, '/').'/';
}


// 如果没找着那么就返回一个 503
// p.s. 这个 error handling 完全可以写成一个函数嘛(不过这样也是可以接受的)
if (! is_dir($system_path)) {
    header('HTTP/1.1 503 Service Unavailable.', true, 503);
    echo 'Your system folder path does not appear to be set correctly. Please open the following file and correct this: '.pathinfo(__FILE__, PATHINFO_BASENAME);

    // pathinfo: 
    // filename: String 
    // options: oneOf 
    //  -PATHINFO_DIRNAME: /Dir/To/File/
    //  -PATHINFO_BASENAME: e.g. app.inc.php -> php.inc.php
    //  -PATHINFO_EXTENSION: e.g. app.inc.php -> php
    //  -PATHINFO_FILENAME: e.g. app.inc.php -> php.inc


    exit(3); // EXIT_CONFIG
}
```

这段显而易见就是在构造一个 Framework Basepath了

``` php
<?php
/*
 * -------------------------------------------------------------------
 *  Now that we know the path, set the main path constants
 * -------------------------------------------------------------------
 */
// The name of THIS file
// 偏见: 定成 `INDEX_FILE` 多好
define('SELF', pathinfo(__FILE__, PATHINFO_BASENAME));

// Path to the system folder
// 依然非常讨厌这种表里不一的命名 写成 `SYSTEM_PATH` 多好
define('BASEPATH', str_replace('\\', '/', $system_path));

// Path to the front controller (this file) 
// BTW 真的很讨厌这种简称 写成 `FRONT_CONTROLLER_PATH` 会死么？
define('FCPATH', dirname(__FILE__).'/');

// Name of the "system folder"
// 这个缩写可以忍
define('SYSDIR', trim(strrchr(trim(BASEPATH, '/'), '/'), '/'));

// The path to the "application" folder
// application/
if (is_dir($application_folder)) {
    if (($_temp = realpath($application_folder)) !== false) {
        $application_folder = $_temp;
    }

    define('APPPATH', $application_folder.DIRECTORY_SEPARATOR);
} else {
    if (! is_dir(BASEPATH.$application_folder.DIRECTORY_SEPARATOR)) {
        header('HTTP/1.1 503 Service Unavailable.', true, 503);
        echo 'Your application folder path does not appear to be set correctly. Please open the following file and correct this: '.SELF;
        exit(3); // EXIT_CONFIG
    }

    // 默认到to system/application/
    define('APPPATH', BASEPATH.$application_folder.DIRECTORY_SEPARATOR);
}
?>
```

设置 `application folder` 如果么有 `application` 这个文件夹就直接 fallback 到 `system/application` 这个文件夹

``` php
<?php
// The path to the "views" folder
// 默认情况下 $view_folder 是 ''
// 所以默认情况下我们就进到了这个 block
if (! is_dir($view_folder)) {
    // 假设我们设置了 $view_folder 叫做 `foo`
    // 看了下 /path/to/application/foo/ 这个文件夹是否存在
    if (! empty($view_folder) && is_dir(APPPATH.$view_folder.DIRECTORY_SEPARATOR)) {
        // $view_folder 就变成了 /path/to/application/foo
        $view_folder = APPPATH.$view_folder;
    } elseif (! is_dir(APPPATH.'views'.DIRECTORY_SEPARATOR)) {
        // 如果说我们没有创建 `application/foo` 这个文件夹那么应该报错
        header('HTTP/1.1 503 Service Unavailable.', true, 503);
        echo 'Your view folder path does not appear to be set correctly. Please open the following file and correct this: '.SELF;
        exit(3); // EXIT_CONFIG
    } else {
        // ah-ha 这里定义了默认的 views folder 叫做 `views`
        $view_folder = APPPATH.'views';
    }
}

// 针对 winserver 的差错代码
// (感觉这个逻辑用的很多应该专门在前面抽下下然后直接调用)
if (($_temp = realpath($view_folder)) !== false) {
    $view_folder = $_temp.DIRECTORY_SEPARATOR;
} else {
    $view_folder = rtrim($view_folder, '/\\').DIRECTORY_SEPARATOR;
}

define('VIEWPATH', $view_folder);
?>
```

这段主要就是定义了 `VIEWPATH` 这一个全局 常量

``` php
<?php
require_once BASEPATH.'core/CodeIgniter.php';
?>

不用解释了这就直接进入了 CI 的核心了。

总结:

以批判的角度来看 CI 的 index.php 写得太业余了。

看看它 define 了多少全局常量就知道了。

``` php
<?php
print_r(get_defined_constants(1));
?>
```

在 `user` 这个分类里面这么多全局的常量确实不太科学

感觉这完全是在用 php4 的风格在写 php5, 怪不得现在已经被人基本抛弃了。

回头看下实现比较良好的 `laravel` 

同样地函数

`user` 这个分类里面只有确实应该设为全局常量的 `LARAVEL_START`

哎，往事不堪回首 想一想 CI 的开发人员维护了这么久的旧版本怪不容易的，不如得饶人处且饶人吧
