---
layout:     post
title:      Javascript concat apply
date:       2015-04-23 23:40
summary:    Javascript concat reduce array_flatten
categories: javascript
---

# Array.prototype.concat

> The concat() method returns a new array comprised of the array on which it is called joined with the array(s) and/or value(s) provided as arguments.

在最近做一个数据分析的 API 的时候 经常调用一个 `laravel` 内部实现的 helper function `array_flatten`

好奇之下探索源代码之后发现实现原理极为简单（PHP 是世界上最好的语言。。。 恩。。。 ）

``` php
<?php
function array_flatten($arr) {
    $result = [];

    array_walk_recursive($arr, function($value) use (&$result) {
        $result[] = $value;
    });

    return $result;
}
```

好吧，javascript 没有这么奢侈的 `array_walk_recursive` 那么只能自己手撸出来了

`recursive` 一看就是 传说中的 recursion function

``` javascript

function array_flatten(arr) {
  var result = [];
  for(var i = 0; i < arr.length; i ++) {
    // if we encountered an array, call recursively
    if(Array.isArray(arr[i])) {
      var flattened = array_flatten(arr[i]);
      for(var j = 0; j < flattened.length; j ++) {
        result.push(flattened[j]);
      }
    } else {
      result.push(arr[i]);
    }

  }
  return result;
}
```

实际上这种实现方法跟 `underscore.js` 比较接近。 而且参考这篇文章 (http://jsperf.com/flatten-an-array-loop-vs-reduce)[loop-vs-reduce] 似乎在性能上面原生的 javascript loop 要好一些

（虽然recursive 最终都可以转化成 iterative 做进一步优化， 但是我觉得上面的代码可读性更强更好理解所以就不再多麻烦了（实际上是懒得钻研了暂时））

上面的方法暂时已经够用了，但是有时候其实情况并没有那么糟糕，有时候并不需要用 10行 代码完成一个完整的 `array_flatten` ，有时候只是仅仅要将一个多维数组的维度降低一级

比如这样


``` javascript
[1,[2,3],[4,5]] => [1,2,3,4,5]
```

这时候有一种非常优雅的写法，那就要引出 `Array.prototype.concat`

> ```javascript
  var new_array = old_array.concat(value1[, value2[, ...[, valueN]]])
  ```

函数很好理解，就是简单地将新的 array 拼接到了 旧的 array 后面

``` javascript
[1,2,3].concat([3,4,5]);
// [1,2,3,3,4,5]
```

调查 MDN 之后发现 concat接受无限多个参数并把它们按照顺序拼接起来

知道了这点之后终于引出今天的奇技淫巧 那就是 array 降维操作

``` javascript
var test = [1,2,3,[4,5,6],[7,8]];
[].concat.apply([], test); // [1,2,3,4,5,6,7,8]
```

这里面又已入了一个新的 概念：`apply`

# Function.prototype.apply

> The apply() method calls a function with a given this value and arguments provided as an array (or an array-like object).

> ``` javascript
  fun.apply(thisArg, [argsArray])
  ```

OK 原来是 `call` 的兄弟。只不过 `call` 是接受的一个个 args 但是 `apply` 相当于直接接受了一个 `arguments` 得到的类数组函数，或者直接就是真实的数组

所以 `[].concat.apply([], test)` 的意思其实是 对于 `test` 数组中的每一个 value， 将它 `concat` 到空数组 [] 中去，而因为 concat 是 Array 的 prototype，所以我们用一个空 array 作载体

类似地，用 `reduce` 可以得到一样的结果

这其实被 MDN 写在了 `reduce` 的例子中

``` javascript
var test = [[1,2],3,4,[5,6]];

var result = test.reduce(function(pre, cur) {
    return pre.concat(cur);
},[]);
// [1,2,3,4,5,6];
```

当然了，这种方法虽然简单优雅，但是功能有限，如果想要实现完整的 `flatten` 功能， `reduce` 的写法实际上跟 `for` 差不多的

``` javascript
var test1 = [1,2,[3,[4,[5]]]];

function flatten(arr) {
   return arr.reduce(function(pre, cur) {
        if(Array.isArray(cur)) {
            return flatten(pre.concat(cur));
        }

        return pre.concat(cur);
   }, []);
}

// [1,2,3,4,5]
```

感觉 `reduce` 怎么看都比 `for` 清爽一点 （个人喜好，勿喷）

happy coding, may the code will always be with you~
