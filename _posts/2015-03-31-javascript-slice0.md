---
layout:     post
title:      javascript Array#slice
date:       2015-03-31 23:00
summary:    Array.prototype.slice()
categories: javascript
---

# slice() 函数

javascript 数组操作中， `slice` 占据着极为重要的地位，因为这个函数总是被作为简便拷贝(浅拷贝)的实现。

## 浅拷贝

浅拷贝其实很简单，其实就是说，这个拷贝只负责拷贝一层， i.e. 单维数组首层拷贝

``` javascript
// show down clone

var test = [1,2,3,4,5,[5,6]];
var clone = [];
for(var i = 0; i < test.length; i++) {
    clone[i] = test[i];
}

console.log(clone); // [1,2,3,4,5,[5,6]];

test[0] = 0;

console.log(clone); // [1,2,3,4,5,[5,6]];

test[5][0] = 10;

console.log(clone); // [1,2,3,4,5,[10,6]];
```

`array.slice` 正是实现了 `浅拷贝`

套用 `Airbnb javascript guideline`

> When you need to copy an array use Array#slice.

``` javascript
var len = items.length;
var itemsCopy = [];
var i;

// bad
for (i = 0; i < len; i++) {
    itemsCopy[i] = items[i];
}

// good
itemsCopy = items.slice();
```

我们可以看到，用 `slice()` 方法，代码更加简洁

> To convert an array-like object to an array, use Array#slice.

``` javascript
function trigger() {
    var args = Array.prototype.slice.call(arguments);
    ...
}
```

为啥拷贝的时候要用 slice() 呢？

这是 MDN 文档对于 slice 说明

> The slice() method returns a shallow copy of a portion of an array into a new array object.

``` javascript
arr.slice([begin[, end]])
```

可以非常粗暴地将 `slice` 方法理解成一个简单的 `for` loop

它的实现和上文中的 `浅拷贝` 一模一样

``` javascript
Array.prototype.slice() = function (begin, end) {
    var result = [];

    for(var i = begin; i < end; i ++) {
        result.push(this[i]);
    }

    return result;
}
```

``` javascript
var test = [1,2,3,4];

var clone = test.slice();

console.log(clone); // [1,2,3,4];
```

如果没有参数，那么 `slice` 默认拷贝整个 array

``` javascript
var test = [1,2,3,4];

var clone = test.slice(1);

console.log(clone);// [2,3,4]
```

如果只提供了一个参数，那么 `slice` 默认从 `start` 开始一直拷贝的结尾

``` javascript
var test = [1,2,3,4];

var clone = test.slice(1, 2);

console.log(clone); // [2,3];
```

如果给了两个参数而且 start < end 那么 `start` 从开始拷贝的结尾

``` javascript
var test = [1,2,3,4];

var clone = test.slice(2, 1);

console.log(clone); // [];
```

但是要时刻记住， slice 有一个语法糖，那就是 begin 可以是 负数

``` javascript
var test = [1,2,3,4];

var clone = test.slice(-1);
console.log(clone); // [4]
```

当 start 是 负数的时候 `slice` 将 后入式 从后往前数，输出倒数第二，如果 `end` 被省略，那么它只会输出一个


``` javascript
var test = [1,2,3,4];

var clone = test.slice(1, -1);

console.log(clone); // [2,3]
```

当 `start` 为正数  `end` 为负数的时候 语法糖使用与 `end` 上面这段代码就会从正数第二，一直拷贝到 倒数第二

``` javascript
var test = [1,2,3,4];

var clone = test.slice(-1, 1);

```

happy coding, may the code will always be with you~
