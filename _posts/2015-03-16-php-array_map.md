---
layout:     post
title:      array_map
date:       2015-03-16 19:28
summary:    php 函数 array_map
categories: PHP
---

> 数组高级函数

PHP 中数组的高级函数可以优雅地解决不少问题 可惜 php.net 上面的文档实在是云里雾里，要理解很久（至少对初学者）

不如直接举个非常直白的栗子
最本能的遍历实现我们一般都通过 `foreach` 实现:

假设我们现在有一个文件名的数组

```php
$filenames = [
    'man',
    'woman',
    '李凌飞',
]
```

但是这一堆文件可能有很多种存储格式 .sql, .md, .json, .xml ...

假设现在的任务是读取 .json, 那么最本能地

```php
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

```php
function jsonFileNames() {

    return array_map(function($file) {

        return $file . '.json';
    }, $filenames);
}
```

输出结果一模一样，但是没有了中间变量 `result` 这个小三，神清气爽 装逼成功

更进一步:

```php
function returnFileNamesByType ($type) {

    return array_map(function($file) use ($type) {

        return "$file.$type";
    });
}
```

注意
1. `array_map` 闭包中只接受一个或者多个参数，闭包的参数数量和 `array_map` 本身的参数数量必须一致
```php
$array1 = ['李凌飞', '他'];
$array2 = ['是', '需要'];
$array3 = ['单身狗', '女朋友'];
print_r (
    array_map(fucntion($value1, $valu2, $value3) {

        return $value1 . $value2 . $value3;
    }, $array1, $array2, $array3);
)
/* [
 *    ['李凌飞是单身狗'],
 *    ['他需要女朋友'],
 * ]
 */
```
2. `array_map` 如果先在操作 content 的同时操作 key，那么简单借助 `array_keys` 即可
```php
$input = ['key' => 'value'];
array_map(function($key, $value) {

    echo $key . $value;
}, array_keys($input), $input)
// 'keyvalue'
```
3. 还有一些小的用法很好，比如说有一个数组的 float (同理可以将 `intval` 转化成其他的单参数带返回值函数)
```php
$floats = [13.12, 12.11, 22.22];
$integers = array_map('intval', $floats);

// $integers = [13, 12, 22];
```

happy coding, may the code will always be with you~
