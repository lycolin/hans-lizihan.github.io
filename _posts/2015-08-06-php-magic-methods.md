---
layout:     post
title:      php 魔术方法
date:       2015-08-06 22:52
summary:    php 基础之 魔术方法
categories: php
---

php 的一大特色应该就是以两个 `__` 开头的魔术方法了。

魔术方法有 这些，虽然很多魔术方法一般都没有什么机会碰到但是有时候研究源代码的时候碰到了会楞

- `__construct`
- `__destruct`
- `__call`
- `__callStatic`
- `__get`
- `__set`
- `__isset`
- `__unset`
- `__toString`
- `__invoke`
- `__clone`
- `__awake`
- `__sleep`
- `__set_state`
- `__debugInfo`


## construct

毫无疑问调用频率最高的魔术方法。该方法在 `new` 关键字创建对象实例的时候被调用。通常包含一些实例的初始化逻辑

## desctruct

也很好理解 但是这里有个坑，根据手册描述

> The destructor method will be called as soon as there are no other references to a particular object, or in any order during the shutdown sequence.

这个跟 js 有点类似。就是说如果

``` php
<?php
class Foo {
  public $name;
  public $foo;
  public function __construct($name) {
    $this->name = name;
  }
  public function setLink(Foo $foo) {
    $this->foo = $foo;
  }
  public function __destruct() {
    echo 'destructing';
  }
}

$foo1 = new Foo('Foo 1');
$foo2 = new Foo('Foo 2');
$foo1->setLink($foo2);
$foo2->setLink($foo1);

$foo1 = null;
$foo2 = null;

// break
?>
```

因为 php 的 gc 也是看指向对象的指针来做的。所以看上面的代码，当 foo1 和 foo2 都是 null 的时候我们只是销毁了外层中 foo1 和 foo2 的指针。

然而之前创建的 foo1 中依然保留着 foo2 的实例。 foo2 中也保留着 foo1 的实例。所以这两个指针没有被设为 null 所以 gc 不会回收 foo1 和 foo2 -> `__destruct` 就不会被销毁。

这种时候有一个 很底层的 function `gc_collect_cycles()` 可以解决这个问题，这个函数简单讲就是看下所有不可以被 references 的内存，然后回收掉。

当然这种情况比较少见了。通常代码中不会有这种二笔依赖，不过也要留意下。

## call, callStatic

这两个方法可以说是很多黑魔法的霍乱之源 举个例子就是 `Eloquent` 里面 `Model.php` 这个类

我们知道 `$post->find(1)` 会找到 id 为 1 的数据。

然而查找源代码 `Model.php` 中并没有 `find` 这个方法。

继续调查发现

```php
<?php
/**
 * Handle dynamic method calls into the model.
 *
 * @param  string  $method
 * @param  array   $parameters
 * @return mixed
 */
public function __call($method, $parameters) {
  if (in_array($method, ['increment', 'decrement'])) {
    return call_user_func_array([$this, $method], $parameters);
  }

  $query = $this->newQuery();

  return call_user_func_array([$query, $method], $parameters);
}

/**
 * Handle dynamic static method calls into the method.
 *
 * @param  string  $method
 * @param  array   $parameters
 * @return mixed
 */
public static function __callStatic($method, $parameters) {
  $instance = new static;

  return call_user_func_array([$instance, $method], $parameters);
}
?>
```

翻译一下

1. 用户调用 `Post::find(1)`
2. php 发现在 `Post` 和 `Model` 类及其父类中并没有 `find` 静态方法
3. 那么就找到了 `__callStatic`
4. `callStatic` 此时新建了一个自己的实例，并且在此实例上面调用了 `find` 方法。
5. php 发现在 `Post` 和 `Model` 及其父类中没有 `find` 实例方法
6. 那么就找到了 `call`

后面的故事太长了改天慢慢看 `Model` 研究 eloquent

总之上面的例子很好地说明了 `call` 和 `callStatic` 的实际用途

p.s.1: `call_user_func_array`
 这个方法，这个在 `callable` 里面看过一部分。
第一个参数 `[array]|[string]` 是方法的名称或者 `instance => method`
第二个参数  `[array]` 是一个索引数组(不是 assoc array)，作为被叫函数的参数们

p.s.2: `call` 和 `callStatic` 第一个参数是方法名，第二个参数是索引数组，为该方法的参数
e.g.: `find(['name' => 'Hans'])` => `$method = find`, `$parameters = [['name' => 'Hans']]`

## get set

getter 和 setter 永远都是 oop 语言里面比较重要的两个概念。正常来讲我们可以用 `getName()`, `setName(name)` 这两个方法来做 getter 和 setter，但是有时候当两个 setter的逻辑一模一样的时候这样的 setter 就要重复两次。除此以外如果一个 property 真的很简单，那么没有理由对每一个 property 都写一个 getter method。

所以和 `__call` 一样 `Post::find(1)->name` 属性会经历下面这些步骤

1. 发现 `name` 并不是 public property
2. 来到了 `__get($key)`

setter 同理

`Post::find(1)->name = 'hans'`

1. 发现 `name` 不是 public property,
2. 来到了 `__set(key, value)`

## isset unset

这两个相对少用。

独立于 `get` 和 `set`

### isset

当外部调用 (isset | empty) 于不可读属性时，可以调用 `__isset` 来截获这个操作并且加入自己的逻辑。

e.g. `isset(Post::find(1)->name)`

### unset

当外部调用 `unset` 函数与不可读属性时，调用 `__unset` 来截获这个操作


## toString

这个也很常用

当外部有方法调用实例并将实例认为是 `String` 的时候这个函数被调用

`echo Post::find(1)`

## invoke

之前也 cover 过。

当一个实例 被用作 `工厂方法` 的时候被调用

## clone

clone 作为一个关键字，是 php 比较特殊的一个。实际操作里面很少遇到 `clone` 的问题。

用法是这样的

``` php
$cloned = clone $instance;
```

这种时候有两种可能。
1. 没有 `__clone` 方法
2. 有 `__clone` 方法，该方法在 clone 过程结束之后被调用，用作 hook

## awake sleep

这两个方法跟一个函数 `serialize` `unserialize` 紧密相连。

``` php
<?php
class Test {
  public function __construct($name = 'hans') {
    $this->name = $name;
  }
}

$test = new Test;

echo serialize($test);
// O:4:"Test":1:{s:4:"name";s:4:"hans";}
?>
```

简单来讲 `serialize` 函数可以将一个 php 数据结构转化为一条 `String`
`unserialize` 函数可以将一个 `serialize` 过的 `String` 转化回 php 数据结构。

可以参照 javascript 的 `JSON.stringify()` 和 `JSON.parse()`


简单讲 `sleep` 就是在 `serialize` 函数作用于某一个实例上面之前被调用的。

``` php
<?php
class Test {
  public function __construct($name = 'hans') {
    $this->name = $name;
  }
  public function __sleep() {
    return [];
  }
}

$test = new Test;

echo serialize($test);
// O:4:"Test":0:{}
?>
```

注意

> \_\_sleep() 魔术方法必须返回一个装有所有即将被 serialize 的 **索引数组**

`__awake` 简单讲就是重新构造的时候的一个钩子

这个方法不返回任何东西，只是作为一个钩子

``` php
<?php
/**
 * When a model is being unserialized, check if it needs to be booted.
 *
 * @return void
 */
public function __wakeup() {
    $this->bootIfNotBooted();
}
```

举例讲就是这样的。在 unserialize 之后重新构造该类需要的一些东西。

## set_state debugInfo

好吧我承认这个命名十分恶心

不过这两个分别是 `var_export` 和 `var_dump` 作用于实例时的钩子 用的不多

* php [官方手册](http://php.net/manual/en/language.oop5.magic.php)
