---
layout:     post
title:      php 数组高级函数
date:       2015-03-16 19:28
summary:    php 高级函数
categories: PHP
---

> 数组高级函数

PHP 中数组的高级函数可以优雅地解决不少问题 可惜 php.net 上面的文档实在是云里雾里，要理解很久（至少对初学者）

不如直接举个非常直白的栗子
最本能的遍历实现我们一般都通过 `foreach` 实现:

假设我们现在有一个文件名的数组

``` php
<?php
$filenames = [
  'man',
  'woman',
  '李凌飞',
];
```

但是这一堆文件可能有很多种存储格式 .sql, .md, .json, .xml ...

假设现在的任务是读取 .json, 那么最本能地

``` php
<?php
function jsonFileNames() {
  $result = [];

  foreach($filenames as $filename) {
    $result[] = $filename. '.json';
  }

  return $result;
}
```

可以接受， 但是总是觉得怪怪的，好像没有那么优雅的样子 `$result` 出现了 `3` 次

俗话说的好

> 生命诚可贵，爱情价更高，若为装逼故，二者皆可抛

# `array_map`

``` php
<?php
function jsonFileNames() {
  return array_map(function($file) {
    return $file . '.json';
  }, $filenames);
}
```

输出结果一模一样，但是没有了中间变量 `result` 这个小三，神清气爽 装逼成功

更进一步:

``` php
<?php
function returnFileNamesByType ($type) {
  return array_map(function($file) use ($type) {
    return "$file.$type";
  }, $filenames);
}
```

注意

* `array_map` 闭包中只接受一个或者多个参数，闭包的参数数量和 `array_map` 本身的参数数量必须一致

``` php
<?php
$array1 = ['李凌飞', '他'];
$array2 = ['是', '需要'];
$array3 = ['单身狗', '女朋友'];
print_r (
  array_map(fucntion($value1, $valu2, $value3) {
    return $value1 . $value2 . $value3;
  }, $array1, $array2, $array3);
);
/* [
 *    ['李凌飞是单身狗'],
 *    ['他需要女朋友'],
 * ]
 */
```

* `array_map` 如果想在操作 content 的同时操作 key，那么简单借助 `array_keys` 即可

``` php
<?php
$input = ['key' => 'value'];
array_map(function($key, $value) {

  echo $key . $value;
}, array_keys($input), $input)
// 'keyvalue'
```

* 还有一些小的用法很好，比如说有一个数组的 `float` 数

``` php
<?php
$floats = [13.12, 12.11, 22.22];
$integers = array_map('intval', $floats);
// $integers = [13, 12, 22];
```

类似于 `array_map`, `array_reduce` 作为另外一个 数组高级函数，也可以非常优雅地解决一些问题

# `array_reduce`

`array_reduce` 最典型的 usecase 是 计算 数组中的和

``` php
<?php
$raw = [1,2,3,4,5,];
array_reduce($raw, function($result, $value) {
  return $result += $value;
})
// 15
```

很简单，但是比较实用，同样的如果用古典的 `foreach` 就会有一些 `$tmp` 看起来没有那么美好

``` php
<?php
$raw = [1,2,3,4,5,];

$sum = 0;
foreach($raw as $value) {
  $sum += $value;
}
$sum;
// 15
```
相比出现3次 `sum` 这种中间量来说我个人还是更加喜欢 array_reduce 这种简洁的感觉

最后 `array_reduce` 的第三个参数是 $result 的初始值

``` php
<?php
$raw = [1,2,3,4,5,];

array_reduce($raw, function($result, $value) {
  $result[$value] = $value;

  return $result;
}, []);
// [1 => 1, 2 => 2, ... 5 => 5]
```
借助 `array_reduce` 我有时候还会嵌套一些 `array_filter` 的效果(其实因为太懒了没有改成 `array_filter`)

只要简单加些 `if else ... ` `array_reduce` 就可以华丽变身成很好理解的 `array_filter`, (如果愿意，`array_reduce` 还可以达到 `array_map` 的效果)

这种灵活性让我挺爱用这个函数的，可以无痛将 `array_reduce` 转化成 `foreach` 还避免了中间量的尴尬

# `array_filter`

这个函数最常用的情况是 过滤数组中的 `null`

``` php
<?php
$test = [null, null, 1, 2, null];

array_filter($test);

// [2 => 1, 4 => 2];
```

更加常见的使用是从一个单维数组里面挑出来符合条件的元素

``` php
<?php
$test [1,2,3,4,5];

array_filter($test, function($content) {
  return $content > 3;
});

// [4,5];
```

happy coding, may the code will always be with you~
