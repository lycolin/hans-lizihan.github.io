---
layout:     post
title:      array_reduce
date:       2015-03-17 17:00
summary:    php 函数 array_reduce
categories: PHP
---

> 数组高级函数

类似于 `array_map`, `array_reduce` 作为另外一个 数组高级函数，也可以非常优雅地解决一些问题

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

这种灵活性让我挺爱用这个函数的，可以无痛讲 `array_reduce` 转化成 `foreach` 还避免了中间量的尴尬

happy coding, may the code will always be with you~
