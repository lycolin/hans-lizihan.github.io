---
layout:     post
title:      php 语法糖
date:       2015-08-04 12:12
summary:    php 基础之 新版本语法糖
categories: php
---

php 5.6 成为当前稳定版本很久了，随着新版本发布，很多语法糖也随之面世，之前非常繁琐的php操作现在可能都有了更好的语法糖做替代。

## ::class

`::class` 这个操作符 可以用来迅速地取得一个类的字符串 `fullname`, 就是带有民命空间的类名。

做一个实验

``` php
<?php
Reflection::class; //  => 'Reflection'
\App\Post::class; // => 'App\Post'
use App\Post;
Post::class; // => 'App\Post'
?>
```

可以在 laravel 5.1 中看到 `::class` 已经被广泛使用了，尤其是在 `app\config.php` 里面，以前基于字符串的 `serviceprovider` 现在已经全都变成了 `::class`.(据说是为了ide友好)


## .. (splat)

这个是从 ruby 借鉴过来的。虽然没有 ruby 支持的那么好(任何地方1..5 => range(1,5)) 但是在 function里面还是很有用的，很多 function 都从 `func_get_args` 中被解放出来了。

``` php
<?php
// old:
function test($param1, $param2 = null, $param3 = null) {
  $params = func_get_args();
  var_dump($params);
}

test(1,2,3); // prints [1,2,3]
test(1,2); // prints [1,2]
test(1); // prints [1]

// new:
function test($param1, ...$rest) {
  var_dump($rest);
}

test(1,2,3); // $rest = [2,3];
?>
```

除此之外还有 unpacking 的简单写法

``` php
<?php
$rest = [1,2,3,4];
function add($param1, $param2, $param3, $param4) {
  return $param1 + $param2 + $param3 + $param4;
}

add(...$rest); // 10
?>
```

## :? (short hand ternary)

这个应该是我最近偶然发现的惊喜了。

正常情况下一个三元经常这么写；意思就是如果有 $judge 不为空则将 $result 赋值为 $judge
``` php
<?php
$result = $judge ? $judge : 'something else';
?>
```

有一个语法糖可以做到相同的事儿

``` php
<?php
$result = $judge ?: 'something else';
?>
```
* php [官方手册](http://php.net/manual/)
