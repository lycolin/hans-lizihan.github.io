---
layout:     post
title:      javascript Array 高级函数
date:       2015-03-19 23:00
summary:    map filter forEach sort
categories: javascript
---

> 数组高级函数

类似于 `PHP` 的数组高级函数 javascript 中更加原声地支持了各种 `数组高级函数`

相对来讲数组高级函数不光可以大大降低代码的行数 还可以非常良好地支持 javascript 中 chaining 的语法 并且支持 javascript 异步编程的精髓

> 注：本文中高级术组函数实现都是最简单的版本，没有 index 和 array 的功能

# `foreach`

`Array.prototype.forEach` 实现

``` javascript
Array.prototype.forEach = function(callback) {
    for(var i = 0; i < this.length; i ++) {
        callback(this[i]);
    }
};
```

`forEach` 实际用法

``` javascript
var test = [1,2,3,4,5];

test.forEach(function(value) {
    console.log(value);
});
// 1 2 3 4 5
```

# `map`

`Array.prototype.map`

``` javascript
Array.prototype.map = function(projection) {
  var result = [];

  // **this** refers to the Array
  this.forEach(function(item) {
      result.push(projection(item));
  });

  return result;
}
```

`map` 实际用法

``` javascript
var squares = [1,4,9,16,25];

console.log(squares.map(Math.sqrt))
// [1,2,3,4,5]
```

# `filter`


`filter` 实现

``` javascript
Array.prototype.filter = function(predicate) {
    var result = [];

    this.forEach(function(item) {
        if(predicate(item)) {
            result.push(item);
        }
    })

    return result;
}
```

`filter` 用法

``` javascript
var result = [1,3,4,5,6,6].filter(function(value) {
    return value > 5;
});

console.log(result);

// [6,6]
```

# `reduce`

`Array.prototype.reduce`

`reduce` 实现

``` javascript
Array.prototype.reduce = function (callback, initial) {
    var i = 0;
    if(arguments.length !== 2) {
        initial = this[0];
    } else {
        i --;
    }

    for(; i < this.length - 1; i ++) {
        initial = callback(initial, this[i+1]);
    }

    return initial;
}
```

`reduce用法`

javascript  中 reduce 使用和 php 十分相像 一般的用法是这样的

``` javascript
var test = [1,2,3,4];
var sum = test.reduce(function(pre, cur) {
    return pre += cur;
});

console.log(sum); //10
```

其中要注意的一点就是如果没有加入第二个参数，那么 `reduce` 会默认 pre 是数组的第一个元素

``` javascript
var test = [1,2,3,4];
var sumInitial10 = test.reduce(function(pre, cur) {
    return pre += cur;
}, 10);

console.log(sumInitial10); // 20
```

# `sort`

`sort` 的实现比较复杂，而且据说不同的厂商还有不同的实现，不同的 array 也有不同的待遇，比如说 纯数字 array 是用 C++的 `qsort` 实现的， 而带有字符串的 array 是通过 `merge-sort` 实现的

所以今天对于这个函数，实现方法就不写了，直接写用法吧。 另外， `sort()` 这个函数也不像其它那些函数直来直去，光用法就要看好久

``` javascript
var test = [1,2,3,5,2,10];
console.log(test.sort());
// 1 10 2 2 3 5
```

默认的 sort 会将数组中的所有元素当成是 字符串 进行 字典序 排序， 所以 `10` 紧紧跟在 `1` 后面

``` javascript
var test = ['a', 'ac','ab', 'abc'];
console.log(test.sort());
// a, ab, abc, ac
```

另外要注意的一个坑， `Array.map` `Array.filter` 会返回一个新的 数组， 而 `sort` 却是直接在数组中做操作，所以做完了之后 `test` 数组的顺序就直接变了

如果想要按照自己的意志 对 **纯数字数组** 排序，那么就要人手加入一个匿名函数，有点类似 C++ 的重载， 这样可以告诉 javascript 谁大谁小

``` javascript
var test = [1,10,3,4];
console.log(test.sort(function(a, b) {
    return a - b;
}));
// 1 3 4 10
```

简单来讲就是传入的匿名函数如果返回 负值 那么 `sort()` 就会升序排序， 可以讲 a b 理解成排序之后的紧邻的两个元素， `左 < 右` 所以 `a - b < 0`

但是 javascript 的 sort 没有这么简单就结束了， 如果说 数组中有 字符串的存在，那么就要用到完整版的 `compare()` 函数

``` javascript
function compare(a, b) {
    if (a is less than b by some ordering criterion) {
        return -1;
    }
    if (a is greater than b by the ordering criterion) {
        return 1;
    }
    // a must be equal to b
    return 0;
}
```

所以看到 其实 `a - b` 是一个对于 数字数组 的作弊写法 遇到字符串的时候就要乖乖自己写了

``` javascript
var test = ['ab', 'ac', 'bd', 'bc'];
test.sort(function(a, b) {
    if(a < b) {
        return -1;
    }

    if(a > b) {
        return 1;
    }

    return 0;
});

console.log(test);
// ab ac bc bd
```

因为 javascript 太特么动态了 所以导致 `'1' + 1 === '11' ` 这种情况是合法的，但是 `isNaN('ab' - 'ac') === true` 然后 `'ab' < 'ac' === true`

所以涉及到字符串比较的时候就要额外多思考下。。。

happy coding, may the code will always be with you~
