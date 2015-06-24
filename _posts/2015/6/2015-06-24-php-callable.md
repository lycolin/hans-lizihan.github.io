---
layout:     post
title:      php callable/callbacks 类型
date:       2015-06-24 12:56
summary:    php 基础之 callable/callbacks 类型
categories: php
---

这两天开始研究 spl 中的 autoload 原理。搞底层开发就要怒查官方文档，但是看来看去发现自己居然很长时间内都忽略了一个简单的问题，那就是官方文档中的 `callable` 是个啥意思?

好吧现在怒看回去还是来得及的。。。作为一只用了一年 php 的 php 狗居然连基本数据类型都没有了解清楚真是掩面而泣。。。

## 传入方式

所有的 `callable` 类型都可以用 `string` 的方式传入 __函数__。除了那些最最底层的 php 核心函数 `array`, `echo`, `empty`, `eval`, `exit`, `isset`, `list`, `print`, `unset()`。

``` php
<?php 
array_map('map_handle', [1,2,3]);

function map_handle($item) {
  echo $item;
}

// 等同于
array_map(function($item) {
  echo $item;
}, [1,2,3]);
?>
```

当我们想传入 __对象方法__ 的时候就要怒传入一个数组了。 这个时候分为5种情况

``` php
<?php
class Foo 
{
  public static function bar($item) {
    echo $item;
  }

  public function __invoke($item) {
    echo $item;
  }

  public function baz($item) {
    echo $item;
  }

  public function bat() {
    array_map([$this, 'baz'], [1,2,3]);
  }
}

class Fot extends Foo {}

// 1. 用数组传入静态方法
array_map(['Foo', 'bar'], [1,2,3]);
// 2. 用 :: 语法直接传入静态方法
array_map('Foo::bar', [1,2,3]);

$foo = new Foo;

// 3. 用数组传入实例方法
array_map([$foo, 'baz'], [1,2,3]);
// 3. 在类中call自己的方法（原理同上）
$foo->bat();
// 4. 在子类中叫父类（相对路径)
array_map(['Fot', 'parent::bar'], [1,2,3]);
// 5. 带有 __invoke 魔术方法
array_map($foo, [1,2,3]);
?>
```

除此之外还可以像 js 那样怒传闭包

``` php
<?php 
$double = function($item) {
  return 2 * $item;
}

$result = array_map($double, [1,2,3]);

// 2 4 6
?>
```

终于真相大白

* php [官方手册](http://php.net/manual/en/language.types.callable.php)


